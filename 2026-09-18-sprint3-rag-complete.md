# Sprint 3 完整記錄 — AI 問答功能(RAG):技術選型、實作、除錯、自動化測試

> 本檔案合併自 Sprint 3 期間的 5 份零散筆記(`gemini-setup`、`embedding-chroma-verified`、`rag-pipeline-integration`、`ask-api-e2e-test`、`ask-api-automated-tests`),整理成一份完整版本,原始 5 份筆記可從 git 歷史查回。

## Sprint 3 目標

整個「AI 文件管理平台」的核心賣點:讓使用者對自己上傳過的所有文件,用自然語言提問,AI 根據真正的文件內容回答(RAG,Retrieval-Augmented Generation,檢索增強生成),而不是憑空回答。

---

## 一、設計思考:RAG 是什麼、為什麼需要它

**核心問題**:AI 模型本身不知道使用者上傳過的文件內容。一般直接呼叫 Gemini,回答問題只能靠訓練時記住的知識,完全不知道資料庫裡存了什麼——比喻:一個博學但沒看過你家帳本的會計師,你問他「我上個月花多少錢」,他答不出來。**AI 模型跟 Django 專案是完全獨立、互不相通的兩個系統**,除非主動把資料餵給它,不然它永遠不會自動知道使用者上傳過什麼文件。

**RAG 的四個階段**(用「圖書館問答服務」比喻):
1. **索引(Embedding / 向量化)**:文件進圖書館時,先拆成一段一段,每段都做一張「內容摘要卡」(向量)。
2. **查詢系統(向量資料庫)**:把所有摘要卡存進一個擅長回答「哪些卡片內容最像目前問題」的特殊資料庫。
3. **檢索(Retrieval)**:使用者發問時,先去查詢系統裡找出最相關的幾段內容,不是直接把問題丟給 AI。
4. **生成(Generation)**:把「使用者的問題」+「找到的相關段落」一起丟給 AI,叫它根據這些真實資料回答。

**範圍決定**:針對使用者上傳過的**所有文件**,AI 自己去所有文件裡找答案(不是先選定某一份文件再問),更貼近真實產品(如 Notion AI、Google Drive 搜尋)。

---

## 二、技術選型

### AI 模型:Google Gemini API
決定條件:免費、好串接、支援中文。有免費方案不需信用卡、額度對學習/作品集用量足夠、官方文件有完整繁中版、Python 有官方 SDK。

**踩坑**:官方套件 `google-generativeai` 已被標記 deprecated,改用新的統一套件 `google-genai`：
```bash
pip uninstall google-generativeai -y
pip install google-genai python-dotenv
```
模型名稱 `gemini-2.5-flash` 也已不開放新使用者,改用 `gemini-3.6-flash`(API 錯誤訊息會直接告訴你該換成哪個模型)。

**教訓**:AI 領域套件、模型版本變化非常快,遇到「安裝哪個套件」「用哪個模型名稱」這類容易變動的資訊,最好先查證最新現況,不要完全憑舊記憶操作。

### Embedding 模型:`gemini-embedding-001`
文字專用(2025 推出),因為專案內容是純文字,選這個而非支援多模態的 `gemini-embedding-2`(更輕量、更貼合情境)。輸出維度固定 `3072`。

### 向量資料庫:Chroma
輕量、**內嵌式**(不用另外架設伺服器,直接在 Python 程式裡跑,可存成本機檔案)、免費,新手/中小型 RAG 專案主流入門選擇。概念上跟現在用的 SQLite(檔案型資料庫)很相似,學習曲線最平緩。

---

## 三、核心實作

### 1. API Key 安全存放
沿用 `.env` 固定模式,`python-dotenv` 讀取:
```python
from dotenv import load_dotenv
import os
load_dotenv()
api_key = os.environ.get('GEMINI_API_KEY')
```
驗證檔案有內容但不洩漏內容:用 `wc -l .env`(顯示行數)不用 `cat .env`。

### 2. Chroma 持久化模式
```python
# 暫存模式(測試用,程式結束資料消失)
chroma_client = chromadb.Client()
# 持久化模式(正式使用)
chroma_client = chromadb.PersistentClient(path="./chroma_db")
collection = chroma_client.get_or_create_collection(name="documents")
```
`get_or_create_collection`(不是 `create_collection`):集合已存在就直接拿來用,不存在才建立,避免程式重跑第二次因「集合已存在」報錯。Chroma 底層實際上也是用 SQLite 存索引(`chroma_db/chroma.sqlite3`)。

### 3. 文件切段(Chunking)
**為什麼需要**:整份長文件一次向量化,會像把十幾種顏色的顏料混在一起攪拌,變成模糊平均值;embedding 模型對單次能處理的文字長度也有上限。**解法**:切成小段各自向量化,相鄰段落間保留重疊,避免語意被切在句子中間。
```python
def chunk_text(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return chunks
```

### 4. 多格式文件讀取
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
        return '\n'.join([para.text for para in doc.paragraphs])
    else:
        raise ValueError(f"不支援的檔案格式: {file_path}")
```
`.pdf` 用 `pypdf`(不是已停止維護的 `PyPDF2`)。`.docx` 用 `python-docx`,注意 import 別名 `Document as DocxDocument`(專案自己的 `Document` model 也叫 `Document`,會衝突)。**`.pdf`/`.docx` 目前僅信任套件成熟度,還沒實際測試過**(見「還不確定」)。

### 5. 安全原則:向量資料也要隔離使用者
延續「只能碰自己的東西」的核心原則,每段向量存進 Chroma 時附加 `metadata` 標記歸屬:
```python
collection.add(
    ids=[unique_id],
    embeddings=[result.embeddings[0].values],
    documents=[chunk],
    metadatas=[{'user_id': document.user.id, 'document_id': document.id, 'title': document.title}]
)
```
查詢時用 `where={'user_id': request.user.id}` 篩選,確保 B 使用者永遠查不到 A 的資料。

### 6. 串接進 Django:上傳觸發整條管線
```python
# documents/views.py
class DocumentUploadView(generics.CreateAPIView):
    serializer_class = DocumentSerializer
    permission_classes = [IsAuthenticated]

    def perform_create(self, serializer):
        instance = serializer.save(user=self.request.user)
        process_document_for_rag(instance)
```
關鍵改動:`serializer.save(...)` 要接收回傳值(`.save()` 會回傳剛存好、已有 `id` 的物件),才能立刻觸發 RAG 管線。

### 7. 問答 API
```python
# documents/serializers.py
class DocumentAskSerializer(serializers.Serializer):
    question = serializers.CharField()

# documents/views.py
class DocumentAskView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        serializer = DocumentAskSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        question = serializer.validated_data['question']

        client = get_gemini_client()
        collection = get_chroma_collection()

        query_result = client.models.embed_content(model='gemini-embedding-001', contents=question)
        results = collection.query(
            query_embeddings=[query_result.embeddings[0].values],
            n_results=3,
            where={'user_id': request.user.id}
        )
        retrieved_chunks = results['documents'][0]
        metadatas = results['metadatas'][0]

        if not retrieved_chunks:
            return Response({'answer': '目前找不到相關的文件內容,請確認是否已上傳相關文件。', 'sources': []})

        context = '\n\n'.join(retrieved_chunks)
        prompt = f"請根據以下文件內容,回答使用者的問題。如果文件內容無法回答問題,請誠實告知。\n\n文件內容:\n{context}\n\n問題:{question}"
        response = client.models.generate_content(model='gemini-3.6-flash', contents=prompt)

        sources = []
        seen_ids = set()
        for meta in metadatas:
            if meta['document_id'] not in seen_ids:
                sources.append({'document_id': meta['document_id'], 'title': meta['title']})
                seen_ids.add(meta['document_id'])

        return Response({'answer': response.text, 'sources': sources})
```
```python
# documents/urls.py
path('ask/', views.DocumentAskView.as_view(), name='document-ask'),
```

### 8. 端對端手動驗證
用 curl 上傳「請假規定」文件,再問「特休要提前幾天跟主管講?」(字面跟原文「特休假需提前3天向主管申請」完全不同),成功拿到正確答案,證明語意檢索真的有效,不是關鍵字比對。

---

## 四、自動化測試

跟 Sprint 2 一樣的邏輯(正常情況 / 未登入 / 跨使用者),但 `document-ask` 沒有 `<int:pk>`,不是針對特定文件,所以跨使用者情境**不是預期 404**,而是預期 200、但 `sources` 裡不該出現對方的文件:

```python
def test_ask_success(self):
    self.client.force_authenticate(user=self.user_a)
    with open('test.txt', 'w') as f:
        f.write('公司地址位於台北市信義區松仁路100號。')
    with open('test.txt', 'rb') as f:
        self.client.post(reverse('document-upload'), {'title': '公司地址文件', 'file': f})
    response = self.client.post(reverse('document-ask'), {'question': '公司在哪裡?'})
    self.assertEqual(response.status_code, status.HTTP_200_OK)
    self.assertIn('answer', response.data)
    self.assertTrue(len(response.data['answer']) > 0)
    source_titles = [s['title'] for s in response.data['sources']]
    self.assertIn('公司地址文件', source_titles)

def test_ask_unauthenticated(self):
    response = self.client.post(reverse('document-ask'), {'question': '隨便問一個問題'})
    self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

def test_ask_does_not_leak_other_users_documents(self):
    self.client.force_authenticate(user=self.user_a)
    with open('test.txt', 'w') as f:
        f.write('特殊通關密語是紫色的獨角獸在跳舞。')
    with open('test.txt', 'rb') as f:
        self.client.post(reverse('document-upload'), {'title': 'A的秘密文件', 'file': f})
    self.client.force_authenticate(user=self.user_b)
    response = self.client.post(reverse('document-ask'), {'question': '特殊通關密語是什麼?'})
    self.assertEqual(response.status_code, status.HTTP_200_OK)
    source_titles = [s['title'] for s in response.data['sources']]
    self.assertNotIn('A的秘密文件', source_titles)
```

**斷言設計教訓**:`assertEqual(len(sources), 1)` 這種寫法,預設了「Chroma 資料是乾淨的」這個不成立的前提(Chroma 不像 Django 測試資料庫,不會隨測試「砍掉重建」)。改成 `assertIn` 只確認「該有的東西有出現」,不依賴總筆數。

---

## 五、一次完整的除錯連環——測試環境污染 Chroma

這是 Sprint 3 最有價值的除錯過程,記錄完整時間軸:

1. **斷言太嚴格失敗**:`assertEqual(len, 1)` 撈到 3 筆 → 改用 `assertIn`。
2. **改完還是失敗,而且更嚴重**:自己剛上傳的文件完全找不到。用 `print(upload_response.status_code, upload_response.data)` 確認上傳本身成功(`201`),排除「上傳失敗」,鎖定問題在查詢端。
3. **追查到根本原因**:`get_chroma_collection()` 路徑寫死成 `"./chroma_db"`,**測試環境跟正式開發環境共用同一份持久化資料庫**。加上 Django 測試資料庫每次方法結束會 rollback,`document.id`、`user.id` 常重複用到同一數字,測試資料跟真實 curl 測試資料的 `user_id` 可能剛好對上,互相污染。
4. **第一次嘗試修正**:加上 `settings.TESTING = 'test' in sys.argv` 開關,讓測試走獨立的 `chroma_db_test/`;同時在 `setUp()` 加 `shutil.rmtree('chroma_db_test', ignore_errors=True)`,想每次測試前清空。
5. **清空資料夾這招反而搞出更嚴重的錯**:`chromadb.errors.InternalError: ... attempt to write a readonly database`——Chroma 背景保留著上一個測試方法還沒關閉的資料庫連線,`shutil.rmtree` 把連線指向的實體檔案硬生生刪掉,SQLite 偵測到檔案消失,為保護資料自動切成唯讀模式拒絕寫入。**暴力清空資料夾跟 Chroma 自己的連線快取機制互相衝突。**
6. **回到問題最根本處**:不該用「清空整個儲存空間」繞過問題,而該讓 `id` 從源頭保證不重複:
   ```python
   import uuid
   unique_id = f"doc{document.id}_chunk{i}_{uuid.uuid4().hex[:8]}"
   ```
   撤銷 `shutil.rmtree`,改用這個方式後,**11 個測試全數通過**。
7. **意外插曲**:撤銷 `shutil.rmtree` 那次,貼 heredoc 指令中途被 `Ctrl+C` 打斷,誤以為改完了,實際上沒有——測試又出現一樣的唯讀資料庫錯誤。用 `cat documents/tests.py | grep -A 3 "def setUp"` 重新確認,證實「以為做完的事,實際上沒做完」。改用更安全、風險更低的精準修改方式:
   ```bash
   sed -i "/shutil.rmtree/d" documents/tests.py
   sed -i "/^import shutil$/d" documents/tests.py
   ```

**整體教訓**:這是一次典型「解法引發新問題,新問題又要用另一個角度解決」的除錯過程。回頭看,真正治本的解法(uuid)其實最簡單,前面繞的彎(改斷言、清空資料夾)都是必要的排查,每一步都排除了一個可能性,才縮小範圍找到真正根源。

---

## 六、其他過程中排查過的問題

- **`cat` heredoc 指令貼到一半,內容跑進檔案本身**:貼一段 `cat > 檔案 << 'EOF' ... EOF` 若沒完整框住整段,bash 會誤判、報 `syntax error`,或把部分內容寫進目標檔案。處理方式:先用 `cat 檔案` 驗證實際內容,錯誤就用 `cat >`(覆蓋)重新完整貼一次,整段一次複製。
- **`urls.py` 一度疑似少了收尾的 `]`**:靠 `wc -l` 加上 `cat -A ... | tail -3`(把換行符號印成 `$`)才確認是真的缺,不是格式瑕疵——遇到矛盾線索時,要找更明確、排除干擾的指令驗證,不能只靠推測下結論。
- **Sprint 2 遺留的未提交改動**:`upload_to` 被改回舊值,`git status`/`git diff` 抓出本地檔案跟正式 commit 不一致,用 `git restore` 修正。

---

## 七、還不確定、需要之後多練習的地方

1. **Chroma 查詢的 `where` 篩選語法進階用法**:目前只用過 `where={'user_id': ...}` 單一條件,還沒接觸更複雜的篩選組合。
2. **`.pdf`/`.docx` 讀取還沒真的測試過**:選擇信任套件成熟度跳過驗證,之後第一次真上傳 PDF/Word 檔要留意排版複雜的 PDF 抽字效果。
3. **一份文件的向量化會呼叫多次 Gemini API**(一段一次),文件越長 API 呼叫次數越多,免費額度限制還沒仔細研究。
4. **Chroma 的連線快取機制細節**:親眼見證刪除正在使用中的資料庫檔案會導致唯讀錯誤,但還不清楚 `PersistentClient` 內部具體怎麼快取連線的。
5. **`chroma_db_test/` 會不會無限累積**:每次測試新向量資料都往同一份資料夾加,長期下來可能變大、變慢,之後可考慮定期清空或研究更乾淨的隔離方式。
6. **測試會打真實 Gemini API**:每次執行都消耗真實額度,之後測試數量變多可能要學習 mock(模擬)外部服務。
7. **`sed` 的其他用法**:目前只會用 `sed -i "/關鍵字/d"` 刪整行,還沒摸熟替換內容等其他常見用法。

---

## 八、下一步

1. 實際測試 `.pdf`/`.docx` 上傳與讀取
2. 更新 `progress.md`,反映 Sprint 3 完整收尾
3. 視進度更新履歷上的專案描述
