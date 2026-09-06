# Sprint 2 — Document 後台註冊、除錯、密碼救援與 media 架構修正

## 今天完成的事

1. 把 `Document` model 註冊到 Django 後台管理介面（`documents/admin.py`）
2. 排查並修正一個 `ImportError`（大小寫錯誤）
3. 忘記後台 superuser 密碼，完成一次密碼救援流程
4. 在後台實際新增一筆 `Document` 測試資料，驗證 model 真的運作
5. 發現「上傳檔案跟 App 程式碼存在同一個資料夾」的架構問題，修正為業界標準做法：獨立的 `media/` 資料夾
6. 清理誤入版控的測試檔案，確認 `git status` 乾淨後，成功 push 到 GitHub

---

## 核心概念整理

### 1. Django admin 註冊 model 的固定寫法

```python
from .models import Document
from django.contrib import admin

admin.site.register(Document)
```

- `from .models import Document`：只是把 model 這張「設計圖」拿到手邊，**不會**自動讓它出現在後台。
- `admin.site.register(Document)`：真正的「登記」動作，跟後台系統說「請把這個 model 加進管理清單」。
- 比喻：`import` 是把表格拿在手上，`register` 才是走去窗口登記蓋章。**兩者缺一不可**，這是 Django admin 的固定規則，不是能靠邏輯推出來的部分，需要記住。
- 驗證方式：只 import 不 register → 後台**不會**顯示這個 model。

### 2. Python 是大小寫敏感的（case-sensitive）

- 這次踩到的坑：`models.py` 裡定義的是 `Document`（大寫 D），但 `admin.py` 裡不小心打成 `document`（小寫 d）。
- 對 Python 來說，`Document` 跟 `document` 是**兩個完全不同的名字**，即使拼字看起來很像。
- 錯誤訊息：
  ```
  ImportError: cannot import name 'document' from 'documents.models'
  ```
  這句話直翻就是：「我沒辦法從 `documents.models` 裡面，把一個叫 `document` 的東西 import 進來」——因為那裡面根本沒有叫這個名字（小寫）的東西，只有 `Document`（大寫）。
- **讀錯誤訊息的習慣**：先看**最後一行**（通常是最關鍵的結論），再往上找是哪一行程式碼觸發的。

### 3. 忘記 superuser 密碼的救援流程

**情況一：忘記帳號名稱**（查詢，不會洩漏密碼本身）：
```bash
python manage.py shell
```
進入 `>>>` 互動模式後：
```python
from django.contrib.auth.models import User
User.objects.filter(is_superuser=True).values_list('username', flat=True)
```
查完打 `exit()` 離開。

**情況二：知道帳號、重設密碼**：
```bash
python manage.py changepassword 帳號名稱
```
- 這行指令必須在**專案資料夾**下執行（要有 `manage.py` 這個檔案在同一層），不然會出現：
  ```
  can't open file 'manage.py': [Errno 2] No such file or directory
  ```
- 這次也踩到一個小提醒：`python manage.py shell` 打開後才有 `>>>` 提示字元，裡面才能貼 Python 語法（例如 `from django...`）；如果直接把這段貼到一般的 bash 終端機，會出現 `syntax error`，因為 bash 看不懂 Python 語法。

**Django 密碼強度檢查**：
- 系統會擋掉「少於 8 字元」或「純數字」的密碼，這是保護機制不是錯誤：
  ```
  This password is too short. It must contain at least 8 characters.
  This password is entirely numeric.
  ```
- 改用「英文字母 + 數字混合、至少 8 碼」即可通過。

### 4. 修改 model 欄位參數，要不要重跑 migration？

- 這次把 `upload_to='documents/'` 改成 `upload_to='uploads/'`，**欄位型態完全沒變**（還是 `FileField`），一開始直覺覺得「不用跑 migration」。
- 但實際上**要跑**。原因：Django 判斷要不要記錄變化，看的是**欄位的完整定義**（包含所有參數），不是只看型態。只要 `models.py` 有任何改動，就要跑一次 `makemigrations` 讓 Django 記錄下來。
- 執行結果會看到符號 `~`（代表「修改」，不是新增 `+` 也不是刪除 `-`）：
  ```
  Migrations for 'documents':
    documents/migrations/0002_alter_document_file.py
      ~ Alter field file on document
  ```

### 5. 上傳檔案不該跟 App 程式碼放在同一個資料夾

**這次發現的問題**：一開始 `upload_to='documents/'`，結果檔案被存進 `documents/images.jfif`——這個路徑剛好跟 `documents` **App**（放 `models.py`、`admin.py` 的地方）撞名，檔案跟程式碼混在一起了。

**業界標準做法**：獨立開一個 `media/` 資料夾，專門放使用者上傳的檔案，跟程式碼資料夾完全分開。

**設定方式**（`config/settings.py` 最後面加兩行）：
```python
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```
- `MEDIA_ROOT`：告訴 Django「使用者上傳的檔案，實際要存在硬碟的哪個資料夾」。
- `MEDIA_URL`：之後要讓瀏覽器讀取這些檔案時，網址要用哪個開頭。

**搭配修改 model**：
```python
file = models.FileField(upload_to='uploads/')
```
- `upload_to='uploads/'` 是 `MEDIA_ROOT` **底下**的子資料夾名稱，改成 `uploads` 避免再跟 App 名稱 `documents` 搞混。
- 驗證方式：
  ```bash
  find media/ -type f
  ```
  看到 `media/uploads/檔名`，代表路徑設定正確生效。

**加進 `.gitignore`**：
```
media/
```
- 跟 `db.sqlite3` 是同一種邏輯：這是「使用者實際產生的資料」，不是「程式碼」，不該上傳到 GitHub，否則以後每次測試上傳都會多出新檔案，越塞越多。

### 6. 「同名不同地方」的陷阱，這次又遇到一次

跟交接文件裡提過的 `learning-notes` 陷阱一樣的邏輯：專案裡同時有
- `documents` **App**（程式碼：`models.py`、`admin.py`、`migrations/`）
- 一開始誤用的 `documents/` **上傳資料夾**（後來改成獨立的 `media/`）

這次问题正是因為兩者一開始用了同一個名字，才會在 `git status` 看到不該出現的 `documents/images.jfif`。**教訓**：資料夹命名要盡量避免跟現有的 App／模組名稱重複，尤其是「程式碼」跟「使用者資料」這種性質完全不同的東西，一開始就該分開想清楚放哪。

### 7. Django admin 列表預設顯示名稱

新增資料後，列表顯示 `Document object (1)` 而不是你填的 Title 內容——這**不代表資料遺失**，只是 Django 預設不知道要拿哪個欄位當「顯示名稱」，用了很籠統的格式 `模型名稱 object (編號)`。點進去看表單內容，資料其實都還在。（之後可以在 `admin.py` 用 `list_display` 或 model 的 `__str__` 方法自訂顯示名稱，這是留給之後練習的細節，這次沒有動手做。）

---

## 還不確定、需要之後多練習的地方

1. **Django admin 的 `list_display` / `__str__` 自訂顯示名稱**：這次只是發現「顯示成 `Document object (1)` 是正常現象」，還沒有動手練習怎麼把它改成顯示 Title 內容，之後可以找機會補上。
2. **忘記密碼救援流程的指令組合**（`shell` 查帳號 → `exit()` → `changepassword`）：這是這次臨時狀況下現學的，還不熟練，需要之後再遇到類似狀況（例如換一台電腦、重建開發環境）時多操作幾次才會內化。
3. **判斷「檔案該放程式碼資料夾還是獨立資料夾」的直覺**：這次是事後才發現問題，之後開新 App、設計會產生檔案的功能時，可以養成「先想清楚這個東西是程式碼還是資料」的習慣，一開始就分清楚，減少事後修正的狀況。

---

## 下一步

- 設計並實作 Documents 的 API（上傳、列表、刪除），流程比照 Sprint 1：先設計思考、再拆小塊寫程式碼
- 之後補上自動化測試（`DocumentTests`），比照 `RegisterTests` / `LoginTests` / `MeViewTests` 的模式
