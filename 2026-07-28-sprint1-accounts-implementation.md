# Sprint 1 實作：建立 accounts App 與 UserProfile

日期：2026-07-28
專案：AI 文件管理平台
本次目標：把 Sprint 1 設計思考落地成真正的程式碼——建立 accounts App、UserProfile model，並在後台驗證一對一關聯真的生效

---

## 一、今天做了什麼（總覽）

- [x] 建立 `accounts` 這個 Django App
- [x] 把 `accounts` 登記進 `config/settings.py` 的 `INSTALLED_APPS`
- [x] 在 `accounts/models.py` 設計 `UserProfile` model（一對一連結 Django 內建的 User）
- [x] 執行 migration，把設計圖真正套用到資料庫
- [x] 在 `accounts/admin.py` 註冊 `UserProfile`，讓它顯示在後台管理介面
- [x] 建立管理員帳號（superuser），登入後台驗證
- [x] 手動新增一個測試使用者，並實際建立一筆連結到他的 `UserProfile`，驗證一對一關聯真的運作

---

## 二、概念筆記

### App 的建立與登記，是兩個分開的步驟
```bash
python manage.py startapp accounts
```
這行只是「建立資料夾與樣板檔案」，Django **不會自動知道要使用它**。必須手動到 `config/settings.py` 的 `INSTALLED_APPS` 清單裡加上 `'accounts',`，Django 才會真的去讀取這個 App 裡的 `models.py`、讓它的功能生效。這是新手很容易漏掉的一步。

### Migration：把「設計圖」翻譯成真正的資料庫結構
`models.py` 裡寫的 class，只是 Python 程式碼，**還沒有真的在資料庫裡建立對應的表格**。需要透過 migration 這個「翻譯」的過程，分兩步：

```bash
python manage.py makemigrations accounts   # 產生「施工說明書」，比對 models.py 跟目前資料庫的差異
python manage.py migrate                    # 照著說明書，真的去修改 db.sqlite3
```

拆成兩步的好處：可以在真正動手改資料庫之前，先「預覽」Django 打算做什麼改動，設計錯了可以先喊停，不會直接搞壞資料庫。

驗證 migration 是否已套用：
```bash
python manage.py showmigrations accounts
# [X] 代表已經套用到資料庫了
```

### UserProfile model 的寫法與每一行的意義
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
- `OneToOneField(User, on_delete=models.CASCADE)`：對應「一對一」關聯，`on_delete=models.CASCADE` 代表「如果對應的 User 被刪除，這筆 Profile 也要跟著一起刪除」
- `blank=True`：這個欄位可以留空，不是必填
- `auto_now_add=True`：第一次建立時自動填入現在時間，之後不會再變動
- `__str__`：讓後台管理介面顯示「這筆 Profile 屬於哪個使用者的名字」，而不是顯示一串看不懂的編號

### Django 後台管理介面（Admin）
Django 內建一個管理後台，不用寫任何前端頁面，就能用瀏覽器直接查看、新增、編輯資料庫裡的資料，開發階段檢查資料非常方便。

新建立的 model 預設不會自動出現在後台，需要在 `accounts/admin.py` 手動註冊：
```python
from django.contrib import admin
from .models import UserProfile

admin.site.register(UserProfile)
```

登入後台需要一組管理員帳號：
```bash
python manage.py createsuperuser
```
後台網址是 `http://127.0.0.1:8000/admin/`（一般網站首頁是 `http://127.0.0.1:8000/`，後台要多加 `/admin/`）。

### Django 密碼強度驗證
`createsuperuser` 建立帳號時，Django 內建會檢查密碼是否「太短（少於 8 碼）」或「全部是數字」，並跳出警告詢問是否要略過驗證。不建議選擇略過，即使是本機開發用的帳號，也應該養成設定較強密碼的習慣。這是 Django 內建幫忙做的資安把關，不需要自己額外寫驗證邏輯。

### 實際驗證「一對一」關聯真的生效
在後台手動測試：建立一個新的一般使用者（testuser），再到「用戶個人資料」新增一筆 Profile，把它連結到 testuser。

驗證重點：**一旦某個 User 已經連結過一筆 Profile，在下拉選單裡就不能再被選去連結第二筆**——這證明「一對一」的規則，不是靠自己手動檢查，而是資料庫結構本身就不允許重複連結，Django 在底層真正幫忙強制執行了這個設計。

### Django 後台會自動翻譯成中文
Django 管理後台會依照瀏覽器語言設定，自動顯示對應語言的介面（不是瀏覽器翻譯功能）。畫面上的中文名詞對照：

| 中文 | 英文/程式碼 |
|---|---|
| 帳戶 | ACCOUNTS（自己建立的 App） |
| 用戶個人資料 | User profiles（UserProfile model） |
| 身份驗證和授權 | AUTHENTICATION AND AUTHORIZATION（Django 內建） |
| 使用者 | User（Django 內建） |

若想切換成英文介面，可在 `config/settings.py` 修改：
```python
LANGUAGE_CODE = 'en-us'
```

---

## 三、操作 SOP（下次要新增一個 Django App 時，照抄流程）

```bash
# 1. 建立 App
python manage.py startapp <app名稱>

# 2. 到 config/settings.py 的 INSTALLED_APPS 加入這個 App

# 3. 在 <app名稱>/models.py 設計 model

# 4. 產生並套用 migration
python manage.py makemigrations <app名稱>
python manage.py migrate

# 5. 確認 migration 已套用
python manage.py showmigrations <app名稱>

# 6. 在 <app名稱>/admin.py 註冊 model，讓它顯示在後台
#    from django.contrib import admin
#    from .models import 模型名稱
#    admin.site.register(模型名稱)

# 7. 若還沒有管理員帳號，建立一個
python manage.py createsuperuser

# 8. 啟動伺服器，登入後台驗證
python manage.py runserver
# 瀏覽器打開 http://127.0.0.1:8000/admin/
```

---

## 四、踩坑紀錄

**問題：`python3 manage.py startapp accounts` 出現 `CommandError: 'accounts' conflicts with the name of an existing Python module`**
- 原因：先前已經用 `python manage.py startapp accounts`（沒有加 3）執行成功過一次，資料夾已經存在。因為指令執行成功時不會顯示明顯的成功訊息，誤以為沒有效果，於是重複執行了第二次。
- 解法：用 `ls` 確認 `accounts` 資料夾其實已經存在、內容完整（`models.py`、`admin.py` 等標準檔案都在），代表第一次其實已經成功，不需要重做。
- 學到的習慣：Linux 指令執行成功通常不會印出明顯的「成功」字樣，看到錯誤訊息時，先查證「這個動作是不是其實已經完成過」，不要立刻假設東西壞掉。

**問題：`createsuperuser` 設定密碼時，被要求輸入兩次密碼**
- 原因：第一次輸入的密碼太短（少於 8 碼）且全部是數字，Django 判定為弱密碼，跳出 `Bypass password validation and create user anyway? [y/N]` 詢問是否強制略過。
- 解法：選擇 `n`（不略過），重新輸入一組更強的密碼（至少 8 碼、包含英文字母與數字）。
- 學到的習慣：即使是開發階段的臨時帳號，也不要選擇略過安全性驗證，養成設定合理強度密碼的習慣。

---

## 五、下次要做的事（Sprint 1 剩餘部分：真正的 API）

目前完成的是「資料庫層面」跟「後台手動驗證」，接下來要做的是讓**外部程式**（不透過後台介面，而是用網址直接呼叫）也能使用這些功能：

1. 安裝 `djangorestframework` 與 `djangorestframework-simplejwt`
2. 實作 `POST /api/accounts/register/`
3. 實作 `POST /api/accounts/login/`（串接 simplejwt，成功後回傳 Token）
4. 實作 `GET /api/accounts/me/`
5. 用 Postman 或類似工具，逐一測試三支 API
