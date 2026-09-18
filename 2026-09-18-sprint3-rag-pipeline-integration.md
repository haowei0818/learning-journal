# Sprint 3 續 — RAG 管線整合進 Django：持久化、切段、文件讀取、端對端串接

## 今天完成的事

延續 Sprint 3 稍早的 embedding/Chroma 驗證，今天把所有零散驗證過的技術，**真正整合進 Django 專案**：Chroma 改成持久化模式、寫出文件切段函式、寫出多格式文件讀取函式、串接進上傳流程，並完成一次完整的端對端驗證（上傳文件 → 自動切段向量化 → 存入 Chroma → 用完全不同字面的問句成功檢索回正確內容）。過程中也意外發現並修正了一個 Sprint 2 遺留的未提交程式碼問題。

---

## 核心概念與操作整理

### 1. Chroma 持久化模式：`Client()` vs `PersistentClient(path=...)`

```python
# 暫存模式（今天稍早測試用，程式結束資料就消失）
chroma_client = chromadb.Client()

# 持久化模式（正式使用）
chroma_client = chromadb.PersistentClient(path="./chroma_db")
collection = chroma_client.get_or_create_collection(name="documents")
```

- 概念跟 `db.sqlite3` 完全一樣：把資料存成硬碟上的檔案（實際上 Chroma 內部本身也是用 SQLite 存索引，`chroma_db/` 資料夾裡真的有一個 `chroma.sqlite3`）。
- `get_or_create_collection`（不是 `create_collection`）：「如果集合已存在就直接拿來用，不存在才建立新的」——避免程式重跑第二次時，因為「集合已存在」而報錯。
- **驗證方式**：故意開兩個完全獨立的 Python shell，第一個只負責存資料、第二個完全不重新存、只做查詢，結果依然查得到——證明資料是真的寫進硬碟，不是只存在記憶體。
- `chroma_db/` 是執行期產生的資料，比照 `db.sqlite3`、`media/` 的處理方式加進 `.gitignore`。

### 2. 文件切段（Chunking）：為什麼需要、怎麼做

**為什麼需要**：如果把一份很長的文件（例如 50 頁）整份丟去做一次向量化，向量化出來的「一個向量」要同時代表所有不同主題的內容，會像把十幾種顏色的顏料混在一起攪拌——變成「什麼都有一點、但什麼都不精準」的模糊平均值。另外 embedding 模型對單次能處理的文字長度也有上限。

**解法：切成小段，每段各自做向量化**，並讓相鄰段落之間刻意保留一小段「重疊」內容，避免語意剛好被切在句子中間、變得破碎難懂（比喻：影片剪輯鏡頭銜接處保留重疊幾秒，避免硬切造成突兀）。

```python
def chunk_text(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start += chunk_size - overlap
    return chunks
```

- `chunk_size=500`、`overlap=50`：業界常見的經驗法則量級（中文抓每段幾百字），沒有絕對正確答案，之後測試效果不好可再調整。
- 關鍵是最後一行 `start += chunk_size - overlap`：下一段的起點不是從 `end` 開始（那樣沒有重疊），而是往回退 `overlap` 個字，讓兩段刻意重疊。
- **驗證方式**：故意把 `chunk_size` 調小成 100，測試一份包含完整句子的文字，實際印出每一段，親眼確認前後兩段確實共用了一段重複文字，且重疊處銜接了完整語意。

### 3. 多格式文件讀取：`.txt` / `.pdf` / `.docx`

先鎖定三種最常見格式（MVP 原則，圖片/掃描檔的 OCR 屬於進階功能，先不處理）。

```python
def extract_text_from_file(file_path):
    if file_path.endswith('.txt'):
        with open(file_path, 'r', encoding='utf-8') as f:
            return f.read()
    elif file_path.endswith('.pdf'):
        from pypdf import PdfReader
        reader = PdfReader(file_path)
        text = ''
        for page in reader.pages:
            text += page.extract_text() or ''
        return text
    elif file_path.endswith('.docx'):
        from docx import Document as DocxDocument
        doc = DocxDocument(file_path)
        text = '\n'.join([para.text for para in doc.paragraphs])
        return text
    else:
        raise ValueError(f"不支援的檔案格式: {file_path}")
```

- 依副檔名判斷、分別對應到不同讀取方式，統一回傳純文字。
- `.pdf` 用 `pypdf`（**不是** `PyPDF2`——這個舊套件已不再維護，`pypdf` 是接手的正式套件，查證後確認）。
- `.docx` 用 `python-docx`：**注意 import 別名** `from docx import Document as DocxDocument`——因為專案自己的 `Document` model 也叫 `Document`，不改名會互相衝突。
- `.pdf`/`.docx` 因為套件成熟穩定，這次選擇「先信任、之後實際遇到再驗證」，沒有另外花時間建立測試檔案，只完整驗證了 `.txt` 這條路徑。

Sources:
- [Python PDF library comparison (2026)](https://www.nutrient.io/blog/best-python-pdf-libraries/)
- [pypdf vs pdfplumber — PDF Parsing & Text Extraction](https://theneuralbase.com/compare/pypdf-vs-pdfplumber/)

### 4. RAG 安全原則：延續 Sprint 2「只能碰自己的東西」

**設計討論的關鍵問題**：如果 A 使用者上傳了一份薪資單，B 使用者問 AI「我的薪水多少」，系統應不應該讓 B 檢索到 A 的文件？答案是**絕對不行**——這跟 Sprint 2 的核心安全原則是同一件事，只是這次要在向量資料庫層面實現。

**做法**：每一段向量存進 Chroma 時，附加 `metadata`（中繼資料），標記這段內容屬於誰：

```python
collection.add(
    ids=[f"doc{document.id}_chunk{i}"],
    embeddings=[result.embeddings[0].values],
    documents=[chunk],
    metadatas=[{
        'user_id': document.user.id,
        'document_id': document.id,
        'title': document.title,
    }]
)
```

之後查詢時要用這個 `user_id` 做篩選（概念上跟 `Document.objects.filter(user=self.request.user)` 是同一件事，只是換了資料庫做法），**這部分還沒實作**，是下一步問答 API 要處理的重點。

**今天驗證時意外印證了這個必要性**：查詢時故意先不加篩選，結果除了正確配對到剛上傳的文件內容，還混進一筆「沒有 metadata」的舊測試資料（今天更早測試持久化功能時，存進去時沒有附加 `metadatas`）。這證明了「不篩選」會讓不相關、甚至沒有主人的資料混進結果，之後正式的問答 API 一定要加上篩選條件。

### 5. 把 RAG 管線串進 `DocumentUploadView`

```python
def process_document_for_rag(document):
    client = get_gemini_client()
    collection = get_chroma_collection()

    text = extract_text_from_file(document.file.path)
    chunks = chunk_text(text)

    for i, chunk in enumerate(chunks):
        result = client.models.embed_content(
            model='gemini-embedding-001',
            contents=chunk
        )
        collection.add(
            ids=[f"doc{document.id}_chunk{i}"],
            embeddings=[result.embeddings[0].values],
            documents=[chunk],
            metadatas=[{
                'user_id': document.user.id,
                'document_id': document.id,
                'title': document.title,
            }]
        )
```

```python
# documents/views.py
class DocumentUploadView(generics.CreateAPIView):
    serializer_class = DocumentSerializer
    permission_classes = [IsAuthenticated]

    def perform_create(self, serializer):
        instance = serializer.save(user=self.request.user)
        process_document_for_rag(instance)
```

**關鍵改動**：原本 `serializer.save(...)` 沒有接收回傳值，這次改成 `instance = serializer.save(...)`——`.save()` 會回傳剛存好的物件（已經有 `id`），拿到 `instance` 後立刻呼叫 `process_document_for_rag(instance)`，觸發整條「讀取 → 切段 → 向量化 → 存進 Chroma」的流程。

**端對端驗證成功**：用 curl 上傳一份請假規定文件，Django 自動處理完整條管線；接著用「請假要提前幾天申請?」這句完全不同字面的問題去查詢 Chroma，成功找到正確段落，且 `metadata` 正確帶著 `document_id`、`title`、`user_id`。

---

## 過程中排查過的問題

### 1. `cat` heredoc 指令貼到一半，內容跑進檔案本身

跟 Sprint 2 遇到過的情況類似：貼一段 `cat > 檔案 << 'EOF' ... EOF` 指令時，如果貼的範圍沒有完整框住整段（漏了開頭或結尾），bash 會把部分內容誤判成「立即要執行的指令」而報錯（`syntax error near unexpected token`），或者把整段指令本身寫進目標檔案裡。**處理方式**：先用 `cat 檔案` 驗證實際內容，如果錯誤就用 `cat >`（覆蓋，不是 `>>` 追加）重新完整貼一次，且務必提醒自己「整段一次複製，不要分次貼」。這次甚至一度貼壞造成終端機卡在多行輸入的 `>` 提示字元裡，靠 `Ctrl + C` 中斷恢復正常。

### 2. 發現 Sprint 2 遺留的未提交改動：`upload_to` 被改回舊值

驗證上傳結果時，發現回傳的檔案路徑是 `media/documents/...`，跟 Sprint 2 記錄的正確設定 `media/uploads/...` 不一致。追查後發現：

```bash
git show c5c82e2 -- documents/models.py   # 確認 Sprint 2 那次 commit 確實正確改成 uploads/
git status                                 # 發現 documents/models.py 有「未提交的改動」
git diff documents/models.py               # 確認本地檔案被改回了 documents/，但這個改動從未被 commit
```

**這代表 Git 的正式歷史紀錄其實一直是對的**，只是本地工作目錄的檔案在某次操作中被改回舊版本，且這個改動一直沒有被發現、沒有被提交。**修正方式**：`git restore documents/models.py`，把檔案還原成最後一次 commit 的正確版本，`python manage.py makemigrations --check` 確認沒有連帶造成 migration 不同步。

**這次學到的教訓**：`git status`/`git diff` 不只是「push 前的檢查清單」，也是隨時可以拿來確認「我現在看到的檔案內容，跟上次記錄的版本是否一致」的工具——這次如果沒有先注意到路徑異常、進而查證，這個問題可能會一直潛伏著不被發現。

---

## 還不確定、需要之後多練習的地方

1. **Chroma 查詢時的 `where` 篩選語法**：今天驗證時，查詢是「不篩選、查全部」，實際上線使用時，一定要依 `user_id` 篩選，但 Chroma 的篩選語法（`where={'user_id': ...}` 之類）今天還沒實際寫過、驗證過。
2. **`.pdf`/`.docx` 讀取還沒真的測試過**：選擇信任套件成熟度，跳過了額外的驗證步驟，之後第一次真的上傳 PDF/Word 檔時要留意有沒有意外狀況（例如某些排版複雜的 PDF 抽字效果不好）。
3. **一份文件的向量化，會呼叫多次 Gemini API（一段一次）**：文件越長，切出的段落越多，累積下來的 API 呼叫次數也越多，這部分的免費額度限制還沒仔細研究，之後如果上傳大量/超長文件，可能會撞到限制。
4. **`git status`/`git diff` 該養成更主動的檢查習慣**：今天是因為路徑異常才意外發現遺留問題，之後或許該養成「每次開始新的一段工作前，先看一眼 `git status`，確認工作目錄是乾淨的」這個習慣，而不是只在 push 前才檢查。

---

## 下一步

RAG 四階段裡，「切段」「向量化」「檢索儲存」已經整合進 Django、驗證可行。接下來：

1. 設計問答 API（例如 `POST /api/documents/ask/`）：接收使用者的問題
2. 把問題向量化，用 Chroma 查詢，**這次要加上 `where` 篩選，只搜尋目前登入使用者自己的文件**
3. 把「問題 + 檢索到的相關段落」組成 prompt，丟給 Gemini 生成答案
4. 回傳答案（可能也一併回傳「這是根據哪份文件回答的」，增加可信度）
5. 之後補上自動化測試
