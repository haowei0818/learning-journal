# 履歷對應資料：AI 文件管理平台

> 用途：履歷上每一句話都有對應的程式碼、測試和 PR 可以佐證；面試時可以直接打開給面試官看。
> 資料核對日期：2026-11-13（數字皆從程式碼實際計算）
> 專案：https://github.com/haowei0818/ai-document-platform

---

## 一、履歷專案描述（可直接貼到 104）

**AI 文件管理平台**｜個人專案｜Python、Django REST Framework、Gemini API、Chroma
GitHub：https://github.com/haowei0818/ai-document-platform

- 以 Django REST Framework 開發文件上傳與 RAG（檢索增強生成）問答 API，支援 `.txt` / `.pdf` / `.docx`，使用 JWT 認證，並依使用者隔離文件與問答結果
- 設計 **LLM Service 層**作為所有 Gemini 呼叫的唯一入口：將錯誤分為暫時性與持續性，暫時性錯誤以指數退避自動重試，並統一轉成自訂例外，呼叫端不需接觸 SDK
- 實作文件處理狀態（`pending` / `ready` / `failed`）與**冪等**的重新處理 API，確保向量資料庫只存放可用文件；刪除文件時同步清除向量
- AI 服務無法使用時回傳 **503** 與統一錯誤格式，內部細節只寫入 log；每次 AI 呼叫留下**結構化 log**（耗時、重試次數、token 用量），且不含 prompt 與文件內容
- 撰寫 **51 個自動化測試**：49 個以 mock 執行，不需 API key、不打網路；2 個真實 Gemini 整合測試預設跳過，用來確認 mock 與現實一致。採 TDD 開發，並以「故意改壞程式、確認測試變紅」驗證測試有效

---

## 二、每一句話的佐證

### 1. RAG 問答 API、JWT、使用者隔離

| 佐證 | 位置 |
|---|---|
| 問答流程（向量化 → 檢索 → 生成） | `documents/views.py` 的 `DocumentAskView._answer()` |
| 依 `user_id` 篩選向量 | 同上，`collection.query(..., where={'user_id': user_id})` |
| 三種檔案格式解析 | `documents/rag_utils.py` 的 `extract_text_from_file()` |
| 跨使用者隔離測試 | `documents/tests.py`：`test_ask_does_not_leak_other_users_documents`、`test_list_only_shows_own_documents`、`test_delete_other_users_document` |
| 實際操作畫面 | README「實際操作範例」 |

### 2. LLM Service 層、錯誤分類、指數退避、統一例外

| 佐證 | 位置 |
|---|---|
| 唯一入口 | `documents/llm_service.py` 的 `LLMService`；全專案只有這個檔案 import Gemini SDK |
| 錯誤分類 | `LLMService._is_transient()`、`_is_daily_quota()` |
| 指數退避重試（1、2、4 秒） | `LLMService._call_with_retry()` |
| 統一例外 | `LLMUnavailableError`，以 `raise ... from exc` 保留原始錯誤 |
| 測試 | `documents/test_llm_service.py` 的 `LLMServiceRetryTests`（8 個） |
| PR | [#2 T4 自動重試](https://github.com/haowei0818/ai-document-platform/pull/2) |

| 錯誤 | 處理 |
|---|---|
| 5xx、429 每分鐘限制、429 無 details、逾時、連線錯誤 | 重試 |
| 429 每日額度（`quotaId` 含 PerDay / Daily）、其他 4xx、其他網路錯誤 | 不重試 |

### 3. 處理狀態、冪等重新處理、刪除同步清除向量

| 佐證 | 位置 |
|---|---|
| 狀態流程 | `documents/rag_utils.py` 的 `process_document()`：pending → 先清舊 chunk → ready / failed |
| 重新處理 API | `documents/views.py` 的 `DocumentReprocessView`（`POST /api/documents/<id>/reprocess/`） |
| 刪除同步清向量 | `DocumentDeleteView.perform_destroy()` → `delete_document_chunks()` |
| 冪等測試 | `documents/tests.py`：`test_reprocess_twice_is_idempotent` |
| 失敗不殘留測試 | `test_upload_partial_failure_leaves_no_chunks`、`test_reprocess_failure_marks_failed_without_chunks` |
| PR | [#1 T0～T3](https://github.com/haowei0818/ai-document-platform/pull/1)、[#3 T5 重新處理](https://github.com/haowei0818/ai-document-platform/pull/3) |

### 4. 503、統一錯誤格式、結構化 log

| 佐證 | 位置 |
|---|---|
| 503 回應 | `DocumentAskView.post()`：`{"code": "llm_unavailable", "detail": "..."}` |
| 不洩漏測試 | `test_ask_generate_failure_returns_503_without_leaking`（例外訊息塞模型名稱、假 API key、Traceback，檢查回應裡都沒有） |
| 結構化 log | `LLMService._log_call()`；`config/settings.py` 的 `LOGGING` |
| log 不含 prompt 測試 | `test_logs_never_contain_prompt` |
| PR | [#4 T6 503](https://github.com/haowei0818/ai-document-platform/pull/4)、[#6 T7 log](https://github.com/haowei0818/ai-document-platform/pull/6)、[#10 log 設定修正](https://github.com/haowei0818/ai-document-platform/pull/10) |

### 5. 51 個測試、mock、整合測試、TDD

| 模組 | 數量 |
|---|---|
| `accounts/tests.py` | 7 |
| `documents/tests.py` | 29 |
| `documents/test_llm_service.py` | 13 |
| `documents/test_integration.py`（預設跳過） | 2 |
| **合計** | **51**（預設 49 個執行，`OK (skipped=2)`） |

| 佐證 | 位置 |
|---|---|
| 不需 API key | README「只跑測試」；已在全新 clone + 全新 venv、沒有 `.env` 的環境實際驗證 |
| 整合測試 | `documents/test_integration.py`；`RUN_LLM_INTEGRATION=1` 才執行，單次 2 個請求 |
| TDD 過程 | 每張卡的 commit 都有「紅燈測試」→「實作」的順序，例如 T2：`a355eb0`（紅燈）→ `91c168c`（實作） |
| PR | [#7 T8 整合測試](https://github.com/haowei0818/ai-document-platform/pull/7) |

---

## 三、面試故事（STAR）

### 故事 1：發現資料外洩漏洞（T0 → T2）
- **S**：接手自己的舊程式碼，準備做工程化改善
- **T**：先查證現況，不要憑印象改
- **A**：檢查刪除流程，發現 `DocumentDeleteView` 只刪資料庫，沒刪向量資料庫；問答又只用 `user_id` 篩選，所以**已刪除文件的內容還會出現在問答結果裡**。先寫測試重現，再修正，並把「向量寫入與刪除集中在同一個模組」定為不變條件
- **R**：補上 4 個測試（刪除後清除向量、已刪除文件不出現在問答來源、不影響他人的向量、沒有 chunk 也不報錯）
- 佐證：`docs/t0-findings.md`、commit `a355eb0` → `75e992b`

### 故事 2：設計 AI 呼叫的錯誤處理（T4、T6）
- **S**：Gemini 免費額度常遇到 429、503，原本錯誤直接變成 500
- **T**：暫時性錯誤自動恢復，持續性錯誤快速失敗，使用者看到清楚的訊息
- **A**：查 SDK 原始碼和開發者論壇，確認 429 有「每分鐘」和「每日」兩種，要看 `quotaId` 區分；設計判斷表、指數退避、統一例外；問答 API 接統一例外回 503
- **R**：用真實 Gemini 驗證：錯誤 API key 回 400 → 判斷為持續性 → 只請求 1 次；不存在的模型回 404 → 同樣不重試

### 故事 3：測試全綠，實際操作才發現的 bug（T7 後續）
- **S**：T0～T9 全部完成，50 個測試全綠
- **T**：用 curl 實際跑一次完整流程，放進 README
- **A**：發現成功的 AI 呼叫 log 沒有印出來。查到原因是 logger 沒設定 handler，Python 的 lastResort 只印 WARNING；測試沒抓到，是因為 `assertLogs` 會暫時打開 log 等級。用子程序模擬真實環境寫測試重現，再補上 LOGGING 設定
- **R**：學到「測試環境和真實環境不同的地方要另外驗證」
- 佐證：[PR #10](https://github.com/haowei0818/ai-document-platform/pull/10)

### 故事 4：整合測試的成本控制（T8）
- **S**：mock 測試可能和真實 Gemini 脫節，但真的打 API 要花免費額度
- **T**：單次執行不超過 5 次請求
- **A**：預設用 `skipUnless` 跳過；發現 T4 的重試最壞會讓 2 個測試打到 8 次，所以整合測試內關閉重試；用 monkey patching 在 SDK 外面包計數器實測請求數
- **R**：實測 2 次請求；用不存在的模型做故意破壞，確認真實 404 只請求 1 次

---

## 四、可能被問的問題

**Q1：為什麼上傳時 AI 失敗回 200，問答時卻回 503？**
上傳的主要工作是「收下文件」，這件事成功了，只是後續處理失敗，所以回 201 + `status: failed`，使用者之後可以重新處理。問答的主要工作就是「產生回答」，AI 不能用時整個請求都無法完成，所以回 503 告訴客戶端「稍後再試」。

**Q2：為什麼要有 LLM Service 這一層？**
集中處理錯誤、重試、log；換模型或換供應商時只改一個檔案；測試可以整個換成假的，不花錢、不打網路。

**Q3：怎麼確認測試真的有效，不是怎樣都會過？**
TDD 先看到紅燈；一跑就綠的測試，暫時把程式改壞，確認測試會變紅再改回來。要改壞的是程式，不是測試。

**Q4：重新處理怎麼做到冪等？**
處理前先刪掉這份文件所有舊的 chunk 再重建，所以處理幾次 chunk 數量都一樣。chunk id 有隨機 uuid，所以測試比對的是數量。

**Q5：log 怎麼確保不洩漏 prompt？**
`_log_call()` 的參數裡根本沒有 prompt，也不記例外物件；測試用特殊字串當 prompt，檢查成功和失敗兩條路徑的 log 都沒有。

**Q6：還有什麼可以改進？**（誠實回答已知限制）
- 文件處理是同步的，大檔案會讓請求等很久，可以改成背景工作（例如 Celery）
- 指數退避沒有加 jitter，多個客戶端同時重試時可能撞在一起
- `SECRET_KEY` 寫在 settings 裡，部署前要改成環境變數
- 429 的 `quotaId` 格式參考社群回報，還沒用真實資料驗證

---

## 五、相關連結

- 專案 README：https://github.com/haowei0818/ai-document-platform#readme
- 任務卡：https://github.com/haowei0818/ai-document-platform/blob/main/docs/tickets-llm-hardening.md
- T0 查證紀錄：https://github.com/haowei0818/ai-document-platform/blob/main/docs/t0-findings.md
- 學習紀錄目錄：https://github.com/haowei0818/learning-journal#readme
