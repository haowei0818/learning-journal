# Sprint 2 — Documents API 設計與實作（Serializer / Views / URLs）

## 今天完成的事

1. 設計思考：確定 Documents 功能範圍先做 MVP（上傳、列表、刪除），暫緩共享/版本管理/重新命名等進階功能
2. 確定三支 API 的 HTTP 方法與路徑
3. 寫完 `DocumentSerializer`
4. 寫完三支 View：`DocumentUploadView`、`DocumentListView`、`DocumentDeleteView`
5. 寫完 `documents/urls.py`，並掛載進 `config/urls.py`
6. `runserver` 驗證整條路徑沒有語法/設定錯誤

---

## 設計思考整理

### MVP 概念：先求「能動」，再求「完整」

一開始想到文件管理系統可能需要「共享、版本管理、重新命名、詳細資料」等功能，但考量專案目前階段（履歷作品、Sprint 2 剛起步、還沒有 AI 問答功能），決定先做**最基本、非做不可**的三個動作：上傳、列表、刪除，之後有餘力再擴充進階功能。這叫 **MVP（最小可行產品）**：先把核心骨架做出來、能動、能展示，比一次求全但卡住不動更重要。

### 核心安全原則：每個使用者只能碰自己的文件

貫穿整個 Documents 功能的關鍵原則——**列表只能看自己的、刪除只能刪自己的，不能碰別人的**。這個原則會反覆用同一種手法實現：篩選查詢範圍（見下方 `get_queryset`）。

### 三支 API 的路徑與方法設計

判斷 HTTP 方法的核心原則：
- `GET` → 只是「查詢」已存在的資料，不會讓資料庫多或少東西
- `POST` → 「新增」一筆全新的資料
- `DELETE` → 刪除資料

對照到 Documents：

| 動作 | HTTP 方法 | 路徑 |
|---|---|---|
| 上傳 | `POST` | `/api/documents/upload/` |
| 列表 | `GET` | `/api/documents/` |
| 刪除 | `DELETE` | `/api/documents/<id>/` |

**「上傳」為什麼是 `POST`**：一開始直覺猜 `GET`，後來對照 Sprint 1 的 `register`（新增一個 User，用 `POST`），才想清楚「上傳」也是「建立一筆全新的資料」，跟查詢的本質不同，應該用 `POST`。

**「刪除」路徑裡的 `<id>`**：一開始猜是「檔案名稱」，但檔名可能重複（例如兩份都叫 `report.pdf`），無法唯一辨識是哪一筆。真正該用的是 `id`——Django 幫每個 model 自動加上的欄位，型態是整數、從 1 開始遞增，**資料庫層級保證不會重複**（這跟自己想的 `uploaded_at` 不同：時間理論上還是有極小機率撞到，但 `id` 完全不會）。這也是後台看到 `Document object (1)`、`(2)` 括號裡數字的由來。

---

## 核心概念整理

### 1. `ModelSerializer` 的欄位開放範圍是安全關鍵

```python
from rest_framework import serializers
from .models import Document


class DocumentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Document
        fields = ['id', 'title', 'file', 'uploaded_at']
```

- `ModelSerializer` 是 DRF 提供的「方便版表格」：只要告訴它對應哪個 model、要哪些欄位，會自動幫每個欄位產生驗證規則，不用像 Sprint 1 手動一個一個寫。
- **關鍵判斷**：`fields` 清單裡**故意不放 `user`**。原因是只要某個欄位出現在 `fields` 裡，DRF 就會自動讓它變成「可以被填寫」的欄位（除非另外設成唯讀）。如果 `user` 也放進去，前端理論上就能在請求裡夾帶 `user` 欄位，冒充自己是別的帳號上傳文件——這是真實存在的資安風險（概念上稱作 mass assignment，這次先理解現象、不用記英文名詞）。
- **設計原則**：能不給的欄位，就一開始不要放進 Serializer，不要等出問題才補洞。

### 2. `self.request.user`：系統自動判斷「這次請求是誰發出的」

這是今天學的新技巧，串起 Sprint 1 跟今天的知識：

```python
# Sprint 1，MeView 已經用過
def get_object(self):
    return self.request.user
```

```python
# 今天，DocumentUploadView
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```

`self.request.user` 代表「目前這次 API 請求，是哪一個已登入的使用者發出的」——這是 DRF 根據登入時的 JWT token 驗證出來的，**使用者自己沒辦法竄改冒充**，不需要使用者在表單裡填寫。

`perform_create` 這個方法的意思是：「在真正把資料存進資料庫之前，額外補一個資訊進去」——這裡補的是把 `user` 欄位設成「目前登入的這個人」，達成「系統自動判斷、不讓使用者自己填」的設計目標。

### 3. `get_queryset`：篩選查詢範圍，是列表跟刪除的共同安全機制

```python
class DocumentListView(generics.ListAPIView):
    serializer_class = DocumentSerializer

    def get_queryset(self):
        return Document.objects.filter(user=self.request.user)


class DocumentDeleteView(generics.DestroyAPIView):
    serializer_class = DocumentSerializer

    def get_queryset(self):
        return Document.objects.filter(user=self.request.user)
```

- `Document.objects.filter(user=self.request.user)`：只挑出「`user` 欄位等於目前登入這個人」的資料。
- **如果寫成 `Document.objects.all()`**（不篩選，撈全部）：任何登入的人打開列表，會看到全平台所有人的文件，嚴重違反「只能碰自己文件」的原則。這種「程式碼能跑、不會報錯，但邏輯不安全」的狀況，是最容易被忽略的坑。
- **同一段邏輯保護了列表跟刪除兩支 API**：如果自己有 id 1、2、3 三份文件，別人有 id 4 一份，故意打 `DELETE /api/documents/4/`，因為 `get_queryset` 已經把範圍限縮成「只有屬於自己的 1、2、3」，系統會直接「找不到 id 4」（通常回應 `404`），不會真的刪到別人的文件。

### 4. DRF 現成的 View 積木（generics）

依照動作性質，各自借用對應的積木：

```python
DocumentUploadView(generics.CreateAPIView)   # 新增
DocumentListView(generics.ListAPIView)       # 列表
DocumentDeleteView(generics.DestroyAPIView)  # 刪除
```

判斷邏輯：「上傳」是新增一筆全新資料，跟 Sprint 1 的 `RegisterView(generics.CreateAPIView)` 本質相同，借用同一種積木。

### 5. `urls.py` 的固定寫法：`.as_view()` 與 `include()`

**App 內部的 `documents/urls.py`**：
```python
from django.urls import path
from . import views

urlpatterns = [
    path('upload/', views.DocumentUploadView.as_view(), name='document-upload'),
    path('', views.DocumentListView.as_view(), name='document-list'),
    path('<int:pk>/', views.DocumentDeleteView.as_view(), name='document-delete'),
]
```

- `.as_view()`：我們寫的 `DocumentUploadView` 等是 class（「還沒蓋章生效的設計圖」），Django 網址系統需要的是「真正能處理請求的東西」。`.as_view()` 就是把 class 轉換成可以真正拿來用的處理函式。**固定公式**：只要是用 `generics.XxxAPIView` 寫的 View，接到 `urls.py` 時一律加 `.as_view()`。
- `path('', ...)`：空字串代表「不加任何後綴，就是這個 App 的根路徑本身」，對應 `/api/documents/`（列表）不需要額外後綴的設計。
- `<int:pk>`：跟前端說「這個位置請填一個整數」，會被當作要操作的那筆資料的 `id`。`pk` 是 Django 的固定命名（primary key 的縮寫），代表我們一直在講的 `id` 欄位，這是規則記憶，不用糾結為什麼不直接叫 `id`。

**專案總表 `config/urls.py`，掛載新 App**：
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/accounts/', include('accounts.urls')),
    path('api/documents/', include('documents.urls')),
]
```

- `include('documents.urls')` 的意思：「去 `documents` 這個 App 資料夾裡，把裡面的 `urls.py` 拿出來套用」。**這裡容易搞混的地方**：`include()` 裡面填的是「資料夾名稱.urls」，不是檔名本身，也不是隨便一個看起來眼熟的字（這次一度誤打成 `config.urls` 跟 `serializers`，都是把「App 資料夾名稱」跟其他東西搞混了）。判斷方法：對照 `accounts` 那行的規律，`accounts.urls` 裡的 `accounts` 就是 App 資料夾的名字，所以換成 `documents` App 時，就是 `documents.urls`。

---

## 過程中排查過的問題

### 1. Class 縮排/巢狀寫錯

一開始不小心把 `class DocumentListView` 寫進 `class DocumentUploadView` 的 `perform_create` 方法裡面（縮排太深），導致兩個 View 被誤合併成一個。**判斷方法**：Python 用縮排決定「這段程式碼屬於誰管」，同一層級的 class 開頭縮排必須完全對齊（貼齊最左邊），互不隸屬。

### 2. `include()` 裡填錯資料夾名稱

分別誤寫成 `config.urls`、`serializers`。**判斷方法**：`include('資料夾名稱.urls')`，資料夾名稱要對照「這個 App 本身叫什麼名字」（用 `startapp` 建立時取的那個名字），不是隨便代入其他檔案或看起來相關的字。

### 3. 在錯的資料夾下指令（`learning-notes` vs `ai_document_platform`）

要找 `config/urls.py` 時，一度人在 `~/projects/learning-notes`（筆記 repo），導致 `find`/`cat` 都找不到檔案。**判斷方法**：`config/urls.py` 只存在於程式碼 repo `ai_document_platform`，不在筆記 repo `learning-notes` 裡。這跟交接文件裡提過的「同名資料夾陷阱」是同一種狀況的變形——這次不是同名，而是**在錯誤的 repo 底下找程式碼檔案**，提醒自己：找不到檔案時，先用 `pwd` 確認自己在哪個 repo，而不是急著懷疑檔案不見了。

### 4. Port 已被佔用

`Error: That port is already in use.`——通常是因為背景還有另一個終端機在跑 `runserver`。這次重跑後自動解決（推測是舊的分頁已經關閉、埠號被釋放），不算是程式碼問題。

---

## 還不確定、需要之後多練習的地方

1. **`include()` 裡「資料夾名稱.urls」這個固定格式的直覺反應**：這次連續猜錯兩次（`config.urls`、`serializers`），需要多寫幾次類似的掛載才能建立肌肉記憶。
2. **多個 View / Serializer 同時開著時的縮排警覺性**：程式碼變多之後，比較容易在複製貼上或手動修改時弄錯縮排層級，之後可以養成「寫完一段就退回最左邊、確認下一個 class 從頭開始」的習慣。
3. **實際用 curl 測試「需要先登入拿 token」的 API**：今天只驗證到 `runserver` 沒有語法錯誤，還沒有實際打過這三支 API，下一步要練習「先登入拿 token → 帶 token 呼叫上傳/列表/刪除」這個完整流程。

---

## 下一步

- 用 `curl` 實際測試三支 Documents API（上傳、列表、刪除），流程比照 Sprint 1：先登入拿 token，再帶 token 呼叫
- 之後補上自動化測試（`DocumentTests`），比照 `RegisterTests` / `LoginTests` / `MeViewTests` 的模式
