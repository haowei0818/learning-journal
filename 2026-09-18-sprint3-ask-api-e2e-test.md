# Sprint 3 續 — 問答 API 串接 urls、bash heredoc 除錯、端對端測試

## 今天完成的事

延續前面已經寫好的 `DocumentAskView`（RAG 問答邏輯：接收問題 → 向量化 → Chroma 檢索（含 `user_id` 篩選）→ 組 prompt → 呼叫 Gemini → 回傳答案與來源），今天把它正式接上路由、除錯了一個 bash 指令的小狀況、並完成一次完整的端對端測試（上傳新文件 → 問一個字面完全不同的問題 → 成功拿到正確答案），確認整條 RAG 管線在 Django 裡真正能用。程式碼已 commit、push。

---

## 核心概念與操作整理

### 1. `urls.py` 除錯：先靠猜測,最後靠證據

一開始貼 `cat documents/urls.py` 的結果，最後一行後面沒有印出收尾的 `]`，懷疑檔案語法不完整。過程中經歷了兩次不同的推測：

- **第一次猜測**：`wc -l` 顯示行數比預期少 1 行，加上提示字元前黏著一個奇怪的 `]`，一度以為「`]` 其實有寫進去，只是檔案結尾少了換行符號，被 `wc -l` 漏算」。
- **驗證後推翻**：改用 `cat -A documents/urls.py | tail -3`（`-A` 會把換行符號明確印成 `$`），結果只印出 2 行、都有 `$`，代表檔案後面根本没有第三行——`]` 是真的不存在，不是格式瑕疵。

**這次學到的教訓**：遇到看起來矛盾的線索時，不要停在「合理的推測」就下結論，要找一個更明確、排除干擾的指令去驗證（`cat -A` + `tail` 組合，直接看換行符號本身），眼見為憑。

### 2. `heredoc` 指令本身打錯字，但檔案結果仍然正確

用 `cat > 檔案 << 'EOF'` 重寫 `urls.py` 時，指令打成了 `<< 'EOF''`（多了一個單引號），導致 heredoc 結束的判斷跑掉，多印出一行 `path('ask/'...)` 被 bash 誤認成獨立指令去執行、跳出 `syntax error`。

**處理方式沒有因為看到紅字就重來**，而是照舊習慣馬上用 `cat documents/urls.py` 重新確認實際內容——結果檔案是完整、正確寫入的，紅字只是 bash 指令解析的小插曲，跟「寫進檔案」這件事是兩回事。之後又用 `python manage.py check` 二次確認 Django 層面沒有語法或設定問題。

**教訓（呼應 Sprint 2 已經學過的原則）**：看到終端機報錯，第一反應不是慌，而是「這個錯誤，跟我真正在乎的結果（檔案內容對不對）有沒有關係」，用指令去查證，而不是用猜的。

### 3. 端對端測試：上傳 → 問答，語意檢索驗證成功

```bash
# 建立測試文件
cat > test_leave.txt << 'EOF'
員工請假規定：
特休假需提前3天向主管申請。
病假需檢附診斷證明，3天以上須補交假單。
EOF

# 上傳
curl -s -X POST http://127.0.0.1:8000/api/documents/upload/ \
  -H "Authorization: Bearer $TOKEN" \
  -F "title=請假規定測試" \
  -F "file=@test_leave.txt"

# 問答（故意用跟原文不同的字眼：「講」而不是「申請」）
curl -s -X POST http://127.0.0.1:8000/api/documents/ask/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "特休要提前幾天跟主管講?"}'
```

回傳結果：
```json
{"answer":"根據文件內容，特休假需提前 **3 天** 向主管申請。","sources":[{"document_id":5,"title":"請假規定測試"},{"document_id":4,"title":"員工請假規定"}]}
```

答案正確，且問題字面跟文件原文不同，證明檢索是靠語意、不是關鍵字比對。

### 4. `sources` 出現「意外」的文件，其實是合理結果——用交叉比對驗證，不用猜的

回傳的 `sources` 裡多了一筆 `document_id: 4`（不是這次上傳的文件），一開始不確定這是不是篩選失效、混進了別人的資料。

**驗證方式**：直接呼叫 `document-list` API，把目前帳號底下的所有文件列出來比對：
```bash
curl -s -X GET http://127.0.0.1:8000/api/documents/ -H "Authorization: Bearer $TOKEN"
```
結果 `id: 4` 確實在清單裡——是同一個帳號（`admin`）之前（Sprint 3 驗證持久化功能時）上傳過的舊文件，只是內容剛好也跟請假相關，被語意檢索判斷為相關文件一起列出。**不是 bug，是 RAG 該有的行為**，同時也順便驗證了 `where={'user_id': ...}` 篩選確實有效（如果失效，混進來的會是別人帳號的文件，而不是自己帳號的舊資料）。

**方法論上的教訓**：懷疑一個結果「是不是有問題」時，不要靠印象猜，要找一個可以直接比對的資料源（這裡是 `document-list`）去交叉驗證。

### 5. Git 流程：只加相關檔案，測試用的臨時檔案不進版控

```bash
git status
# modified: documents/serializers.py, urls.py, views.py
# untracked: test_leave.txt  ← 只是臨時測試檔，不是專案程式碼

git add documents/serializers.py documents/urls.py documents/views.py
git commit -m "新增文件問答 API：DocumentAskView、DocumentAskSerializer、urls 路由"
git push
```

`push` 前依照 SOP 先跑 `git status` 確認清單正確；`test_leave.txt` 刻意不 `git add`，因為它只是測試時臨時產生的檔案，不屬於專案程式碼本身。

---

## 還不確定、需要之後多練習的地方

1. **`document-ask` API 還沒有自動化測試**：目前只用 `curl` 手動驗證過一次「正常情況」，還沒仿照 `DocumentTests` 的邏輯，補上「未登入」「跨使用者（B 使用者的問題不該檢索到 A 的文件）」等測試案例——這是下一步最重要的待辦。
2. **還沒真正測過「跨使用者」情境**：目前驗證的 `document_id: 4` 是同一個帳號自己的舊資料，證明了篩選「至少沒有放行不該出現的自己資料以外的東西」，但還沒真的用兩個不同帳號實測，親眼確認 B 帳號問問題時，A 帳號的文件真的完全不會出現。
3. **`.pdf`/`.docx` 讀取仍然沒有實際測試過**（沿續 Sprint 3 之前記錄的待辦），這次測試只用了 `.txt`。
4. **`test_leave.txt` 等測試殘留檔案的整理**：本機資料夾裡現在有好幾份測試用文件（`test.txt`、`test_document.txt`、`test_leave.txt`），之後可以考慮養成測試完就清掉的習慣，避免越堆越多。

---

## 下一步

1. 幫 `DocumentAskView` 補上自動化測試（仿照 `DocumentTests` 的設計邏輯：正常情況、未登入、跨使用者情境）
2. 找機會用兩個不同帳號實測一次「跨使用者」情境，親眼確認安全邊界真的守住
3. 之後找時間實際測試 `.pdf`/`.docx` 上傳與讀取
4. 視進度更新 `progress.md`
