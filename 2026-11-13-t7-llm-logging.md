# T7 LLM 呼叫可觀察：結構化 log、一次呼叫一筆、不記 prompt

> 專案：AI 文件管理平台（LLM 整合層工程化，任務卡 T7）
> 分支：`t7-llm-logging`，PR #6
> 測試數量：44 → 48（新增 4 個）

---

## 一、今天完成的事

每次 `embed` / `generate` 都寫一筆 log，像計程車的行車紀錄器：記下去哪、花多久、有沒有繞路、跳表多少，但**不錄乘客的對話**。

實際長這樣：

```
INFO:documents.llm_service:llm_call operation=generate model=gemini-3.6-flash duration_ms=850 success=True retries=0 tokens=1234
```

### 程式碼（`documents/llm_service.py` 重點）

```python
logger = logging.getLogger(__name__)     # 'documents.llm_service'


def _token_count(result):
    """從回應拿 token 用量；任何一層沒有就回傳 None（embed 的回應通常沒有）。"""
    usage = getattr(result, 'usage_metadata', None)
    return getattr(usage, 'total_token_count', None)


class LLMService:
    def _call_with_retry(self, operation, model, func):
        attempt = 0
        start = time.monotonic()                     # 碼錶，不受電腦校時影響
        while True:
            try:
                result = func()
            except (errors.APIError, httpx.TransportError) as exc:
                if not self._is_transient(exc) or attempt >= self.MAX_RETRIES:
                    self._log_call(operation, model, start, success=False, retries=attempt)
                    raise LLMUnavailableError('LLM 服務暫時無法使用') from exc
                self._sleep(self.BASE_DELAY * 2 ** attempt)
                attempt += 1
                continue                             # 回到迴圈開頭，再試一次
            self._log_call(operation, model, start, success=True,
                           retries=attempt, tokens=_token_count(result))
            return result

    @staticmethod
    def _log_call(operation, model, start, success, retries, tokens=None):
        fields = {
            'operation': operation,
            'model': model,
            'duration_ms': round((time.monotonic() - start) * 1000),
            'success': success,
            'retries': retries,
            'tokens': tokens,
        }
        level = logging.INFO if success else logging.WARNING
        logger.log(
            level,
            'llm_call operation=%(operation)s model=%(model)s duration_ms=%(duration_ms)s '
            'success=%(success)s retries=%(retries)s tokens=%(tokens)s',
            fields,                  # 給人看：%(欄位名)s 自動從 dict 取值
            extra={'llm': fields},   # 給程式讀：掛在 log 紀錄上
        )
```

### 測試與故意破壞驗證

| 測試 | 一跑的結果 | 故意破壞 | 破壞後 |
|---|---|---|---|
| 成功的呼叫產生一筆完整的 log | 🔴 FAIL：no logs（真正的紅燈） | — | — |
| 失敗的呼叫也有 log，`retries=3`、WARNING | 🔴 FAIL：no logs（真正的紅燈） | — | — |
| 沒有 token 用量時仍正常輸出 | 🟢 | `_token_count` 不用 `getattr` | ERROR `AttributeError` |
| log 不含 prompt（成功、失敗都測） | 🟢 | 把 prompt 塞進 model 欄位 | FAIL，印出洩漏的那行 log |

---

## 二、核心概念

### 1. log 寫在哪裡：看「誰知道這些資訊」
耗時和重試次數只有 `_call_with_retry` 知道，所以 log 寫在那裡；它原本不知道操作類型和模型，就多傳兩個參數進去。

### 2. 結構化 log：收據上的文字和條碼
- 訊息用 `key=value`，人在終端機一眼看得懂
- `extra={'llm': fields}` 把同一包欄位掛在 log 紀錄上，程式（和測試）直接讀 `record.llm['retries']`，不用拆字串

### 3. 一次呼叫一筆，不是每次嘗試一筆
行車紀錄器是「一趟一筆」，不是每個紅燈一筆。統計失敗率時，log 筆數就等於呼叫次數，不會被重試灌水。

### 4. log 等級的選擇
- 成功 `INFO`、失敗 `WARNING`
- 不用 `ERROR`：呼叫端（T3 上傳、T6 問答）接住例外後會自己寫 ERROR，這裡不重複喊
- 等級要動態決定時，用 `logger.log(level, ...)`（`logger.info()` 就是它的簡寫）

### 5. `time.monotonic()` vs `time.time()`
`time.time()` 像牆上的時鐘，有人調時間它就跳；`time.monotonic()` 像碼錶，只會往前走，專門用來算經過時間。

### 6. 先查證回應格式
SDK 的 generate 回應有 `usage_metadata.total_token_count`，embed 回應**沒有**。所以「沒有 token 用量」不是少見的邊緣情況，而是每次 embed 都會遇到。

### 7. `getattr(物件, '屬性', None)` 一層一層防禦
`getattr(None, '任何屬性', None)` 也會回傳 `None`，所以兩行就能處理「哪一層不存在都不出錯」。

### 8. 不洩漏的做法是「根本不給」
`_log_call` 的參數裡根本沒有 prompt，失敗時也不記例外物件、不用 `logger.exception`。防洩漏最可靠的方法是讓 log 函式拿不到敏感資料。

### 9. 兩層 `with` 的順序
```python
with self.assertLogs(...) as logs:
    with self.assertRaises(LLMUnavailableError):
        llm.embed('hello')
```
`assertRaises` 要在裡面，不然例外會讓外層的 `assertLogs` 來不及檢查。

---

## 三、待釐清問題

- [ ] 目前 log 只輸出到 Django 預設的 handler。正式環境若要做統計，可以加一個 JSON formatter，把 `record.llm` 直接輸出成一行 JSON
- [ ] `duration_ms` 包含重試的等待時間。如果想知道「Gemini 本身回應多快」，要另外記每次嘗試的時間
- [ ] embed 沒有 token 用量，要估算免費額度只能看呼叫次數。Gemini 的免費額度是按「請求數」還是「token 數」計算？

---

## 四、下一步

**T8 真實整合測試（預設不執行）**：1～2 個真的打 Gemini 的測試，確認 mock 和現實沒有脫節。預設跳過，設定環境變數才執行，而且單次不超過 5 次請求。

---

## 五、後續發現（2026-11-13，PR #10）：成功的 log 在實際執行時看不到

用 curl 實際操作時發現：失敗的 `llm_call`（WARNING）會出現在 `runserver` 的輸出裡，**成功的（INFO）卻一筆都沒有**。

### 原因
- `documents` 這個 logger 沒有設定任何 handler（負責把 log 印出來的東西）
- 找不到 handler 時，Python 有一個備用機制 **lastResort**（最後手段），但它**只印 WARNING 以上**
- 所以 WARNING 看得到，INFO 被過濾掉

### 為什麼 T7 的測試沒抓到
`assertLogs` 會**暫時把 logger 的等級打開**來收集 log，所以測試環境裡永遠看得到 INFO。測試環境和真實環境不一樣。

> 在駕訓班考場，教練幫你把後照鏡調好了，考試都過；真的上路才發現自己的車後照鏡是歪的。

### 修正
`settings.py` 加 `LOGGING`，`documents.*` 平常印 INFO、跑測試時只印 WARNING（避免洗版）。為此把 T1 移除的 `TESTING = 'test' in sys.argv` 加回來，並註解它的用途。

### 測試：用子程序模擬真實環境
```python
result = subprocess.run(
    [sys.executable, '-c',
     'import logging, django; django.setup(); '
     'logging.getLogger("documents.llm_service").info("probe-visible")'],
    cwd=settings.BASE_DIR,
    env={**os.environ, 'DJANGO_SETTINGS_MODULE': 'config.settings'},
    capture_output=True, text=True,
)
self.assertIn('probe-visible', result.stderr)
```
子程序的指令裡沒有 `test`，所以載入的是真實執行時的設定。修正前 FAIL（stderr 是空的），修正後 OK。

### 學到的事
- **測試全綠只代表「在測試環境裡」沒問題**。有些東西（log 設定、環境變數、部署設定）測試環境和真實環境本來就不同，要另外驗證
- **實際操作一次很有價值**：這個問題是 T0～T9 全部做完、curl 實際跑一次才發現的
- 確認雜訊是不是自己造成的：用 `git stash` 暫時收起修改，比較修改前後的輸出（測試中的 4 行 WARNING 在修改前就存在）
