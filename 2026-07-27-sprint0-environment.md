# Sprint 0：開發環境建置

日期：2026-07-27
專案：AI 文件管理平台
本次目標：建立可長期使用、與企業實務一致的開發環境（不寫程式碼，先蓋地基）

---

## 一、今天做了什麼（總覽）

沒有寫任何一行專案程式碼，而是先建立「開發用的地基」，確保之後開發、測試、部署時環境是一致的，不會出現「我電腦可以跑，別人電腦/伺服器不能跑」的狀況。

完成項目：
- [x] 安裝並確認 WSL2 + Ubuntu
- [x] 安裝並驗證 Docker Desktop
- [x] 建立專案資料夾（放在 Linux 檔案系統內，不是 `/mnt/c/`）
- [x] VS Code 透過 Remote-WSL 連進 Linux 環境開發
- [x] 建立並啟動 Python 虛擬環境（venv）

---

## 二、概念筆記（面試可直接用這層回答「為什麼」）

### WSL2（Windows Subsystem for Linux 2）
- **是什麼**：讓 Windows 電腦裡，同時運行一個真正的 Linux 系統。
- **為什麼需要**：企業伺服器幾乎都是 Linux，不是 Windows。如果開發時用 Windows、部署時用 Linux，容易出現環境不一致的問題（例如路徑寫法不同：`C:\Users\...` vs `/home/user/...`），導致「我電腦可以跑，伺服器跑不起來」。
- **解決的問題**：Windows 與 Linux 作業系統本身的行為落差。
- **具體例子**：如果程式裡把路徑寫死成 `C:\Users\loveg\uploads\file.pdf`，搬到 Linux 伺服器會直接找不到路徑，因為 Linux 沒有「C槽」的概念。應該用 `os.path` 或 `pathlib` 這類工具處理路徑，而不是手動寫死。

### Docker / Docker Desktop
- **是什麼**：把一整套執行環境（程式 + 需要的服務，例如資料庫）打包成一個獨立、標準化的「容器（Container）」。
- **為什麼需要**：解決「我的電腦」跟「別人的電腦/伺服器」環境版本不一致的問題（例如 PostgreSQL 版本不同、套件版本衝突）。
- **核心觀念**：
  - **Image（映像檔）**：像設計圖／安裝包，靜態、不會變動。
  - **Container（容器）**：用 Image 建立出來、真正在運行的實體，像用設計圖蓋出來的房子。
- **今天做到哪**：只驗證 Docker 能正常運作（`docker run hello-world`），還沒有真的拿它來跑資料庫或 Django。

### VS Code + Remote-WSL
- **是什麼**：讓 VS Code 的操作介面，實際上是在「連線」操作 WSL2 裡的 Linux 環境，而不是操作 Windows 本機檔案。
- **為什麼需要**：如果程式碼放在 Windows 檔案系統，但執行環境是 Linux，兩邊要一直互相「翻譯」讀取檔案，速度變慢，行為也可能不一致。讓「寫程式的地方」跟「跑程式的地方」是同一個環境，才是原生速度、原生行為。
- **判斷指標**：VS Code 左下角出現綠色 `WSL: Ubuntu` 標示，代表連線成功。

### Python 虛擬環境（venv）
- **是什麼**：在專案資料夾裡，建立一個只屬於這個專案的 Python 套件安裝空間。
- **為什麼需要**：避免不同專案之間的套件版本互相衝突（例如專案 A 需要 Django 4.0，專案 B 需要 Django 5.0，若全部裝在系統共用空間會打架）。
- **判斷指標**：終端機提示字元最前面出現 `(venv)`。

### 這四個工具的共通邏輯
| 工具 | 解決的「不一致」問題 |
|---|---|
| WSL2 | Windows 與 Linux 作業系統本身的不一致 |
| Docker | 「我的電腦」與「別人的電腦/伺服器」環境版本不一致 |
| VS Code Remote-WSL | 「寫程式的地方」與「跑程式的地方」不一致 |
| venv | 同一台電腦上，不同專案間套件版本不一致 |

一句話總結：**今天做的所有事，核心都是「消除環境不一致」，讓開發環境從第一天就跟未來的部署環境盡量一致。**

---

## 三、操作指令 SOP（下次換電腦或重裝時，照抄即可）

```bash
# 1. 安裝 WSL2（在系統管理員權限的 cmd 裡執行）
wsl --install
# 完成後需要重新啟動電腦，重開機後會要求設定 Linux 使用者帳號與密碼

# 2. 安裝 Docker Desktop
# 前往 https://www.docker.com/products/docker-desktop/ 下載安裝
# 安裝時選擇 Per-user installation（使用 WSL2 backend）
# 安裝完成後，需要打開一次 Docker Desktop App，完成初次設定（可以 Skip 註冊帳號）

# 3. 驗證 Docker 是否正常
docker --version
docker run hello-world

# 4. 建立專案資料夾（在 Ubuntu 終端機裡執行，不要放在 /mnt/c/ 底下）
cd ~
mkdir -p projects/ai_document_platform
cd projects/ai_document_platform
pwd   # 確認路徑是 /home/使用者名稱/projects/ai_document_platform

# 5. 用 VS Code 打開（會自動安裝 VS Code Server 到 WSL2 內）
code .
# 若沒自動打開資料夾，手動點「開啟資料夾」，輸入：
# /home/使用者名稱/projects/ai_document_platform

# 6. 安裝 venv 工具（第一次使用需要，Ubuntu 有時未內建）
sudo apt update
sudo apt install python3-venv python3-pip -y

# 7. 建立並啟動虛擬環境
python3 -m venv venv
source venv/bin/activate
# 成功會看到提示字元最前面出現 (venv)
```

---

## 四、踩坑紀錄（除錯字典，面試時的真實素材）

**問題：`cd~` 顯示 `Command 'cd~' not found`**
- 原因：指令跟參數之間忘記加空格，`cd~` 被系統當成一整個未知的指令名稱。
- 解法：Linux 底下指令與參數之間一定要有空格，正確寫法是 `cd ~`。

**問題：`CD ~` 顯示 `CD: command not found`**
- 原因：Linux 指令是區分大小寫（case-sensitive）的，系統裡只認得小寫 `cd`，沒有 `CD`。
- 解法：Linux 指令一律用小寫。

**問題：`wsl --version` 沒有顯示版本號，反而印出整份使用說明**
- 原因：當時 WSL 功能可能還沒真正被系統啟用，或該版本 wsl.exe 不支援這個參數。
- 解法：改用 `wsl --status` 確認狀態；最終直接執行 `wsl --install` 完成安裝與啟用。

**問題：`docker run hello-world` 失敗，錯誤訊息是連不上 `docker API`／`daemon`**
- 原因：`docker --version` 只是讀取 CLI 工具本身版本，不需要背景引擎；但 `docker run` 需要 Docker Desktop 的背景引擎（daemon）真正啟動起來才能用。當時 Docker Desktop 安裝完成，但還沒有被打開、走完初次設定畫面。
- 解法：手動打開 Docker Desktop 應用程式，完成 Welcome 畫面（可以 Skip 登入），引擎啟動後再重試指令即可成功。

**問題：`mkdir ai_document_platform` 後，`pwd` 顯示還在上一層資料夾**
- 原因：`mkdir` 只是「建立」資料夾，不會自動「進入」該資料夾，是兩個分開的動作。
- 解法：建立後要再執行 `cd ai_document_platform` 才會真正進入。之後養成每次 `cd` 完順手打一次 `pwd` 確認位置的習慣。

---

## 五、關於「換電腦」與版本控制的重要觀念

- **環境/工具類（WSL2、Docker、venv 本身）**：無法直接複製到別台電腦，換電腦要重新安裝。但因為已經理解每個步驟的原理，重裝速度會比第一次快很多。
- **專案內容（自己寫的程式碼）**：只要用 Git 做版本控制、推送到 GitHub，就能在任何一台電腦上，把環境裝好之後 `git clone` 下來接續。
- **接下來要記住的關鍵觀念**：
  - `venv` 資料夾不應該上傳到 GitHub（體積大、且可以用 `requirements.txt` 這份「食譜清單」在任何電腦上重新安裝出一樣的套件環境）。
  - 之後要建立 `.gitignore` 檔案，告訴 Git 忽略 `venv` 等不必要上傳的東西。

---

## 六、下次要做的事（Sprint 0 收尾）

1. 重新啟動虛擬環境（每次重開機/重開終端機都要做）：
   ```bash
   source venv/bin/activate
   ```
2. 安裝 Django：`pip install django`
3. 建立 Django 專案骨架
4. 第一次執行 `python manage.py runserver`，瀏覽器看到 Django 歡迎畫面（第一個里程碑）
5. 設定 Git + GitHub，開始版本控管專案內容
