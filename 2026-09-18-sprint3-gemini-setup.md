# Sprint 3 開始 — RAG 概念、Gemini API 選型與串接驗證

## 今天完成的事

正式開始 **Sprint 3：AI 問答功能（RAG）**，這是整個「AI 文件管理平台」的核心賣點。今天完成了整體架構的概念釐清、技術選型（Gemini API）、API Key 安全存放、套件安裝（過程中排查了一次官方套件改版），並用最小範例驗證 Key、套件、中文回應都正常運作。

---

## 設計思考：RAG 是什麼、為什麼需要它

### 核心問題：AI 模型本身不知道你上傳過的文件內容

一般的 AI 模型（例如直接呼叫 Gemini），回答問題時只能靠它「訓練時記住的知識」，完全不知道使用者資料庫裡存了什麼。比喻：一個博學但沒看過你家帳本的會計師，你問他「我上個月花多少錢」，他答不出來，因為根本沒看過資料。

**AI 模型跟 Django 專案是完全獨立、互不相通的兩個系統**，除非主動把資料餵給它，不然它永遠不會自動知道你上傳過什麼文件。

### RAG（Retrieval-Augmented Generation，檢索增強生成）的四個階段

用「圖書館問答服務」比喻整個流程：

1. **索引（Embedding / 向量化）**：文件進圖書館時，先拆成一段一段，每段都做一張「內容摘要卡」（向量），方便之後快速比對「這段在講什麼」。
2. **查詢系統（向量資料庫）**：把所有「摘要卡」存進一個擅長回答「哪些卡片內容最像目前問題」的特殊資料庫。
3. **檢索（Retrieval）**：使用者發問時，先去查詢系統裡找出「最相關的幾段內容」，不是直接把問題丟給 AI。
4. **生成（Generation）**：把「使用者的問題」+「找到的相關段落」一起丟給 AI，叫它根據這些真實資料來回答，而不是憑空回答。

**這次專案的範圍決定**：選擇「針對使用者上傳過的**所有文件**，AI 自己去所有文件裡找答案」（而不是先選定某一份文件再問），這個做法更貼近真實產品（如 Notion AI、Google Drive 搜尋），但工作量也比單一文件問答更大，之後仍會照 MVP 精神拆解實作。

---

## 技術選型：為什麼選 Google Gemini API

決定條件是「免費、好串接、支援中文」。查詢當下（2026年9月）的現況後，比較幾個主要選項：

- **Google Gemini API**：有免費方案，不需要信用卡即可申請 Key；額度對學習/作品集用量足夠；官方文件有完整繁體中文版本；Python 有官方 SDK，串接容易。
- **其他選項（如 Groq）**：也有免費方案、速度快，但主要跑開源模型（如 Llama），中文表現普遍不如 Gemini 穩定，對「文件問答」這種需要精準理解的任務評價稍弱。

**結論**：選用 Google Gemini API。

Sources:
- [Gemini API Free Tier 2026: 1,500 Req/Day, 1M TPM — No Card](https://tokenmix.ai/blog/gemini-api-free-tier-limits)
- [Is Google Gemini API Free? Free Tier Limits & Upgrade Triggers (2026)](https://costbench.com/software/llm-api-providers/google-gemini-api/free-plan/)
- [Google Gemini API免费层限制完全指南（2026年1月更新）](https://yingtu.ai/zh/blog/google-gemini-api-free-tier-limits-2026)
- [Best Free AI APIs in 2026: Real Limits Compared](https://freeapihub.com/blog/best-free-ai-apis-developers)

---

## 核心概念與操作整理

### 1. API Key 的安全存放方式（沿用 `.env` 的固定模式）

```bash
cat > .env << 'EOF'
GEMINI_API_KEY=你的Key
EOF
```

- `.env` 這個檔名，專案的 `.gitignore` 之前就已經排除過，這次直接沿用，不用再改設定。
- **驗證檔案有內容但不洩漏內容**：用 `wc -l .env`（顯示行數，不顯示內容），不要用 `cat .env`（會把 Key 整個印出來貼進對話裡）。
- **驗證 `.gitignore` 真的有排除**：`git status` 之後，`.env` 不應該出現在任何清單（tracked 或 untracked）裡，這才代表 Git 完全忽略它。

### 2. `python-dotenv`：讓 Python 讀取 `.env` 裡的內容

```python
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.environ.get('GEMINI_API_KEY')
```

- `load_dotenv()`：讀取 `.env` 檔案，把裡面的內容載入成系統的「環境變數」。
- `os.environ.get('GEMINI_API_KEY')`：從環境變數裡把指定的值取出來，用變數名稱去對應 `.env` 裡設定的那行。
- 驗證時只印 `api_key is not None`（`True`/`False`），不要把 Key 本身印出來。

### 3. 意外踩坑：Google 官方套件已經改版棄用

一開始照常理裝了 `google-generativeai`，但查證後發現這個套件**已被 Google 官方標記為 deprecated（棄用）**，改用新的統一套件 **`google-genai`**。

```bash
pip uninstall google-generativeai -y
pip install google-genai python-dotenv
```

**這次學到的教訓**：AI 領域的套件、模型版本變化非常快，過去學過的知識（甚至是 Claude 自己記得的資訊）可能已經過時，遇到「安裝哪個套件」「用哪個模型名稱」這類容易變動的資訊時，最好的做法是**先查證目前最新現況，再動手**，而不是完全憑舊記憶操作。

Sources:
- [GitHub - google-gemini/deprecated-generative-ai-python: This SDK is now deprecated, use the new unified Google GenAI SDK](https://github.com/google-gemini/deprecated-generative-ai-python)
- [Gemini API libraries | Google AI for Developers](https://ai.google.dev/gemini-api/docs/libraries)

### 4. 最小驗證：確認 Key、套件、中文回應都正常

```python
from google import genai

client = genai.Client(api_key=api_key)
response = client.models.generate_content(
    model='gemini-3.6-flash',
    contents='請用一句話跟我打招呼'
)
print(response.text)
```

- `genai.Client(api_key=api_key)`：用 Key 建立一個跟 Gemini 溝通的「客戶端」，概念上類似 curl 呼叫自己 API 時的那個連線角色。
- `client.models.generate_content(...)`：真正呼叫 Gemini，`model` 指定用哪個模型、`contents` 是要問的問題。

**中途又踩了一個小坑**：一開始用的模型名稱 `gemini-2.5-flash` 已經不開放給新使用者，API 直接在錯誤訊息裡回覆正確的替代模型名稱 `gemini-3.6-flash`，改了之後成功拿到中文回應：

```
你好！很高興遇見你，今天有什麼我可以為你服務的嗎？
```

**這次的判斷方式值得記住**：遇到 API 回傳的錯誤訊息時，很多時候（尤其是官方服務）錯誤訊息本身就會**直接告訴你怎麼修正**（像這次直接寫出該換成哪個模型名稱），先仔細讀錯誤內容，往往比自己亂猜或到處找資料更快。

---

## 還不確定、需要之後多練習的地方

1. **RAG 架構裡「向量化」跟「向量資料庫」實際該怎麼選、怎麼串接**：目前只停留在概念理解（比喻成「摘要卡」跟「查詢系統」），還沒有實際技術選型跟動手做，這是下一步要處理的部分。
2. **AI 相關套件/模型改版速度很快，之後查資料要留意時效性**：這次連續踩到兩個「舊知識已過時」的坑（套件改名、模型下架），之後遇到 AI 相關的技術細節，需要養成「先確認是不是最新資訊」的習慣，不能完全依賴舊的印象。
3. **`client.models.generate_content` 的其他參數**：這次只用了最基本的 `model` 和 `contents`，實際串進 Django 專案時，可能還需要了解如何控制回答長度、溫度（隨機性）等參數，這部分還沒碰過。

---

## 下一步

- 設計 RAG 整體架構的技術選型：向量化要用哪個 embedding 模型（可能還是用 Gemini 提供的 embedding API）、向量資料庫要用什麼工具（例如輕量級、免費、容易在本機/雲端跑的方案）
- 確認選型後，開始設計文件「切段（chunking）」與「向量化」的實作流程
- 之後才會進到「檢索 + 生成」的問答 API 設計與實作
