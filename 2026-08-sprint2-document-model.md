# Sprint 2 開始 — Document Model 設計與建立

## 今天完成的事

正式進入 Sprint 2：文件管理模組（Documents App）。完成 App 建立、`Document` model 設計與實作、`makemigrations` + `migrate`，資料庫已經有 `Document` 這張表。中間也做了一次「重新開始前的暖身」，確認舊進度（Sprint 1 的 7 個測試）都還完好。

### 建立 App
```bash
python manage.py startapp documents
```
跟 `accounts` App 一樣的流程：先建立資料夾與樣板檔案，接著要手動登記進 `config/settings.py` 的 `INSTALLED_APPS`，Django 才會真的啟用它。

```python
INSTALLED_APPS = [
    ...
    'accounts',
    'documents',
    'rest_framework',
]
```

### Document model
```python
# documents/models.py
from django.db import models
from django.contrib.auth.models import User


class Document(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    file = models.FileField(upload_to='documents/')
    title = models.CharField(max_length=255)
    uploaded_at = models.DateTimeField(auto_now_add=True)
```

### Migration
```bash
python manage.py makemigrations documents
# Migrations for 'documents':
#   documents/migrations/0001_initial.py
#     + Create model Document

python manage.py migrate
# Applying documents.0001_initial... OK
```

---

## 今天學到的核心概念

### 一對多關係：`ForeignKey`
- Sprint 1 學過「一對一」用 `OneToOneField`（如 `User─UserProfile`）。
- 這次判斷「一個使用者可以上傳很多份文件」是一對多關係，Django 對應的語法是 `ForeignKey`，不是直覺猜測的 `OneToMoreField`（這是 Django 固定命名，屬於規則記憶，不是完全能用邏輯推出來的）。
- 比喻：每份文件上貼一張小紙條寫著「屬於誰」，多份文件的小紙條可以寫同一個人的名字，但每張紙條只能寫一個人——這就是「多的一方，各自指向同一個一」。

### `on_delete=CASCADE` vs `PROTECT`
- `CASCADE`（連鎖）：主要角色（User）被刪除時，附屬資料（Document）跟著一起自動刪除。
- `PROTECT`（保護）：只要附屬資料還存在，系統會直接拒絕刪除主要角色。
- 這次的設計討論：一開始直覺想選 `PROTECT`（怕連鎖刪除搞壞系統），但考量到「這是第一版學習型專案」「大部分真實服務如 Google／Facebook 刪帳號預設就是連鎖清空內容」，第一版先選 `CASCADE`，之後有需要可以再調整成 `PROTECT`（只要改一個參數，不是不可逆的決定）。

### 檔案不該直接存進資料庫
- 資料庫設計來存「結構化的小資料」（文字、數字、日期），不適合存大型二進位檔案（例如 500MB 影片）。
- 如果硬塞進資料庫，會導致：資料庫檔案暴增、讀寫效能明顯變差、備份與搬遷變得困難。
- 業界標準做法：**檔案本身存在硬碟（或雲端儲存空間），資料庫只存「路徑」這串文字**。
- 比喻：資料庫像圖書館的目錄卡片，卡片上寫「這本書放在 3 樓 B 區 15 號書架」，不會把整本書內容印在卡片上。

### `FileField`
```python
file = models.FileField(upload_to='documents/')
```
- Django 內建的欄位類型，`upload_to='documents/'` 告訴 Django「使用者上傳的檔案，自動存到專案資料夾底下的 `documents/` 子資料夾」。
- 不用自己處理「怎麼存檔案」「檔名重複怎麼辦」這些細節，Django 都會處理好。
- 資料庫裡實際存的內容，是類似 `documents/report.pdf` 這樣一串文字路徑，不是檔案本身。

### 為什麼要多存一個 `title`（原始檔名）欄位
- 系統為了避免檔名重複，常會把實際存在硬碟上的檔名改成一串亂碼（例如 `documents/a8f3e91c.pdf`）。
- 使用者光看這串亂碼路徑，完全無法辨識這是自己上傳的哪份文件。
- 所以額外用 `CharField` 存一份「原始檔名」，跟 `file`（實際存放路徑）分開，兩者用途不同：
  ```python
  title = models.CharField(max_length=255)
  ```
  - `CharField`：存文字/字串型態的資料（型態），跟用途是兩件事要分開想——`max_length=255` 是「這個欄位最多存幾個字」的限制規則，不是欄位裡實際存的內容。
  - 實際存的內容範例：`"resume.pdf"`。

### `DateTimeField(auto_now_add=True)` 的複習應用
- 跟 Sprint 1 的 `UserProfile.created_at` 邏輯完全一樣：這筆資料第一次建立時自動記錄當下時間，之後修改其他欄位也不會被改變。
- 套用到 Document，欄位改名成 `uploaded_at`，語法邏輯不變。

---

## 今天的暖身環節（隔一段時間沒碰專案後的作法）

因為隔了一段時間沒碰專案，重新開始前先做了三件事，確認狀態，這個習慣值得保留，下次隔久沒碰也可以先做一次：
1. 確認 `(venv)` 有沒有啟動
2. 重新跑一次舊測試（`python manage.py test accounts`），確認先前寫的東西沒壞（結果：`Found 7 test(s)`，全過）
3. 用自己的話重新講一次舊概念（Register / Login / Me 三支 API 分別在做什麼），當作暖身小測驗

---

## 目前還不太確定、需要之後多練習的地方

1. **「觀念懂、但 code 手感模糊」**：這次自己承認這個狀態，這是正常的學習階段，觀念判斷力（一對多、CASCADE vs PROTECT、檔案該不該存資料庫）已經建立得不錯，但實際打程式碼、跑指令的熟練度還需要更多次重複練習才會內化，不是聽解釋就能跳過的部分。
2. **`cd` / `pwd` 這類基本指令仍需要邊查邊打**：這也是需要靠重複次數累積的肌肉記憶，不是概念問題。
3. **`git status` / `git add . / commit / push` 目前是複製貼上執行**：這組指令本身是固定公式，很多工程師也是直接打不特別去背，重點是理解每一步在做什麼（這點已經沒問題），純粹敲鍵盤的熟練度會隨時間自然提升。

---

## 下一步

- 把 `Document` 註冊到後台管理介面（`documents/admin.py`），像 Sprint 1 的 `UserProfile` 一樣，實際在後台建立一筆資料驗證 model 真的運作
- 接著設計並實作 Documents 的 API（上傳、列表、刪除等），流程會比照 Sprint 1：先設計思考、再拆小塊寫程式碼
- 之後補上自動化測試（`DocumentTests`），比照 `RegisterTests` / `LoginTests` / `MeViewTests` 的模式
