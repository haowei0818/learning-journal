# LLM 整合層工程化 T0～T3 — LLM Service、測試隔離、刪除不殘留向量、處理狀態

> 本檔案合併自 2026-11-08 的 3 份學習筆記（T0+T1、T2、T3）。
> 對應專案：`haowei0818/ai-document-platform`，PR #1（`t3-document-status` → `main`）。
> 任務卡來源：`docs/tickets-llm-hardening.md`（/grill-me 共識 Q1～Q9）。

## 這一輪的目標

讓 RAG 文件問答從「能跑」變成「工程化」：測試不花錢、刪除不外洩、上傳狀態看得見。全程零成本（只用 Gemini 免費額度）。

| 卡 | 主題 | 測試數 |
|---|---|---|
| T0 | 查證現況（只讀） | 18 |
| T1 | LLM Service、測試不打網路不留垃圾 | 18 |
| T2 | 刪除文件不殘留向量 | 22 |
| T3 | 上傳處理狀態 pending / ready / failed | 29 |
## 第一部分：T0 查證 + T1 LLM Service 與測試隔離

### 今天完成的事

- **T0**：查證現況，產出 `docs/t0-findings.md`（分支 `t0-findings`）
  - 確認漏洞：刪除文件不會清 Chroma 向量 → 交給 T2
  - 額外發現：測試會真的打 Gemini、`test_upload_unauthenticated` 依賴別的測試先建立 `test.txt`
- **T1**：建立 LLM Service（總機），測試不打網路、不留垃圾（分支 `t1-llm-service`）
  - 紅燈：沒有 API key 時 18 個測試裡 8 個 `ValueError: No API key was provided`
  - 綠燈：沒有 key、封鎖網路下 18 個全過；連跑兩次、打亂順序、反向、每個單獨跑都通過；跑完沒有 `chroma_db*/`、`media/`、`test.txt`

### 程式碼重點

#### 1. 總機 `documents/llm_service.py`
```python
class LLMService:
    EMBED_MODEL = 'gemini-embedding-001'
    GENERATE_MODEL = 'gemini-3.6-flash'

    def __init__(self, api_key=None):
        self._api_key = api_key
        self._client = None              # 先不連線

    def _get_client(self):
        if self._client is None:         # 第一次用到才讀 .env、才連線
            from dotenv import load_dotenv
            from google import genai
            load_dotenv()
            api_key = self._api_key or os.environ.get('GEMINI_API_KEY')
            self._client = genai.Client(api_key=api_key)
        return self._client

    def embed(self, text: str) -> list[float]: ...
    def generate(self, prompt: str) -> str: ...
```

#### 2. 呼叫端改用總機（依賴注入）
```python
def process_document_for_rag(document, llm=None):
    llm = llm or LLMService()      # 有傳就用傳進來的，沒傳就用真的
    ...
    embedding = llm.embed(chunk)
```

#### 3. Chroma 倉庫可以替換
```python
_chroma_client = None

def get_chroma_client(): ...        # 第一次用到才建立 PersistentClient("./chroma_db")
def set_chroma_client(client): ...  # 測試換成 EphemeralClient()
```

#### 4. 測試
```python
class FakeLLMService:
    def embed(self, text):   return [float(len(text)), 1.0, 0.0]
    def generate(self, prompt): return '這是測試用的假回答'

# setUp：擦黑板 + 兩個地方都換成假總機
for c in self.chroma.list_collections():
    self.chroma.delete_collection(c.name)
for target in ('documents.rag_utils.LLMService', 'documents.views.LLMService'):
    patcher = patch(target, FakeLLMService); patcher.start(); self.addCleanup(patcher.stop)

# 上傳用記憶體裡的假檔案
SimpleUploadedFile('test.txt', content.encode('utf-8'))
```

### 核心概念

| 名詞 | 中文 / 比喻 | 今天用在哪 |
|---|---|---|
| Deep module（深模組） | 公司總機：按鈕少、裡面藏很多細節 | `LLMService` 只有 `embed`、`generate` |
| 不讓細節外洩 | 呼叫端不用懂 SDK 結構 | 回傳 `list[float]` / `str`，不回傳 SDK 物件 |
| Lazy initialization（延遲初始化） | 要出門才發動車子 | 沒 API key 也能建立 `LLMService()` |
| Dependency injection（依賴注入） | 有帶傘用你的，沒帶我借你 | `llm=None` → `llm or LLMService()` |
| Duck typing（鴨子型別） | 長得像鴨子就當鴨子用 | `FakeLLMService` 有同樣兩個方法就能替換 |
| patch 要換「使用的地方」 | 換掉每一支分機，不是總公司 | 要 patch `rag_utils` 和 `views` 兩處 |
| 測試隔離 | 每節下課要擦黑板 | DB 會還原但 Chroma 不會，`user_id` 會重複使用 |
| Refactor（重構） | 外在行為不變、只整理內部 | 重構前後紅燈的「8 個錯誤」完全一樣 |
| Dead code（死碼） | 沒人用但會誤導人 | 刪掉 `settings.TESTING` |
| DRY | 不要重複自己 | `_upload()` 小幫手 |

### 今天踩到的坑

- 搬程式碼時差點弄丟 `load_dotenv()`：原本只有 `get_gemini_client()` 會讀 `.env`，總機第一版漏掉，正式環境會拿不到 key。→ **重構時要逐行比對「原本做了哪些小事」**

### 待釐清問題

- WIP commit 沒辦法整理成一個（雲端環境不允許 force push）。要不要在本機用 `git rebase` 或 GitHub 的 *Squash and merge* 合成一個？
- `accounts` 的 7 個測試沒有碰 LLM / Chroma，所以沒有改；之後會不會需要共用的測試基底類別？

### 下一步

- **T2｜刪除文件不殘留向量**：建立「依 `document_id` 刪除全部 chunk」的單一入口，`DocumentDeleteView` 刪除時同步呼叫
- 開始前先想：刪除向量要寫在 view 的 `perform_destroy`，還是 model 的 signal？各有什麼優缺點？

---

## 第二部分：T2 刪除文件不殘留向量

### 今天完成的事

- 修補 T0 發現的漏洞：刪除文件後，向量（chunk）還留在 Chroma，內容會被問答找出來
- 分支：`t2-delete-vectors`（從 `t1-llm-service` 開出來）
- 紅燈：`AssertionError: Lists differ: ['doc1_chunk0_572b2b0b'] != []`
- 綠燈：22 個測試全過（T1 的 18 個 + T2 新增 4 個），打亂順序也通過
- 反向驗證：故意拿掉刪除向量那一行，有 2 個測試會失敗 → 證明測試真的抓得到漏洞

### 程式碼

#### 刪除的單一入口（`documents/rag_utils.py`）
```python
def delete_document_chunks(document_id):
    """刪除某份文件在 Chroma 裡的全部 chunk；沒有 chunk 時什麼都不做。"""
    get_chroma_collection().delete(where={'document_id': document_id})
```

#### 刪除 API（`documents/views.py`）
```python
class DocumentDeleteView(generics.DestroyAPIView):
    ...
    def perform_destroy(self, instance):
        delete_document_chunks(instance.id)   # 先刪向量
        instance.delete()                     # 再刪資料庫紀錄
```

#### 新增的 4 個測試
| 測試 | 對應驗收條件 |
|---|---|
| `test_delete_removes_document_chunks` | 刪除後找不到該文件的 chunk；同一使用者的其他文件還在 |
| `test_deleted_document_not_in_ask_sources` | 問答的 `sources` 不會出現已刪文件 |
| `test_delete_does_not_affect_other_users_chunks` | 其他使用者的資料不受影響 |
| `test_delete_document_without_chunks` | 刪除沒有向量的文件不會報錯（手動先清掉向量來模擬） |

### 核心概念

| 名詞 | 中文 / 比喻 | 今天用在哪 |
|---|---|---|
| `perform_destroy` vs signal | 櫃台辦退件順手通知 vs 自動警報器 | 選 `perform_destroy`：看得到、好測試、符合任務卡範圍 |
| 兩個系統沒有共同交易 | 資料庫和 Chroma 無法保證「一起成功或一起失敗」 | 所以要想「中途壞掉時哪種結果比較安全」 |
| 先刪向量、再刪資料庫 | 失敗時頂多「不方便」，而不是「外洩」 | `perform_destroy` 的順序 |
| 讀錯誤訊息 | `FAIL`＝條件不成立；`ERROR`＝程式當掉 | `Lists differ: [...] != []` |
| Regression test（回歸測試） | 防止舊問題又跑回來 | 把 T0 的肉眼發現變成自動證據 |
| 反向驗證（mutation） | 故意弄壞程式，看測試會不會叫 | 拿掉刪除那一行 → 2 個測試失敗 |
| 補驗收的測試 ≠ TDD 紅燈 | 功能做好後補的測試會一開始就綠 | 所以要用反向驗證證明它有用 |

### 待釐清問題 / 未決事項

- **刪除使用者時**（`on_delete=CASCADE`），文件跟著刪，但不會經過 `perform_destroy` → 向量仍會殘留。不在 T2 範圍；之後如果要處理，signal 或在刪除使用者的流程中呼叫 `delete_document_chunks` 是候選做法
- 刪除文件時，`media/` 裡的實體檔案也不會被刪（Django `FileField` 的預設行為），同樣不在這次範圍

### 下一步

- **T3｜上傳處理狀態（`pending` / `ready` / `failed`）**
  - `Document` 新增 `status` 欄位 + migration
  - 上傳流程：`pending` → 先清舊 chunk（重用 `delete_document_chunks`）→ 處理 → `ready` 或 `failed`
  - 開始前先想：既有的舊文件 migration 之後，狀態應該是什麼？為什麼？

---

## 第三部分：T3 上傳處理狀態

### 今天完成的事

- 分支：`t3-document-status`（從 `t2-delete-vectors` 開出來）
- 上傳後看得到文件能不能問答；不會再出現「顯示上傳成功、其實問不到」
- 測試：22 → 29 個，全部通過（含打亂順序）
- 反向驗證：
  - migration 第一步改成 `pending` → migration 測試變紅
  - 拿掉「失敗時清掉殘留 chunk」→ 2 個測試變紅

### 程式碼

#### 1. Model（`documents/models.py`）
```python
class Document(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending', '處理中'
        READY = 'ready', '可問答'
        FAILED = 'failed', '處理失敗'
    ...
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.PENDING)
```

#### 2. Migration 分兩步（`documents/migrations/0003_document_status.py`）
```python
operations = [
    migrations.AddField(..., default="ready"),    # 既有文件全部填 ready
    migrations.AlterField(..., default="pending"), # 之後新增的才預設 pending
]
```

#### 3. 處理流程（`documents/rag_utils.py`）
```python
def process_document(document, llm=None):
    document.status = Document.Status.PENDING
    document.save(update_fields=['status'])
    delete_document_chunks(document.id)          # 先清舊 chunk
    try:
        _index_chunks(document, llm)
        document.status = Document.Status.READY
    except Exception:
        logger.exception('document processing failed: document_id=%s', document.id)
        delete_document_chunks(document.id)      # 清掉處理到一半的殘留
        document.status = Document.Status.FAILED
    document.save(update_fields=['status'])
    return document
```

#### 4. View 變得很薄（`documents/views.py`）
```python
def perform_create(self, serializer):
    instance = serializer.save(user=self.request.user)
    process_document(instance)
```

#### 5. Serializer 白名單＋唯讀
```python
fields = ['id', 'title', 'file', 'uploaded_at', 'status']
read_only_fields = ['status']
```

#### 6. 測試技巧
```python
# 會壞掉的假總機（繼承）
class FailingLLMService(FakeLLMService):
    def embed(self, text):
        raise RuntimeError('模擬 LLM 失敗')

# 接住 log，畫面不亂、又能檢查有被記錄
with self.assertLogs('documents.rag_utils', level='ERROR') as logs:
    response = self._upload()
self.assertIn(f"document_id={response.data['id']}", logs.output[0])

# migration 時光機
executor.migrate([('documents', '0002_alter_document_file')])  # 倒帶
OldDocument = old_apps.get_model('documents', 'Document')         # 歷史版本的 model
executor.migrate([('documents', '0003_document_status')])         # 快轉
```

### 核心概念

| 名詞 | 中文 / 比喻 | 今天用在哪 |
|---|---|---|
| Migration 兩步走 | 在學生先填「已領取」、新生預設「待領取」 | 舊文件 `ready`、新文件 `pending` |
| Historical model（歷史版本的 model） | 用當年的表格填當年的資料 | migration 測試 |
| `TransactionTestCase` | 真的寫進資料庫、結束再清空 | 改資料庫結構的測試 |
| `tearDown` | 不管成敗都會執行的收尾 | 時光機一定要回到現在 |
| `NodeNotFoundError` | migration 圖上找不到這一站 | 0003 還沒建立 |
| `TextChoices` | 只能選清單裡的值 | `Document.Status` |
| Mass assignment（大量賦值） | 使用者夾帶不該由他決定的欄位 | `read_only_fields = ['status']` |
| 白名單 vs `'__all__'` | 只開放列出來的欄位 | serializer 的 `fields` |
| `except Exception` + log | 接住所有錯誤，但一定要留紀錄 | 處理失敗 → `failed` |
| Inheritance（繼承） | 兒子自動有爸爸的方法，只改要改的 | `FailingLLMService(FakeLLMService)` |
| 先綠燈、再重構 | 先求對、再求好 | 先寫在 view，綠燈後搬到 `rag_utils` |
| 薄 view | view 只接請求，流程交給別的模組 | `perform_create` 只剩兩行 |

### 待釐清問題

- `except Exception` 會不會藏住程式的 bug？T4 之後 LLM 錯誤會變成統一例外，到時候要不要把範圍縮小？
- 上傳時處理是「同步」的（使用者要等處理完才拿到回應）。文件很大時會等很久——這是任務卡「不在範圍內」的非同步佇列（Celery）要解決的事

### 下一步

- 照建議順序是 **T4｜暫時性錯誤自動重試**（T5 依賴 T3，也可以先做 T5）
- T4 開始前先想：「429 流量限制」和「每日額度用完」都是 Gemini 在說「不行」，為什麼一個要重試、一個不要？
