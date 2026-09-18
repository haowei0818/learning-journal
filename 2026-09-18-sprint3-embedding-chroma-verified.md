# Sprint 3 續 — Embedding 模型、Chroma 向量資料庫選型與端對端驗證

## 今天完成的事

延續今天稍早的 Gemini API 串接驗證，繼續完成 RAG 架構的第二塊積木：技術選型「向量化」與「向量資料庫」，並成功做出一次**端對端的最小驗證**——證明「語意相近但文字不同的兩句話，也能被正確配對」這個 RAG 的核心能力真的可行。

---

## 技術選型：embedding 模型與向量資料庫

### Embedding 模型：`gemini-embedding-001`

查證後確認 Gemini 目前有兩個 embedding 模型：
- `gemini-embedding-001`：文字專用（2025 年推出）
- `gemini-embedding-2`：最新版，支援多模態（文字/圖片/影片/音訊）與 100+ 種語言

因為專案的文件內容是純文字，選用 **`gemini-embedding-001`**——專為文字設計、更貼合情境，也比多模態版本更輕量。

Sources:
- [Gemini Embedding model | Gemini API | Google AI for Developers](https://ai.google.dev/gemini-api/docs/models/gemini-embedding-001)
- [Embeddings | Gemini API | Google AI for Developers](https://ai.google.dev/gemini-api/docs/embeddings)

### 向量資料庫：Chroma

比較了 Chroma、FAISS、Pinecone/Qdrant 三類選項：
- **Chroma**：輕量、**內嵌式**（不用另外架設伺服器，直接在 Python 程式裡跑，可存成本機檔案），免費，新手/中小型 RAG 專案的主流入門選擇
- **FAISS**：效能好但偏底層，需要自己處理更多細節
- **Pinecone / Qdrant**：功能強大，但通常需要額外架設服務或雲端帳號，超出目前專案規模

**選用 Chroma**，理由是延續 MVP 精神：不需要額外服務或帳號，裝一個 Python 套件就能在現有 WSL2 環境跑起來，概念上跟現在用的 SQLite（檔案型資料庫，不用開額外服務）很相似，學習曲線最平緩。

Sources:
- [Chroma Vs FAISS Vs Pinecone In 2026: Which Vector Database Fits RAG?](https://www.designveloper.com/blog/chroma-vs-faiss-vs-pinecone/)
- [Best Vector Databases in 2026: A Complete Comparison Guide](https://www.firecrawl.dev/blog/best-vector-databases)

---

## 核心概念與操作整理

### 1. 安裝 Chroma

```bash
pip install chromadb
```

跟今天前面裝 `google-genai` 一樣，套件較大、下載可能逾時，直接重跑同一行指令即可（不是操作錯誤）。

### 2. 單獨驗證「向量化」——把文字轉成一串數字

```python
result = client.models.embed_content(
    model='gemini-embedding-001',
    contents='這是一份關於請假規定的文件'
)
print(len(result.embeddings[0].values))
# 輸出：3072
```

- `embed_content(...)`：呼叫 Gemini 的向量化功能，把一段文字轉換成一長串數字（向量）。
- `result.embeddings[0].values`：實際的向量內容（一個長度固定的數字陣列）。
- `3072` 是 `gemini-embedding-001` 固定的輸出維度——之後每一段文字轉出來的向量都會是這個長度，這樣才能互相比對「像不像」。

### 3. 驗證 Chroma：存進去、再用語意查詢找出來

```python
import chromadb

chroma_client = chromadb.Client()   # 暫存模式，程式結束資料就消失，純測試用
collection = chroma_client.create_collection(name="test_collection")

# 存進去
collection.add(
    ids=["doc1"],
    embeddings=[result.embeddings[0].values],
    documents=["這是一份關於請假規定的文件"]
)

# 用完全不同的文字去查詢
query_result = client.models.embed_content(
    model='gemini-embedding-001',
    contents='員工可以請幾天假?'
)
results = collection.query(
    query_embeddings=[query_result.embeddings[0].values],
    n_results=1
)
print(results['documents'])
# 輸出：[['這是一份關於請假規定的文件']]
```

**這次驗證證明的關鍵能力**：「請假規定」跟「員工可以請幾天假」這兩句話**完全沒有重複的文字**，但 Chroma 還是正確把它們配對在一起——因為向量化後比對的是「意思」，不是「字面」。這就是向量搜尋的核心價值，也是整個 RAG 機制能運作的基礎：使用者不需要用文件裡一模一樣的字眼發問，AI 也能找到相關內容。

**API 拆解**：
- `chromadb.Client()`：建立一個 Chroma 連線，這次用最簡單的暫存模式（之後正式整合進專案時，要改成能持久化存檔的模式，不會因為程式結束就消失）。
- `create_collection(name=...)`：建立一個「集合」，概念上類似資料庫裡的一張表，用來放某一批向量。
- `collection.add(ids=..., embeddings=..., documents=...)`：把「識別碼」「向量」「原始文字」三者綁在一起存進去。
- `collection.query(query_embeddings=..., n_results=1)`：拿一個新的向量去查詢，`n_results` 控制要回傳幾筆最相似的結果。

---

## 過程中排查過的問題

沒有出現需要排查的錯誤，兩個套件、兩個 API 呼叫都一次成功——這次比稍早裝 `google-generativeai`/模型下架那兩次順利，因為查證模型名稱這個習慣已經先養成了。

---

## 還不確定、需要之後多練習的地方

1. **Chroma 的持久化模式**：今天用的是 `chromadb.Client()`（暫存、程式結束就消失），還沒用過能真正把資料存成檔案、下次啟動還能讀回來的模式（應該是 `PersistentClient`），這是下一步整合進 Django 時必須弄懂的部分。
2. **`n_results` 以外的查詢參數**：目前只用了「回傳最相似的 1 筆」，還沒了解如何篩選相似度門檻、或做更細緻的查詢條件。
3. **向量維度是否有免費方案的限制**：`gemini-embedding-001` 每次呼叫都要花一次 API 請求額度，一份文件切成很多段時，可能會累積不少次呼叫，這部分的額度限制還沒仔細研究。

---

## 下一步

RAG 四階段裡，「向量化」與「向量資料庫」這兩塊積木的**技術選型跟最小驗證**已經完成。接下來要做的是把這些零散驗證過的程式碼，**整合進 Django 專案**：

1. Chroma 改用持久化模式，讓向量資料能真正保存下來
2. 設計「文件切段（chunking）」邏輯：一份文件要怎麼拆成適合向量化的小段落
3. 讀取 `Document.file` 實際存放的檔案內容（目前只存了路徑，要能真的打開檔案、取出文字）
4. 把「上傳文件」的流程，串接上「切段 → 向量化 → 存進 Chroma」這條管線
5. 設計問答 API：使用者發問 → 向量化問題 → Chroma 查詢相關段落 → 組成 prompt 丟給 Gemini → 回傳答案
6. 之後補上自動化測試
