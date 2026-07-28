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
- [x] 實作真正的 API：`register` / `login` / `me`
- [x] 用 curl 完整驗證三支 API 的認證流程
- [x] 補上 register 的自動化測試（`APITestCase`）

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

### Register / Login / Me API 實作
```python
# accounts/views.py
from rest_framework import generics
from django.contrib.auth.models import User
from .serializers import RegisterSerializer, UserSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class RegisterView(generics.CreateAPIView):
    queryset = User.objects.all()
    serializer_class = RegisterSerializer

class MeView(generics.RetrieveAPIView):
    serializer_class = UserSerializer

    def get_object(self):
        return self.request.user
```
- `MeView` 不用自己寫權限判斷邏輯，`self.request.user` 是 DRF 搭配 `JWTAuthentication` 自動從 `Authorization: Bearer <token>` 解析出來的使用者
- `login` 直接沿用 `simplejwt` 內建的 `TokenObtainPairView`，不需要自己寫

```python
# accounts/serializers.py
class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ('username', 'email', 'password')

    def create(self, validated_data):
        return User.objects.create_user(
            username=validated_data['username'],
            email=validated_data.get('email', ''),
            password=validated_data['password']
        )
```
- `write_only=True`：密碼只能被填入，絕對不會出現在任何 API 回傳內容裡
- `User.objects.create_user()`：Django 內建方法，會自動把密碼加密後才存進資料庫，不用自己手刻雜湊邏輯

### 路由設計：兩層疊加
```python
# config/urls.py（主路由，決定前綴）
path('api/accounts/', include('accounts.urls'))

# accounts/urls.py（app 內路由，決定後半段）
path('register/', RegisterView.as_view(), name='register'),
path('login/', TokenObtainPairView.as_view(), name='login'),
path('me/', MeView.as_view(), name='me'),
```
兩者疊加才是完整路徑：`/api/accounts/register/`、`/api/accounts/login/`、`/api/accounts/me/`。

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

**問題：在同一個終端機貼上 curl 指令，出現一堆 `command not found`，且顯示 `Could not connect to server`**
- 原因：該終端機正被 `python manage.py runserver` 佔用（前景執行），貼上的 curl 指令被當成 server 的輸入吃掉；後來按 `Ctrl+C` 關閉 server 後，之前堆積的字才被 shell 逐行拆開執行，這時 server 已經關閉，所以連線失敗。
- 解法：另開一個獨立的終端機分頁下 curl 指令，讓 server 那個分頁保持安靜運行。
- 學到的概念：**跑 server 用一個分頁、下指令用另一個分頁**，兩者永遠分開。

**問題：呼叫 `me` API 回傳 `bad_authorization_header` / `token_not_valid`**
- 原因：`Authorization` header 格式錯誤（例如貼到範例裡的佔位文字），或使用了舊的、已過期的 access token（JWT 每次簽發內容都不同、且有時效性）。
- 解法：確保每次呼叫 `me` 前，都用「這次」重新 login 拿到的最新 token；更好的做法是用 shell 變數自動接住 token，避免手動複製貼上出錯：
  ```bash
  ACCESS_TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/accounts/login/ \
    -H "Content-Type: application/json" \
    -d '{"username": "testuser01", "password": "StrongPass123!"}' \
    | python3 -c "import sys, json; print(json.load(sys.stdin)['access'])")

  curl -X GET http://127.0.0.1:8000/api/accounts/me/ \
    -H "Authorization: Bearer $ACCESS_TOKEN"
  ```

**問題：終端機貼上 Python class 程式碼，出現大量 `command not found` 與 `syntax error`**
- 原因：把 `.py` 程式碼直接貼進 bash 終端機執行，bash 把 `import`、`class` 等 Python 語法逐字誤判為 shell 指令。
- 解法：Python 程式碼一律寫進編輯器（`tests.py`）存檔，只有「執行指令」才下在終端機。

**問題：`python manage.py test accounts` 報錯 `ModuleNotFoundError: No module named 'rest_framework'`**
- 原因（排查過程）：
  1. 先確認終端機提示字元有沒有 `(venv)` → 一開始沒有，代表沒啟用虛擬環境
  2. `source venv/bin/activate` 後 `(venv)` 出現了，但錯誤依然存在
  3. 用 `pip list` 對照 `requirements.txt`，發現這個 venv 裡實際上根本沒裝 `djangorestframework`，即使檔案裡有列出來
- 解法：
  ```bash
  source venv/bin/activate
  pip install -r requirements.txt
  ```
- 學到的概念：**`requirements.txt` 是「聲明」，不是「事實」**，環境跟檔案要靠 `pip install -r requirements.txt` 主動同步，不能只看檔案內容就假設環境沒問題。

**問題：以為自己在同一個專案資料夾底下，`git status` 卻一直顯示 `nothing to commit`**
- 原因：`ai_document_platform` 專案裡有一個叫 `learning-notes` 的子資料夾，但真正連結到 GitHub `learning-journal` repo 的，是 `~/projects/` 底下**另一個獨立**的 `learning-notes` 資料夾，兩者同名但完全是不同路徑，容易搞混。
- 解法：用 `git remote -v` 確認目前所在資料夾實際連結到哪個 GitHub repo，而不是只看資料夾名稱判斷。
- 學到的概念：**同名資料夾容易造成誤會，push 前先用 `git remote -v` 確認自己在正確的 repo 裡**。

---

## 五、Register API 手動測試流程

### 判斷驗證機制：從 import 看出用了什麼套件
```python
from rest_framework_simplejwt.views import TokenObtainPairView
```
看到這行就能確定這個專案用的是 JWT（SimpleJWT），而不是 Session Authentication。

### 三支 API 的完整測試流程（curl）
```bash
# 1. 開一個終端機分頁，啟動虛擬環境並跑 server
cd ~/projects/ai_document_platform
source venv/bin/activate
python manage.py runserver
# 這個分頁接下來不要再輸入任何指令

# 2. 另開一個新的終端機分頁，一樣先啟動 venv
cd ~/projects/ai_document_platform
source venv/bin/activate

# 3. 測試 register
curl -X POST http://127.0.0.1:8000/api/accounts/register/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser01", "email": "test01@example.com", "password": "StrongPass123!"}'

# 4. 登入並自動取得 token，直接呼叫 me
ACCESS_TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/accounts/login/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser01", "password": "StrongPass123!"}' \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['access'])")

curl -X GET http://127.0.0.1:8000/api/accounts/me/ \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### 測試結果
- register：`201 Created`，回傳 `{"username": "testuser01", "email": "test01@example.com"}`（沒有 `password`，因為 `write_only=True`；也沒有 `id`，因為 `RegisterSerializer` 的 `fields` 沒有列出來，屬於設計上的必然結果）
- login：`200 OK`，回傳 `access` + `refresh` 兩組 token
- me：`200 OK`，回傳 `{"id": 4, "username": "testuser01", "email": "test01@example.com"}`（`id` 不是從 1 開始，是因為之前測試失敗留下的髒資料累積過 user 記錄——這也是為什麼自動化測試要用獨立的測試資料庫，避免汙染本機開發用的 `db.sqlite3`）

---

## 六、自動化測試（DRF APITestCase）

### 核心概念
```python
from django.contrib.auth.models import User
from rest_framework.test import APITestCase
from rest_framework import status
from django.urls import reverse

class RegisterTests(APITestCase):
    def setUp(self):
        self.register_url = reverse('register')

    def test_register_success(self):
        data = {"username": "newuser01", "email": "newuser01@example.com", "password": "StrongPass123!"}
        response = self.client.post(self.register_url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertTrue(User.objects.filter(username="newuser01").exists())

    def test_register_duplicate_username(self):
        User.objects.create_user(username="existuser", password="Pass123!")
        data = {"username": "existuser", "email": "another@example.com", "password": "AnotherPass123!"}
        response = self.client.post(self.register_url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)

    def test_register_password_blank(self):
        data = {"username": "newuser02", "email": "newuser02@example.com", "password": ""}
        response = self.client.post(self.register_url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```
- `self.client`：DRF 內建的模擬 HTTP client，不會真的啟動 `runserver`，速度快
- `reverse('register')`：用 `urls.py` 裡設定的 `name=` 反查網址，路徑改了測試不用跟著改
- `setUp()`：每個 `test_` 方法執行前都會重新跑一次，確保測試互不干擾
- 執行測試時，Django 會自動建立一個全新、獨立的**測試資料庫**，跑完自動銷毀，不會汙染本機的 `db.sqlite3`
- `CharField` 的 `allow_blank` 參數預設是 `False`，代表空字串 `""` 會被 validation 擋下、回傳 `400`，不需要額外自己寫檢查邏輯

### 執行結果
```bash
python manage.py test accounts
```
```
Found 3 test(s).
Creating test database for alias 'default'...
...
Ran 3 tests in 1.181s

OK
Destroying test database for alias 'default'...
```
3 個測試全數通過，並已推送上 GitHub（`ai-document-platform` repo）。

### 讀 Traceback 的正確順序：先看最後一行
除錯過程中學到：Python 錯誤堆疊很長，但**真正的錯誤永遠在最後一行**，上面一長串是「呼叫鏈」，通常不用細看。

---

## 七、下次要做的事

Sprint 1 主體功能已完成並驗證，`accounts` App 的自動化測試還沒補完：

1. **`LoginTests`**：`test_login_success`、`test_login_wrong_password`、`test_login_nonexistent_user`
2. **`MeTests`**：需要學會在 `APITestCase` 裡模擬「先登入拿 token，再帶 token 呼叫受保護的 API」這個技巧
3. 完成後，Sprint 1 的核心認證流程就有完整的自動化測試覆蓋，可以正式收尾，準備進入下一個 Sprint（文件上傳/管理模組）
