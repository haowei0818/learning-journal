# Sprint 3 收尾 — .pdf/.docx 格式驗證、AI 幻覺問題、Prompt Engineering

## 今天完成的事

Sprint 3 最後一塊待辦:實際驗證 `.docx`、`.pdf` 兩種格式的上傳與問答是否真的可行(先前只驗證過 `.txt`)。過程中意外發現 AI 生成答案時出現「幻覺」(講出文件裡沒有的內容),嘗試用 Prompt Engineering(強化提示文字)緩解。**Sprint 3 核心功能與格式支援至此全部驗證完成並收尾。**

---

## 核心概念與操作整理

### 1. 用程式碼產生測試檔案,不用真的開 Word/找現成 PDF

`.docx` 用已經安裝的 `python-docx` 直接生成:
```python
from docx import Document
doc = Document()
doc.add_paragraph('公司福利規定：')
doc.add_paragraph('員工滿一年可享有年終獎金，金額為月薪的兩倍。')
doc.save('test_benefits.docx')
```

`.pdf` 因為 `pypdf`(讀取用的套件)不支援「寫入」,另外安裝 `reportlab`(專門產生 PDF 的套件):
```bash
pip install reportlab
```
```python
from reportlab.pdfgen import canvas
c = canvas.Canvas("test_policy.pdf")
c.drawString(100, 750, "Remote work policy:")
c.drawString(100, 730, "Employees may work from home up to 2 days per week.")
c.save()
```
**注意**:`reportlab` 內容故意用英文——`reportlab` 預設字型不一定支援中文字元,先用英文排除「字型問題」這個變數,單純驗證「`.pdf` 讀取管線本身能不能跑」。也因為 `reportlab` 只是臨時拿來造測試資料,不是專案正式依賴,沒有加進 `requirements.txt`。

### 2. 兩種格式端對端驗證結果

- **`.docx`**:上傳「福利規定」文件,問「做滿一年可以拿到多少年終?」,正確答出「月薪的兩倍」。**驗證通過。**
- **`.pdf`**:上傳「遠端工作政策」文件,問「一週可以在家工作幾天?」,正確答出「2 days」。**驗證通過。**

至此 `.txt`/`.docx`/`.pdf` 三種格式都已端對端驗證過,`extract_text_from_file` 這個函式的多格式分派邏輯確實可靠。

### 3. 發現 AI「幻覺」(Hallucination)問題

第一次測 `.docx` 時,AI 除了正確答出「月薪兩倍」,還多講了一段:
```
2. 年終獎金發放規定：發放金額依當年度公司整體營運狀況而定，並參考個人當年度績效考核結果。
```
但上傳的測試文件裡**完全沒有**「績效考核」「營運狀況」這些內容——AI 根據自己訓練時學過的「一般常識」(很多公司年終確實會參考績效),編造出一段聽起來合理、但不是根據提供文件的內容。

**這個現象叫做「幻覺」(Hallucination)**:即使 RAG 系統的檢索(retrieval)這一步做對了,生成(generation)這一步的 LLM 還是可能不老實地只根據給定資料回答,摻雜自己的臆測。這是所有 LLM 應用的已知風險,不是程式碼寫錯。

### 4. Prompt Engineering:嘗試用更嚴謹的指令緩解幻覺

修改 `DocumentAskView` 的 prompt,原本:
```python
prompt = f"""請根據以下文件內容，回答使用者的問題。如果文件內容無法回答問題，請誠實告知。
文件內容：{context}
問題：{question}"""
```
改成更明確、更嚴格的版本:
```python
prompt = f"""你是一個嚴謹的文件問答助理。請「只根據」以下提供的文件內容回答問題，絕對不要加入文件內容沒有提到的資訊，也不要根據你自己的知識、常識或推測做任何補充或延伸建議。

如果文件內容不足以回答問題，請直接回覆「文件中沒有提到相關資訊」，不要嘗試自行推論、舉例或給出一般性建議。

文件內容：
{context}

問題：{question}"""
```

**結果:幻覺沒有完全消失**——用同一個問題重測,AI 還是講出了「績效考核」這段文件外的內容。這證實了一個重要認知:**就算把 prompt 寫得再嚴謹,LLM 也不保證 100% 遵守指令,幻覺只能降低發生機率,無法用一句話徹底根除**。業界為此發展出更進階的技巧(例如要求模型附上逐字引用原文出處、用另一個模型做二次事實查核),超出目前 MVP 範圍,先誠實記錄為已知限制(known limitation),不追求短期內完全解決。

### 5. 除錯過程的插曲:一場誤會

上傳 `.pdf` 時一度看到一整片 Django `DEBUG=True` 的錯誤除錯頁面(HTML),貼過來的內容剛好又是頁面後半段(環境變數、settings 清單),看不到真正的錯誤訊息。改用 `curl -o 檔案` 把回應存成檔案、再用 `head -60` 看檔案最前面的做法後,才發現**這次的問題其實跟前面那份看起來嚇人的 HTML 完全無關**——真正的原因很單純:**JWT access token 過期了**(`"message":"Token is expired"`),重新登入拿新 token 就解決了。

**教訓**:
- 遇到看起來複雜的錯誤,不要被一大片不熟悉的輸出嚇到,先想辦法**篩選/存檔看關鍵部分**,而不是被整片內容淹沒。
- 這次也學到 `curl -o 檔案` 這個參數:把 curl 的回應內容直接存成檔案,而不是印在終端機——對付這種可能很長的回應(不管是正常的大量資料,還是錯誤頁面)特別好用,搭配 `head`/`grep` 篩選閱讀更輕鬆。

---

## 還不確定、需要之後多練習的地方

1. **AI 幻覺沒有徹底解決方案**:目前只能靠 prompt 措辭降低機率,還沒接觸更進階的緩解技巧(例如要求附上逐字引用、多模型交叉驗證)。
2. **JWT token 效期管理**:目前每次 token 過期都要手動重新登入,還沒研究 `djangorestframework-simplejwt` 的 refresh token 機制怎麼在 API 層面自動換發新的 access token。

---

## 下一步

Sprint 3 核心功能與測試至此**全部完成**。接下來確認過方向:目標是「GitHub repo 程式碼紀錄完整、README 說明清楚,靠履歷文字 + 口頭介紹」,不做前端/部署。所以下一步是:

1. 撰寫 `ai-document-platform` 的完整 `README.md`(專案簡介、技術棧、架構說明、API 清單、本機執行方式、測試方式)
2. 更新履歷專案描述,補上 Sprint 3(RAG 問答功能)完成的內容
3.(可選加分項)研究 mock 外部 API 呼叫的技巧,讓測試不用每次都打真實 Gemini API
