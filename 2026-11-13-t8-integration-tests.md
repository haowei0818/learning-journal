# T8 真實 Gemini 整合測試：預設跳過、關掉重試控制成本、用真實錯誤驗證 T4

> 專案：AI 文件管理平台（LLM 整合層工程化，任務卡 T8）
> 分支：`t8-integration-tests`，PR #7
> 測試數量：48 → 50（新增 2 個，預設跳過）

---

## 一、今天完成的事

前面 48 個測試用的都是假的 Gemini。假的是照「查證到的格式」做的，Gemini 哪天改格式或模型下架，假測試還是全綠。所以加 2 個**真的**打 Gemini 的測試，平常跳過，偶爾手動跑。

> 消防演習（假測試）天天練，但一年也要真的打一次 119 確認號碼還通。

### 程式碼（`documents/test_integration.py`）

```python
RUN_INTEGRATION = os.environ.get('RUN_LLM_INTEGRATION') == '1'


@skipUnless(RUN_INTEGRATION, '設定 RUN_LLM_INTEGRATION=1 才執行（會呼叫真實 Gemini）')
class GeminiIntegrationTests(SimpleTestCase):

    def setUp(self):
        self.llm = LLMService()
        self.llm.MAX_RETRIES = 0      # 關掉重試：每個測試最多 1 次請求

    def test_embed_returns_float_vector(self):
        vector = self.llm.embed('整合測試')
        self.assertIsInstance(vector, list)
        self.assertEqual(len(vector), 3072)
        self.assertTrue(all(isinstance(v, float) for v in vector))

    def test_generate_returns_text_and_token_usage(self):
        with self.assertLogs('documents.llm_service', level='INFO') as logs:
            answer = self.llm.generate('請只回覆「OK」兩個字。')
        self.assertIsInstance(answer, str)
        self.assertTrue(answer.strip())
        self.assertIsNotNone(logs.records[0].llm['tokens'])
```

執行方式：
```bash
python manage.py test                                              # 平常：skipped=2，不打網路
RUN_LLM_INTEGRATION=1 python manage.py test documents.test_integration   # 手動：真的打 Gemini
```

### 驗證結果

| 執行 | 結果 | 實際請求數 |
|---|---|---|
| 預設 | 50 個測試，`OK (skipped=2)` | 0 |
| `RUN_LLM_INTEGRATION=1` | 2 個通過 | 2 |
| 故意破壞：模型改成 `gemini-does-not-exist` | ERROR | 1 |

---

## 二、核心概念

### 1. `skipUnless(條件, 原因)`
條件不成立時整個類別跳過，**連 `setUp` 都不執行**。所以 `LLMService()` 根本沒被建立，不會讀 `.env`、不會連線。`-v 2`（verbose）可以看到每個測試和跳過的原因。

### 2. 關掉重試來控制成本
T4 的重試最壞會讓 2 個測試打到 8 次，超過 5 次上限。整合測試的目的是「確認格式」不是「確認重試」，重試已有 T4 的 mock 測試保護，所以設 `MAX_RETRIES = 0`，最壞 2 次。
- `llm.MAX_RETRIES = 0` 在**物件**上建立同名屬性，蓋過類別屬性，其他 `LLMService` 不受影響

### 3. 整合測試沒有「功能還沒做」的紅燈
功能早就寫好了，整合測試是確認「我們的程式跟真實世界對得上」。不假裝有紅燈，改用故意破壞驗證它真的有在檢查。

### 4. 用真實錯誤驗證 T4
模型改成不存在時，真的 Gemini 回 `404 NOT_FOUND`：
```
google.genai.errors.ClientError: 404 NOT_FOUND ...
documents.llm_service.LLMUnavailableError: LLM 服務暫時無法使用
```
- 例外鏈（`raise ... from exc`）兩行都看得到
- 404 是 4xx 但不是 429 → 持續性錯誤 → 不重試，實際只請求 1 次
- T4 當時全用假錯誤，這次第一次用真實錯誤驗證

### 5. 量測請求數：在 SDK 外面包計數器
寫一個暫存腳本，把 `google.genai.models.Models.embed_content` / `generate_content` 換成「先計數、再呼叫原本的函式」，然後照常執行 `manage.py test`。專案程式完全不用改。這種「把函式換成包了一層的版本」的技巧叫 **monkey patching**。

### 6. 測試時也要省額度
prompt 寫「請只回覆 OK」，回答越短 token 越少。

---

## 三、待釐清問題

- [ ] SDK 每次 `generate_content` 都印 `Direct use of automatic function calling (AFC) ... is not recommended`。AFC 是讓 AI 自動呼叫 Python 函式的功能，我們沒用到但預設開著。要不要在呼叫時關掉？
- [ ] 整合測試多久跑一次比較好？可以考慮每次升級 `google-genai` 套件、或換模型之前跑一次
- [ ] 429 的真實格式（`quotaId`）還是沒有驗證到，因為不想故意打爆額度

---

## 四、下一步

**T9 文件與履歷更新**：README 補上 T7（結構化 log）和 T8（兩種測試的執行方式），確認照著 README 在全新環境不需要 API key 就能跑完測試；更新履歷的專案描述，數字要跟實際程式碼一致。
