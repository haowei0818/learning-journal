# 到新電腦（例如公司）接續開發 SOP

用途：任何一台新電腦，只要照這份清單走一次，就能接續 AI 文件管理平台的開發進度。

---

## 第 0 步：確認電腦權限

先確認這台電腦能不能自由安裝軟體（尤其公司電腦可能有 IT 政策限制）。

```bash
wsl --version
```

- 有顯示版本號 → 環境可能已就緒，或至少 WSL 功能存在，跳到第 1 步確認即可
- 顯示找不到指令 → 嘗試 `wsl --install`，觀察有沒有跳出「需要系統管理員權限」或被擋下的訊息
- 如果完全不能裝任何東西 → 先跟主管理人（自己）確認能不能申請權限，暫時只能做不需要跑程式的事（看筆記、規劃、寫文件）

---

## 第 1 步：安裝 WSL2（若這台電腦還沒有）

於系統管理員權限的 cmd 或 PowerShell 執行：

```bash
wsl --install
```

完成後依提示重新啟動電腦，重開機後會要求設定 Linux 使用者帳號與密碼（可以跟其他電腦設定不同帳密，這只是本機登入用，互不影響）。

---

## 第 2 步：安裝 Docker Desktop

前往 https://www.docker.com/products/docker-desktop/ 下載安裝，安裝時選擇：
- Per-user installation（使用 WSL2 backend）

安裝完成後打開一次 Docker Desktop App，完成初次設定畫面（可以 Skip 註冊帳號）。

驗證：
```bash
docker --version
docker run hello-world
```

---

## 第 3 步：安裝 Git，並設定身份

```bash
sudo apt update
sudo apt install git -y

git config --global user.name "haowei0818"
git config --global user.email "lovegjo3au@gmail.com"
```

確認：
```bash
git config --global user.name
git config --global user.email
```

---

## 第 4 步：產生這台電腦專屬的 SSH 金鑰

⚠️ 重點觀念：SSH 金鑰是「一台電腦一把」，不是把其他電腦的私鑰複製過來。每台新電腦都要重新產生一把。

```bash
ssh-keygen -t ed25519 -C "lovegjo3au@gmail.com"
```
（三個提示全部直接按 Enter 跳過，不用打檔名或密碼）

印出公鑰內容：
```bash
cat ~/.ssh/id_ed25519.pub
```

把印出來的整串文字複製起來，貼到：
GitHub → 右上角大頭貼 → Settings → SSH and GPG keys → New SSH key
- Title：建議寫「公司電腦」這類好辨認的名字（方便之後分辨這是哪一台電腦的鑰匙）
- Key：貼上剛剛複製的公鑰內容

驗證連線：
```bash
ssh -T git@github.com
```
第一次連線會問 `Are you sure you want to continue connecting (yes/no)?`，輸入 `yes`。
看到 `Hi haowei0818! You've successfully authenticated...` 代表成功。

---

## 第 5 步：把專案從 GitHub 抓下來（git clone）

`git clone` = 把 GitHub 上的 repo，完整複製一份到這台電腦。

```bash
cd ~
mkdir -p projects
cd projects

git clone git@github.com:haowei0818/ai-document-platform.git
git clone git@github.com:haowei0818/learning-journal.git
```

執行完後，`ai-document-platform` 資料夾會出現，裡面有 `.gitignore`，但**不會有 `venv` 資料夾**——因為 `.gitignore` 讓 venv 沒有被上傳，GitHub 上本來就沒有它。

---

## 第 6 步：重新建立虛擬環境（每台新電腦都要做這步）

```bash
cd ai-document-platform
sudo apt install python3-venv python3-pip -y
python3 -m venv venv
source venv/bin/activate
```

確認提示字元出現 `(venv)`，代表虛擬環境就緒。

之後如果專案裡已經有 `requirements.txt`（之後 Sprint 會產生這個檔案），還要多一步：
```bash
pip install -r requirements.txt
```
這行會依照清單，把之前裝過的套件全部重新安裝一次。

---

## 第 7 步：用 VS Code 打開，接續工作

```bash
code ~/projects
```

VS Code 左下角出現 `WSL: Ubuntu` 綠色標示，且左側同時看到 `ai-document-platform` 與 `learning-journal` 兩個資料夾，代表環境完全就緒，可以接續開發。

---

## 之後每次做完進度，記得同步回 GitHub

不管在哪台電腦，做完一段進度後：

```bash
git add .
git commit -m "說明這次改了什麼"
git push
```

換一台電腦要接續前，先執行：

```bash
git pull
```

這樣可以把其他電腦推上去的最新版本抓下來，避免兩邊進度不同步。
