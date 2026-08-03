# 2026-08-03 Sprint 1 — LoginTests 撰寫紀錄

## 今天完成的事

在 `accounts/tests.py` 新增 `LoginTests`，涵蓋三個測試案例，`python manage.py test accounts` 全數通過（`Found 6 test(s)` / `OK`），已 push 到 GitHub（`ai-document-platform`）。

### 1. `setUp()`
```python
class LoginTests(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.login_url = reverse('login')
```
- 用 `create_user()` 而不是 `create()`，因為 `create_user` 會自動把密碼加密（hash），`create` 不會，會導致密碼變明文、登入永遠失敗。
- `setUp()` 會在**每一個** `test_xxx` 方法執行前自動先跑一次，避免三個測試各自重複寫「建立使用者」的程式碼。

### 2. `test_login_success`（帳密正確 → 應拿到 token）
```python
def test_login_success(self):
    data = {"username": "testuser", "password": "testpass123"}
    response = self.client.post(self.login_url, data, format='json')
    self.assertEqual(response.status_code, status.HTTP_200_OK)
    self.assertIn("access", response.data)
    self.assertIn("refresh", response.data)
```
- 登入成功用 `200`，不是 `201`（`201` 專指「新建立了一筆資料」，登入沒有新建資料，只是核發 token）。
- `assertIn("access", response.data)`：檢查 response 裡有沒有出現 `"access"` 這個 key。

### 3. `test_login_wrong_password`（帳號對、密碼錯 → 401）
```python
def test_login_wrong_password(self):
    data = {"username": "testuser", "password": "wrongpass111"}
    response = self.client.post(self.login_url, data, format='json')
    self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

### 4. `test_login_nonexistent_user`（帳號不存在 → 也是 401，不是 404）
```python
def test_login_nonexistent_user(self):
    data = {"username": "nonexistentuser", "password": "somepass123"}
    response = self.client.post(self.login_url, data, format='json')
    self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

---

## 今天學到的核心概念

### 400 vs 401 vs 403（郵戳比喻）
- **400 Bad Request**：客戶端送來的資料本身格式就不完整（例如密碼欄位空白）。
- **401 Unauthorized**：資料格式沒問題，但身份驗證不通過（密碼錯誤、帳號不存在）。
- **403 Forbidden**：已經確認你是誰了，但你沒有權限做這件事。

### `assertEqual` 的運作原理
底層邏輯等同於：
```python
def assertEqual(self, 實際值, 期待值):
    if 實際值 == 期待值:
        pass  # 安靜通過，測試繼續往下跑
    else:
        raise AssertionError(f"{實際值} != {期待值}")  # 丟出錯誤，這個測試被標記 FAILED
```
- `AssertionError: 401 != 200` 這句話的意思：左邊 401 是**實際**拿到的值，右邊 200 是**期待**的值，兩者不相等所以失敗。
- 一個測試方法失敗，不會讓整個測試框架當機，其他測試方法照樣會繼續跑（用接力賽裁判吹哨比喻：這一棒不算，但比賽繼續）。
- 親自動手做過「故意寫錯期待值、看紅字報錯」的實驗，實際看過 `.F...` 這種點與 F 混合的輸出，理解「一個點 = 一個測試通過，一個 F = 一個測試失敗」。

### Access Token vs Refresh Token（遊樂園手環比喻）
- **Access Token**：像入場手環，證明「我是已登入的使用者」，用來呼叫受保護的 API（如 `/api/accounts/me/`），有效期較短。
- **Refresh Token**：像換手環的憑證卡，不直接拿去呼叫一般 API，只在 access token 過期時，用來換一張新的 access token，效期較長。

### Username Enumeration（帳號枚舉）資安概念
- 如果系統誠實區分「帳號不存在」（如回傳 404）跟「密碼錯誤」（如回傳 401），攻擊者可以靠亂猜帳號、觀察回傳的錯誤類型，反推出「這個帳號真實存在」。
- 正確作法：不論是帳號不存在還是密碼錯誤，一律回傳同樣的 `401`，讓外部無法分辨，這也是為什麼銀行 ATM 也只會說「帳號或密碼錯誤」而不會明講是哪一項錯。

### Git 相關
- 用 `git diff <檔案>` 可以看到某個檔案「實際被改了什麼」，`-` 開頭是舊版被刪除的內容、`+` 開頭是新增的內容，用來在 `git add` 前確認改動是不是乾淨、預期內的。

---

## 目前還不太確定、需要之後找機會再釐清的地方

1. **`assertEqual` 底層原理雖然講過一次，但還沒有很篤定**——尤其是「丟出錯誤（raise）之後，測試框架怎麼接住它、決定要不要繼續跑下一個測試」這一段，感覺理解得比較模糊，之後可以用更多小實驗（例如故意讓不同種類的 assert 失敗）來加深印象。
2. **VS Code 自動完成（IntelliSense）跳出來的程式碼**，容易在還沒真正想清楚的情況下就被帶著走、直接採用——需要練習「先自己想過一次邏輯，再讓自動完成幫忙打字」，而不是反過來。
3. 尚未實作、下一步要學的新技巧：**測試裡怎麼「帶著 token」呼叫受保護的 API**（`MeTests` 會用到），包括要先登入拿到 access token、再把它放進 request 的 header 裡一起送出去的具體寫法，這部分還沒開始。

---

## 下一步

- 撰寫 `MeTests`（帶 token 呼叫 `/api/accounts/me/`）
- 完成後 Sprint 1 正式收尾
- 進入 Sprint 2：文件管理模組（Documents App）
