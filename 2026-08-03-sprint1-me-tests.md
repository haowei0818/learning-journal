# 2026-08-03 Sprint 1 收尾 — MeViewTests 撰寫紀錄

## 今天完成的事

新增獨立的 `class MeViewTests(APITestCase)`，測試「帶 access token 呼叫受保護的 `/api/accounts/me/`」。連同先前的 `RegisterTests`、`LoginTests`，`python manage.py test accounts` 全數通過（`Found 7 test(s)` / `OK`），已 push 到 GitHub。**Sprint 1（Accounts 模組）正式收尾。**

### 完整程式碼
```python
class MeViewTests(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.login_url = reverse('login')

    def test_me_success(self):
        """測試項目 4：登入後，使用 token 取得自己的資訊"""
        # 先登入拿到 token
        data = {
            "username": "testuser",
            "password": "testpass123"
        }
        response = self.client.post(self.login_url, data, format='json')
        access_token = response.data['access']

        self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {access_token}')
        me_url = reverse('me')
        response = self.client.get(me_url)

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['username'], 'testuser')
```

---

## 今天學到的核心概念

### GET vs POST，怎麼判斷該用哪個
- `views.py` 裡的 class 名稱本身就是線索：
  - `CreateAPIView`（建立新資料，如 `RegisterView`）→ 對應 `POST`
  - `RetrieveAPIView`（取回單筆既有資料，如 `MeView`）→ 對應 `GET`
- 比喻：`POST` 是「我要交一包東西給你」（帳密、申請表）；`GET` 是「我什麼都不用交，只是想跟你要一份資料回來看」。
- `GET` 不需要傳 `data`，因為沒有東西要交出去——伺服器是靠 token 知道「你是誰」，不是靠你在請求裡填 username。

### Header（標頭）是什麼
- 每次 HTTP 請求，除了「內容」（`data`）之外，還有一個叫 Header 的部分，可以想成「信封上寫的地址跟寄件人資訊」，是附加在信件外面的說明，不是信件本身的內容。
- Access token 就是夾帶在 Header 的 `Authorization` 欄位裡傳過去的，不是放進 `data`。

### `self.client.credentials()` 語法拆解
```python
self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {access_token}')
```
- `self.client.credentials(...)`：DRF 測試工具固定的方法，效果是「設定：接下來 `self.client` 發送的每個請求，都自動附上括號裡指定的 Header」。
- `HTTP_AUTHORIZATION=`：DRF 測試工具規定的固定參數名稱，代表「這是 Authorization 這個 Header 欄位」，照抄即可，不用自己發明名稱。
- `f'Bearer {access_token}'`：Python 的 **f-string** 語法，字串前面加 `f`，`{}` 裡的變數會被替換成實際的值。`Bearer` 是業界固定要寫的字（代表「這是一張手環/憑證」），跟 token 用空格接在一起。
- 跟之前用 curl 測試時打的 `-H "Authorization: Bearer eyJhbG..."` 本質上是同一件事，只是換成 Python 測試程式碼裡的寫法。

### 從字典裡取值：`response.data['access']`
- `login_response.data` 是一個字典，裡面裝著伺服器回傳的內容（例如 `{"access": "...", "refresh": "..."}`）。
- `['access']` 是 Python 從字典裡取值的標準寫法：`字典名['key名稱']`。

### Access Token vs Refresh Token 的複習應用
- `test_login_success`（前一階段）測的是「怎麼拿到手環」。
- `test_me_success`（這次）測的是「拿到手環之後，能不能成功用它去玩設施」——多了「先登入拿 token → 設定進 Header → 再呼叫 API」這個完整流程。

---

## 這次踩過的坑（除錯練習紀錄）

### 1. Class 巢狀縮排錯誤
一開始不小心把 `class MeViewTests(APITestCase):` 寫在 `LoginTests` 裡面（縮排多了 4 個空格），變成「class 裡面包 class」，而且底下沒接任何內容，會直接讓 Python 報錯。
- **修正方式**：把 `class MeViewTests` 的縮排拿掉，讓它跟 `class LoginTests` 一樣完全靠左，兩者是平行、獨立的 class，不是互相包裹。
- **學到的原則**：每個獨立的 `class` 定義，都必須完全靠左對齊（除非本來就故意要巢狀，但測試 class 不會這樣做）。

### 2. 獨立 class 需要有自己的 `setUp()`
一開始把 `test_me_success` 直接寫在 `LoginTests` 底下，沒問題是因為當時共用了 `LoginTests` 的 `setUp()`（裡面有 `self.login_url`）。但拆成獨立的 `MeViewTests` 後，這個 class 是全新、獨立的「箱子」，**不會自動繼承或借用其他 class 的 `setUp()`**，必須自己重新寫一次（建立使用者 + 準備 `login_url`），內容可以照抄，但一定要各自擁有。

### 3. 拼字筆誤
`test_me_surccess` 打成 `surccess`（多一個 `r` 少排列）。Python 不會檢查方法名稱的英文拼字對不對，所以這種筆誤**不會報錯**，但會讓測試報告、程式碼可讀性變差，值得養成寫完檢查一次名稱的習慣。

### 4. 終端機指令打錯
打 `~CD /home/loveg/...` 想切換資料夾，結果整個指令是亂碼（`cd` 打成大寫且前面多了 `~`），導致 `command not found`。
- **正確寫法**：`cd` 全小寫，路徑前不要多加符號。可以用完整路徑 `cd /home/loveg/projects/ai_document_platform`，或用 `~` 代表 home 目錄的簡寫 `cd ~/projects/ai_document_platform`。

---

## 目前還不太確定、需要之後找機會再釐清的地方

1. `assertEqual` 底層「丟出錯誤後，測試框架怎麼接住它」這段機制，上次提過還不算篤定，這次沒有再深入練習，之後可以找機會用更多故意失敗的實驗來加深印象。
2. `self.client.credentials()` 設定之後，如果同一個測試方法裡**還有下一次請求**，token 會不會一直留著繼續被夾帶？（這次沒有實際測試過這個情境，是潛在的疑問點，之後寫 Documents App 測試如果有類似情境可以順便驗證。）

---

## Sprint 1 完整成果（Accounts 模組收尾）

- `RegisterTests` × 3：成功註冊、重複帳號、密碼空白
- `LoginTests` × 3：成功登入、密碼錯誤、帳號不存在（含 401 vs 400 分類、username enumeration 資安概念）
- `MeViewTests` × 1：帶 token 取得自己的資訊
- 共 7 個測試案例，全數通過，已 push 到 GitHub（`ai-document-platform`）

---

## 下一步

- 更新履歷用的專案描述（如有需要）
- 進入 **Sprint 2：文件管理模組（Documents App）**
