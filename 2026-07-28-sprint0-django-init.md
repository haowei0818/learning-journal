# Sprint 0 收尾：Django 專案初始化

日期：2026-07-28
專案：AI 文件管理平台
本次目標：安裝 Django、建立專案骨架、第一次成功啟動伺服器，並修正版控上的常見誤區

---

## 一、今天做了什麼（總覽）

- [x] 安裝 Django，並用 `pip freeze` 產生 `requirements.txt`
- [x] 用 `django-admin startproject` 建立 Django 專案骨架
- [x] 第一次執行 `python manage.py runserver`，瀏覽器看到 Django 歡迎畫面
- [x] 發現並修正 `.gitignore` 沒設好、`__pycache__` 與 `db.sqlite3` 誤傳到 GitHub 的問題

---

## 二、概念筆記

### requirements.txt
- **是什麼**：記錄這個專案用到的所有 Python 套件與精確版本號的清單檔案。
- **為什麼需要**：別人（或自己換電腦）拿到專案後，只要執行 `pip install -r requirements.txt`，就能把環境裝到跟原本完全一樣，不用一個個手動找套件安裝。
- **產生方式**：
  ```bash
  pip freeze > requirements.txt
  ```
- **重要提醒**：一定要在「正確的資料夾」、且虛擬環境已啟動的狀態下執行，否則清單內容或存放位置會錯（今天踩過這個坑，見下方踩坑紀錄）。

### django-admin startproject
- **是什麼**：Django 提供的指令工具，自動產生一套標準專案目錄結構（設定檔、URL 路由入口、啟動腳本等），不用自己從零手刻。
- **指令**：
  ```bash
  django-admin startproject config .
  ```
  - `config`：設定檔資料夾的名稱。企業實務上常取名叫 `config` 或 `core`（中性名稱），而不是跟公司/專案同名。
  - 最後的 `.`（英文句點）：代表「就在目前這個資料夾生成，不要多包一層」。若省略這個點，Django 會多建立一層資料夾，跟 Git repo 的根目錄對不齊。

### 專案骨架裡每個檔案的用途
| 檔案 | 用途 |
|---|---|
| `settings.py` | 整個專案最重要的設定檔，資料庫連線、已安裝的 App、安全性設定都在這裡 |
| `urls.py` | 網址路由的總入口，決定「使用者訪問某個網址時，要交給哪個功能處理」 |
| `manage.py` | 之後最常用到的管理指令工具（建立 App、跑資料庫遷移、啟動伺服器都靠它） |
| `wsgi.py` / `asgi.py` | 部署到正式伺服器時，讓 Web 伺服器（例如 Nginx）跟 Django 溝通的介面，開發階段先不用管 |

### 虛擬環境是「整個終端機 session 生效」，不是「針對資料夾生效」
- 今天發生過一次誤會：以為指令下錯資料夾就會裝到錯地方，後來確認發現：`source venv/bin/activate` 啟動後，**接下來整個終端機視窗，不管切換到哪個資料夾，都還是用同一個虛擬環境**，除非重新開一個終端機或重新 cd 到別的地方手動再啟動一次。
- 但即使虛擬環境沒裝錯地方，**用 `pip freeze > requirements.txt` 產生的檔案，還是會依照「目前所在的資料夾」寫入**，所以檔案本身還是可能放錯位置，這跟套件裝在哪裡是兩件事，要分開判斷。

### .gitignore 不只是 venv，還要排除這些
今天真正踩到的坑：第一次 commit 時，忘了排除以下東西，導致被誤傳到 GitHub：

- **`__pycache__/` 與 `*.pyc`**：Python 執行時自動產生的暫存編譯檔案，不是原始碼，每台電腦、每次執行都會自動重新生成，上傳沒有意義。
- **`db.sqlite3`**：資料庫檔案本身（存實際資料）。企業實務上資料庫檔案幾乎絕對不放進 Git 版控，因為內容一直變動，且多人協作時衝突難處理。資料庫應該用「程式碼（migration 遷移檔）」重新產生結構，而不是把資料庫檔案當原始碼管理。
- **`.env`**：之後會用到的環境變數檔案，裡面會放 API 金鑰、密碼等機密資訊，絕對不能上傳，先預先排除養成習慣。

正確的 `.gitignore` 內容（Django 專案）：
```
venv/
__pycache__/
*.pyc
db.sqlite3
.env
```

### 如何移除「已經被 Git 追蹤」的檔案
只加 `.gitignore` 沒用在「已經 commit 過」的檔案上——這種情況要多一個步驟：
```bash
git rm -r --cached config/__pycache__
git rm --cached db.sqlite3
```
`--cached` 的意思是「只從 Git 的追蹤清單移除，不要真的刪除電腦上的檔案」，這樣可以安全地把誤傳的檔案從 GitHub 紀錄中清掉，同時保留本機檔案不受影響。

---

## 三、操作 SOP（下次重建 Django 專案骨架時，照抄即可）

```bash
# 1. 進入專案資料夾，啟動虛擬環境
cd ~/projects/ai_document_platform
source venv/bin/activate

# 2. 安裝 Django
pip install django

# 3. 確認版本
python3 -m django --version

# 4. 記錄套件清單
pip freeze > requirements.txt

# 5. 建立 Django 專案骨架（注意最後的點）
django-admin startproject config .

# 6. 建立正確的 .gitignore（Django 專案版本）
cat > .gitignore << 'EOF'
venv/
__pycache__/
*.pyc
db.sqlite3
.env
EOF

# 7. 第一次啟動伺服器測試
python manage.py runserver
# 瀏覽器打開 http://127.0.0.1:8000/ 確認看到 Django 歡迎畫面
# 按 Ctrl + C 停止伺服器

# 8. 推送到 GitHub
git add .
git commit -m "Sprint 0: initial Django project setup"
git push
```

---

## 四、踩坑紀錄

**問題：今天重新打開終端機後，發現自己身處 `learning-notes` 資料夾，卻在裡面執行了 `pip install django` 與 `pip freeze > requirements.txt`**
- 原因：昨天最後一次操作是在 `learning-notes` 資料夾做 Git 推送，今天重新打開終端機時，新視窗延續了「上次關閉時所在的資料夾」，而不是自動回到專案資料夾。
- 影響釐清：因為虛擬環境是「整個終端機 session 生效」，Django 其實還是正確裝進了 `ai_document_platform/venv` 裡（沒有裝錯地方）；但 `requirements.txt` 這個檔案本身，因為是依照「目前所在資料夾」寫入，所以被錯誤產生在 `learning-notes` 裡。
- 解法：
  ```bash
  rm ~/projects/learning-notes/requirements.txt   # 刪除放錯位置的檔案
  cd ~/projects/ai_document_platform               # 切到正確資料夾
  pip freeze > requirements.txt                     # 重新產生在正確位置
  ```
- 學到的習慣：**每次開始工作前，先用 `pwd` 確認自己「人在哪裡」**，不要憑印象假設。

**問題：第一次 `git push` 後，發現 `__pycache__/` 與 `db.sqlite3` 都被上傳到 GitHub 了**
- 原因：`django-admin startproject` 執行時建立專案骨架後，沒有先設定 `.gitignore` 就直接 `git add . / commit / push`，導致 Python 自動產生的暫存檔與資料庫檔案一起被記錄進去。
- 解法：
  ```bash
  cat > .gitignore << 'EOF'
  venv/
  __pycache__/
  *.pyc
  db.sqlite3
  .env
  EOF

  git rm -r --cached config/__pycache__
  git rm --cached db.sqlite3

  git add .
  git commit -m "Add gitignore, remove pycache and db from tracking"
  git push
  ```
- 學到的習慣：**建立新專案骨架後，第一件事就是先設好完整的 `.gitignore`，再做第一次 commit**，順序顛倒過來很容易漏東西。

---

## 五、下次要做的事（Sprint 1：帳號模組）

按照最初的 Roadmap，接下來要進入 **Sprint 1：帳號模組（Accounts）**——設計並實作註冊、登入的 API（DRF + JWT）。

在動手寫程式碼之前，會先走一輪設計思考：
1. User Story 對應到哪些具體的 API 端點
2. 資料庫層面：要不要擴充 Django 內建的 User model
3. 認證方式的選擇與理由（Session vs Token/JWT）
