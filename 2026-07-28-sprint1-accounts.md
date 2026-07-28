# Sprint 1：帳號模組（Accounts）

日期：2026-07-28
專案：AI 文件管理平台
本次目標：設計並實作帳號模組——先想清楚 API 設計、資料庫設計、認證方式，再落地成程式碼，最後在後台驗證真的運作

---

## 一、目前進度

- [x] 設計思考：確認認證方式、API 端點、資料庫設計
- [x] 建立 `accounts` App，登記進 `INSTALLED_APPS`
- [x] 設計 `UserProfile` model（一對一連結 Django 內建 User）
- [x] 執行 migration，套用到資料庫
- [x] 註冊到後台管理介面，建立管理員帳號並驗證
- [ ] 實作真正的 API：`register` / `login` / `me`（下一步）

---

## 二、設計思考

### 核心問題：伺服器怎麼「記住」使用者已經登入？

HTTP 是「無狀態」的，伺服器不會自動記得上一次請求發生過什麼事。不能用 IP 判斷身份（同一網路底下的人常共用對外 IP，且 IP 本身會變動）。需要額外機制：

| | Session | Token（JWT） |
|---|---|---|
| 運作方式 | 登入後伺服器建記錄，發識別碼給瀏覽器；之後每次請求都要查記錄 | 登入後發一張「已簽名」的通行證；之後每次請求只要驗證簽名，不用查資料庫 |
| 適合場景 | 傳統網頁 | API 服務、手機 App、前後端分離架構 |
| 多台伺服器擴充 | 較麻煩（要共享記錄） | 簡單（任何一台都能自己驗證） |

**本專案選擇 Token（JWT）**：因為是 Django REST Framework 的 API 服務，未來可能有網頁、App 多方存取，JWT 不依賴伺服器端狀態，更適合這種架構。

**Token 實際運作流程**：登入成功 → 伺服器發 Token → 用戶端（App/瀏覽器）存在本地 → 之後每次請求自動夾帶 Token → 伺服器驗證簽名即可放行，不用重新輸入密碼。Token 有效期限到了才需要重新登入（刻意的安全機制，避免 Token 永久有效、外流後被永久冒充）。

### API 端點設計

判斷 GET / POST 的原則：**要資料（查看、瀏覽）→ GET；動資料（新增、修改、含機密資訊）→ POST**。登入不能用 GET，因為 GET 的參數會顯示在網址列、被瀏覽器歷史與伺服器 log 記錄，帳號密碼會變成明文外洩。

```
POST /api/accounts/register/   ← 新增使用者
POST /api/accounts/login/      ← 驗證帳號密碼，成功後發 Token
GET  /api/accounts/me/         ← 查看目前登入者的基本資料
```

### 資料庫設計：Profile 模式

不自己從零設計使用者表（密碼加密等資安邏輯不該自己重刻），而是保留 Django 內建 User，另外用「一對一（One-to-One）」外掛一張 `UserProfile` 表存放額外資訊。

判斷一對一 vs 一對多的原則：想「現實生活中，這兩個東西的關係，合不合理是『一個對一個』還是『一個對很多個』」。`User─UserProfile` 是一對一（一個人只該有一份個人檔案）；`User─Document`（之後會做的）是一對多（一個人可以有很多份文件）。

### 認證套件選型

使用 `djangorestframework-simplejwt`——業界標準、經過廣泛驗證的 JWT 套件，不自己手刻加密邏輯。

---

## 三、實作記錄

### App 建立與登記，是兩個分開的步驟
```bash
python manage.py startapp accounts
```
只是建立資料夾與樣板檔案，Django **不會自動知道要用它**，必須手動到 `config/settings.py` 的 `INSTALLED_APPS` 加上 `'accounts',`。

### Migration：把設計圖翻譯成真正的資料庫結構
```bash
python manage.py makemigrations accounts   # 產生「施工說明書」
python manage.py migrate                    # 照著說明書真的修改資料庫
python manage.py showmigrations accounts    # [X] 代表已套用
```

### UserProfile model
```python
from django.db import models
from django.contrib.auth.models import User

class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    job_title = models.CharField(max_length=100, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.user.username
```
- `on_delete=models.CASCADE`：對應的 User 被刪除時，這筆 Profile 也跟著刪除
- `blank=True`：非必填欄位
- `auto_now_add=True`：第一次建立時自動填入時間，之後不變

### 註冊到後台管理介面
```python
# accounts/admin.py
from django.contrib import admin
from .models import UserProfile

admin.site.register(UserProfile)
```
建立管理員帳號：`python manage.py createsuperuser`。後台網址是 `http://127.0.0.1:8000/admin/`。

### 實際驗證一對一關聯生效
在後台建立一個測試使用者，再建一筆 Profile 連結到他。驗證重點：**一旦某個 User 已連結過一筆 Profile，下拉選單就不能再選它連結第二筆**——證明這不是靠自己手動檢查，而是資料庫結構本身強制執行了一對一規則。

---

## 四、踩坑軌跡（僅記錄理解錯誤，操作手滑不記）

**問題：以為 `pip install django` 裝錯資料夾，導致 Django 沒裝成功**
- 原因：不理解「虛擬環境啟動後，是整個終端機 session 生效，不是針對資料夾生效」，看到自己身處錯誤資料夾，就誤以為套件也裝錯地方。
- 釐清：實際上 Django 有正確裝進 `ai_document_platform/venv`，真正錯誤只有 `requirements.txt`（這個檔案是依「目前所在資料夾」寫入的，跟套件裝在哪裡是兩件事）放錯位置。
- 學到的概念：虛擬環境的生效範圍是「終端機視窗」，不是「資料夾」；但某些指令的輸出檔案位置，仍然跟著「目前所在資料夾」走，兩者要分開判斷。

**問題：第一次 `git push` 後，`__pycache__/` 與 `db.sqlite3` 被一起上傳到 GitHub**
- 原因：不理解「哪些東西不該進版控」——`__pycache__` 是 Python 自動產生的暫存編譯檔，`db.sqlite3` 是會一直變動的資料庫檔案，兩者都不是「原始碼」，不該被 Git 追蹤。建立專案骨架後沒有先設 `.gitignore` 就直接 commit。
- 解法：補上 `.gitignore`（排除 `venv/`、`__pycache__/`、`*.pyc`、`db.sqlite3`、`.env`），並用 `git rm -r --cached` 把已經追蹤的檔案從 Git 紀錄中移除（`--cached` 只從追蹤清單移除，不會刪除本機檔案）。
- 學到的概念：建立新專案骨架後，第一件事就該先設好完整的 `.gitignore`，再做第一次 commit。

---

## 五、下次要做的事：真正的 API

1. 安裝 `djangorestframework` 與 `djangorestframework-simplejwt`
2. 實作 `POST /api/accounts/register/`
3. 實作 `POST /api/accounts/login/`（串接 simplejwt，回傳 Token）
4. 實作 `GET /api/accounts/me/`
5. 用 Postman 或類似工具，逐一測試三支 API
