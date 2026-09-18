# Sprint 2 收尾 — curl 實測、DocumentTests、發現並修正權限漏洞

## 今天完成的事

1. 暖身：重新跑 `accounts` 舊測試，確認隔了 12 天沒壞（7 個測試全過）
2. 用 `curl` 實際測試 Documents 三支 API（上傳、列表、刪除），驗證行為符合設計
3. 設計 `DocumentTests` 的完整測試案例（8 個），涵蓋正常情況、未登入、跨使用者情境
4. 動手寫測試過程中，意外發現三支 View **沒有設定「必須登入」的權限保護**，屬於真正的安全漏洞，已修正並補上 `IsAuthenticated`
5. 全專案測試（`accounts` + `documents`）共 15 個，全數通過
6. 程式碼已 push 到 `ai-document-platform`

**Sprint 2 正式收尾。**

---

## 核心概念整理

### 1. JWT 是無狀態（stateless）——每次請求都要重新帶 token

一開始誤以為「login 驗證過一次，之後 API 就不用再帶 token」。實際上 JWT 認證下，伺服器不會「記住」誰登入過，**每一次**呼叫需要登入的 API，都要在**這次請求本身**帶上 `Authorization: Bearer $TOKEN`，伺服器才知道「這次是誰在問」。比喻：像每次結帳都要重新出示會員卡，不是刷過一次以後都不用再拿出來。

### 2. curl 實測三支 API，結果都符合設計

```bash
# 登入拿 token（跟 Sprint 1 一樣，用 shell 變數存，不手動複製貼上）
TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/accounts/login/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "..."}' \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['access'])")

# 上傳（-F 是 form 格式，不是 -d 的 JSON 格式，因為要夾帶真實檔案）
curl -s -X POST http://127.0.0.1:8000/api/documents/upload/ \
  -H "Authorization: Bearer $TOKEN" \
  -F "title=測試文件A" \
  -F "file=@test.txt"
# → 201，回傳 {id, title, file, uploaded_at}，沒有 user 欄位（設計如此）

# 列表
curl -s -X GET http://127.0.0.1:8000/api/documents/ \
  -H "Authorization: Bearer $TOKEN"
# → 200，回傳陣列，只有自己的文件

# 刪除（-i 讓 curl 顯示 HTTP 狀態碼，因為刪除成功通常不回傳內容）
curl -s -i -X DELETE http://127.0.0.1:8000/api/documents/2/ \
  -H "Authorization: Bearer $TOKEN"
# → 204 No Content
```

`file=@test.txt` 裡的 `@` 符號代表「這是一個檔案，不是文字」，curl 會讀取檔案實際內容夾帶上去。

`204 No Content`：刪除成功的標準狀態碼，代表「成功，但沒有內容要回傳」，業界對刪除操作的標準做法。

### 3. HTTP 狀態碼：401 / 403 / 404 的分辨

這次設計測試案例時，第一次完整釐清三者的差異：

- **401 Unauthorized**：「你根本沒帶身份證來」——沒有提供有效的登入憑證（沒帶 token，或 token 過期/無效）
- **403 Forbidden**：「你有帶身份證，但這張證件不准你進這個房間」——系統已經知道你是誰，但判斷你沒有權限做這件事
- **404 Not Found**：這筆資料**根本不在你能查詢的範圍裡**，對系統來說「找不到」，不是「主動拒絕」

**這次專案的關鍵判斷**：A 使用者想刪 B 使用者的文件，該回 403 還是 404？

因為 `get_queryset` 已經用 `.filter(user=self.request.user)` 把查詢範圍**從一開始就限縮成「只有屬於自己的文件」**，B 的文件根本不在 A 能查到的範圍內。這不是「系統知道有這筆資料、主動判斷你沒權限」（那樣才是 403），而是「這筆資料對你來說本來就不存在」，所以回 **404**。

這個設計方式還有一個好處：**不會洩漏「這筆資料到底存不存在」**。如果回 403，攻擊者至少能確定「這個 id 真的有資料，只是我不能動」；回 404 的話，攻擊者完全無法分辨「沒有這筆資料」還是「有，但不是我的」。

### 4. 測試案例設計邏輯：正常 / 未登入 / 跨使用者

三支 API，每支都用同一套邏輯去想測試案例：

```
上傳：test_upload_success, test_upload_unauthenticated
列表：test_list_success, test_list_unauthenticated, test_list_only_shows_own_documents
刪除：test_delete_success, test_delete_unauthenticated, test_delete_other_users_document（預期 404）
```

判斷方法：「正常情況」一定要測；「未登入會不會被擋」是身份驗證的根本防線；「列表/刪除」還要多測「跨使用者」情境，因為這是整個 Documents 功能的核心安全原則（每人只能碰自己的東西），必須用測試證明系統真的有守住，不能只憑感覺。

### 5. `DocumentTests` 的新技巧

**`setUp`：測試共用的前置資料**
```python
def setUp(self):
    self.user_a = User.objects.create_user(username='user_a', password='testpass123')
    self.user_b = User.objects.create_user(username='user_b', password='testpass123')
```
每個測試方法執行前都會先跑一次 `setUp`，適合放「每個測試都會用到的共同資料」。這次因為要測「使用者之間互不干擾」，第一次在 `setUp` 裡建立了**兩個**使用者。

**注意**：`setUp` 裡不要放 `force_authenticate`（登入動作），因為不同測試需要的登入狀態不一樣（有的要登入、有的要故意不登入），登入與否要留給各自的測試方法決定。

**`force_authenticate`：測試環境直接「假裝」登入**
```python
self.client.force_authenticate(user=self.user_a)
```
測試環境不需要真的走一次 login API 拿 token（太麻煩），這個方法可以直接讓測試客戶端「假裝」是某個使用者已登入。可以重複呼叫來切換身份（例如先是 `user_a`，中途切換成 `user_b`），也可以傳 `user=None` 代表「登出」。

**測試中動態建立檔案**
```python
with open('test.txt', 'w') as f:
    f.write('測試內容')
with open('test.txt', 'rb') as f:
    response = self.client.post(reverse('document-upload'), {'title': '...', 'file': f})
```
測試要能在任何環境重複執行，不能依賴「電腦裡剛好有某個檔案」，所以用程式碼當場產生一個小檔案。`'w'`（write，寫入文字模式）建立內容，`'rb'`（read binary，讀取二進位模式）重新打開來夾帶上傳。

**`reverse()` 反查網址，`args` 傳入路徑參數**
```python
reverse('document-list')                    # 對應沒有參數的路徑
reverse('document-delete', args=[doc_id])   # 對應 <int:pk> 這種需要參數的路徑
```
用 `urls.py` 裡設定的 `name=...` 反查出實際網址，不用手打字串路徑，以後路徑改了測試也不用跟著改。有路徑參數（如 `<int:pk>`）時用 `args=[值]` 傳入。

**從 `response.data` 取出剛建立資料的 `id`**
```python
doc_id = response.data['id']
```
`response.data` 是上傳 API 回傳的 JSON，可以當成 Python 字典操作，直接用 key 取值。這是 `test_delete_success` 等測試裡，先上傳拿到 `id`、再用這個 `id` 去刪除，串起完整流程的關鍵寫法。

### 6. 意外發現的安全漏洞：三支 View 沒有限制「必須登入」

寫 `test_upload_unauthenticated`（驗證未登入應該被拒絕）時，測試直接讓程式當機：

```
ValueError: Cannot assign "<django.contrib.auth.models.AnonymousUser object at ...>":
"Document.user" must be a "User" instance.
```

**原因拆解**：`DocumentUploadView` 原本完全沒有設定「這支 API 必須登入才能用」的規則。一個沒登入的請求，一路跑進 `perform_create`，這時候 `self.request.user` 不是 `None`，而是 Django 給的特殊佔位符 `AnonymousUser`（代表「這是一個訪客」）。程式碼硬要把這個「訪客」塞進 `Document.user`（這個欄位要求必須是真正註冊過的 `User`），直接爆炸。

**更隱蔽的問題**：如果同樣情況發生在「列表」的 `get_queryset`（`.filter()` 而不是 `.create()`），資料庫查詢通常不會直接報錯，而是**悄悄回傳空列表**——沒有明確拒絕（沒有 401），只是安靜地放行、給出一個「看起來正常、其實不該發生」的結果。這種不報錯但邏輯錯誤的狀況，比直接當機更難被發現。

**修正方式**：三支 View 都加上

```python
from rest_framework.permissions import IsAuthenticated

class DocumentUploadView(generics.CreateAPIView):
    serializer_class = DocumentSerializer
    permission_classes = [IsAuthenticated]
    ...
```

`IsAuthenticated` 是 DRF 內建的權限規則，會在請求進到 View 的邏輯（`perform_create`、`get_queryset` 等）**之前**，先檢查「這個人有沒有登入」。沒登入的話直接回 401，請求根本不會深入到後面的程式碼，自然不會發生剛剛那種意外。

**這次學到的原則**：一個良好設計的 API，遇到未登入的請求，應該「一開始就在門口被禮貌地擋下來，回一個清楚的 401」，而不是「讓它進來、卡在中間才意外爆炸出一堆看不懂的紅字」。這也是為什麼寫測試很有價值——`test_upload_unauthenticated` 這個案例親手抓到了一個真實存在的漏洞，不是紙上談兵。

---

## 過程中排查過的問題

### 存檔沒有真的寫進檔案

用編輯器存檔後，`cat documents/tests.py` 卻顯示還是 Django 自動產生的原始空白樣板，代表編輯器畫面看起來對，實際上沒有真的存進硬碟（可能是分頁認錯檔案）。**教訓**：不能只信任編輯器畫面，存檔後養成用 `cat 檔案路徑` 從終端機再次確認內容的習慣。之後改用 `cat > 檔案 << 'EOF' ... EOF` 這種指令直接寫入檔案，繞過編輯器，更保險。

---

## 還不確定、需要之後多練習的地方

1. **`cat >> 檔案 << 'EOF'` 這種一次性寫入多行程式碼的指令**：這次是遇到編輯器存檔異常才改用的應急方案，語法還不熟，之後可以多留意這種 heredoc 寫法的使用時機。
2. **`permission_classes` 除了 `IsAuthenticated`，DRF 還有哪些現成的權限規則**：這次只學了「必須登入」這一種，之後如果需要更細緻的權限控管（例如管理員專屬功能），可以再深入了解。
3. **測試案例之間會不會互相影響**：目前每個測試方法都是各自獨立呼叫 `setUp`、各自建立自己需要的資料，這個機制背後怎麼保證「測試資料庫每次都是乾淨的」，還沒完全搞懂原理，先照著用，之後有機會可以問清楚。

---

## 下一步

Sprint 2（Documents App）正式完成，包含：App、Model、Migration、後台管理、media 架構、Serializer、三支 View、urls、curl 實測、完整自動化測試（含權限漏洞修正）。

- 更新 `progress.md`，反映 Sprint 2 完整收尾的狀態
- 視需要更新履歷用的專案描述
- 進入 **Sprint 3**：AI 問答功能（RAG），這是整個「AI 文件管理平台」的核心賣點，預期會接觸 LLM API、向量搜尋/embedding 等全新技術領域
