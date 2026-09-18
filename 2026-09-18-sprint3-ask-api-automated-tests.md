# Sprint 3 續 — 問答 API 自動化測試,以及一連串測試環境污染除錯

## 今天完成的事

延續上一份筆記(問答 API 串接、端對端手動測試),今天幫 `DocumentAskView` 補上完整的自動化測試(正常情況、未登入、跨使用者情境三種),過程中意外挖出一個比預期嚴重的架構問題——**測試環境跟真實開發環境,原來一直共用同一份 Chroma 向量資料庫**,順藤摸瓜一路除錯,最後從根本上修正。程式碼已 commit、push。

---

## 核心概念與操作整理

### 1. 測試案例設計:先想清楚「這支 API 沒有 `<int:pk>`,代表什麼」

延續 Sprint 2 的邏輯(正常情況 / 未登入 / 跨使用者三個角度去設計測試),但這次多了一層思考:`document-ask` 這支 API 的網址 `path('ask/', ...)` **沒有 `<int:pk>`**,不像刪除是「針對某一份特定文件」。

這代表「跨使用者」這個情境,不能照搬 Sprint 2「預期回 404」的邏輯——因為 404 的核心概念是「這個特定的東西對你來說不存在」,但 `document-ask` 從頭到尾都不是在指定查某個特定文件。正確的測試設計是:**B 使用者問一個「答案只存在 A 文件裡」的問題,預期還是回 200,但 `sources` 裡不該出現 A 的文件**——安全邊界要看「回應內容有沒有洩漏」,不是看狀態碼。

三個測試案例:
```python
def test_ask_success(self):
    # 上傳文件、問相關問題,確認 answer 有內容、sources 有找到剛上傳的文件

def test_ask_unauthenticated(self):
    # 完全不帶 token,預期 401

def test_ask_does_not_leak_other_users_documents(self):
    # A 上傳一份內容獨特的文件,切換成 B 去問,確認 sources 裡不會出現 A 的文件
```

### 2. 斷言設計別預設「資料庫是乾淨的」

一開始寫 `test_ask_success` 用 `self.assertEqual(len(response.data['sources']), 1)`,結果測試失敗,`sources` 有 3 筆——原因是 **Chroma 不像 Django 測試資料庫,不會隨著每次測試「砍掉重建」**,之前手動 curl 測試、之前跑過的測試留下的舊資料,會一直累積在同一份 `chroma_db/` 裡。

**教訓**:當一個外部資料來源沒辦法保證「每次都是乾淨、從零開始」的時候,斷言不該依賴「總共有幾筆」,而該改成確認「該有的東西有沒有出現」:
```python
source_titles = [s['title'] for s in response.data['sources']]
self.assertIn('公司地址文件', source_titles)   # 而不是 assertEqual(len(...), 1)
```

### 3. 發現真正的架構問題:測試環境跟正式環境共用同一份 Chroma

斷言改對之後,測試還是失敗,而且更嚴重——**這次連自己剛上傳的文件都完全沒被找到**。用印出上傳回傳結果(`print(upload_response.status_code, upload_response.data)`)確認上傳本身是成功的(`201`),代表問題不在上傳這一步,而在「查詢撈出來的結果不對」。

追查後發現根本原因:`get_chroma_collection()` 裡把路徑寫死成 `path="./chroma_db"`,**不管是 `python manage.py test` 還是 `python manage.py runserver`,用的都是同一個資料夾**。加上 Django 測試資料庫每次測試方法結束都會 rollback(回滾),`document.id`、`user.id` 常常會重複用到同一個數字(例如都是 `1`)——這代表測試存進 Chroma 的資料,跟你手動 curl 測試時、真實帳號留下的資料,`user_id` 可能剛好對上,兩邊的資料混在一起,查詢時互相干擾。

**修正:讓 `get_chroma_collection()` 依照環境切換路徑**
```python
# config/settings.py 最後加上
import sys
TESTING = 'test' in sys.argv
```
```python
# documents/rag_utils.py
def get_chroma_collection():
    import chromadb
    from django.conf import settings

    if getattr(settings, 'TESTING', False):
        path = "./chroma_db_test"
    else:
        path = "./chroma_db"

    chroma_client = chromadb.PersistentClient(path=path)
    return chroma_client.get_or_create_collection(name="documents")
```
`'test' in sys.argv`:檢查執行 `python manage.py` 時帶的參數清單裡,有沒有出現 `'test'` 這個字——`runserver` 不會有,`test` 才會有,藉此判斷目前是不是在測試環境,測試環境用完全獨立的 `chroma_db_test/` 資料夾,不會再跟正式環境的資料混在一起。

### 4. 「清空資料夾」這招,撞上 Chroma 自己的連線管理,反而搞出更嚴重的錯

環境隔開之後,想到同一次測試裡,不同測試方法之間 `document.id` 還是可能重複(因為 Django 測試資料庫的 rollback),於是嘗試在 `setUp()` 裡加上 `shutil.rmtree('chroma_db_test', ignore_errors=True)`,每個測試方法開始前先把測試專用的 Chroma 資料夾整個砍掉重來。

結果跳出新的錯誤:
```
chromadb.errors.InternalError: Query error: Database error: error returned from database:
(code: 1032) attempt to write a readonly database
```

**原因**:Chroma 在背景很可能保留著上一個測試方法還沒關閉的資料庫連線,`shutil.rmtree` 把這個連線指向的實體檔案硬生生刪掉,SQLite(Chroma 底層儲存引擎)偵測到自己的檔案不見了,為了保護資料不被破壞,直接切成唯讀模式拒絕寫入——**暴力清空資料夾的做法,跟 Chroma 自己內部的連線快取機制互相衝突**,不是個好方法。

**教訓**:解決「id 可能重複造成互相覆蓋」這個問題,不該用「清空整個儲存空間」這種暴力手段去繞過,而該回到問題最根本的地方——直接讓 id 保證不會重複。

### 5. 真正的修法:讓每一筆存進 Chroma 的 id,加上隨機尾巴保證不重複

撤銷 `shutil.rmtree` 的做法,改成從源頭讓 `id` 不可能撞名:

```python
import uuid
# ...
unique_id = f"doc{document.id}_chunk{i}_{uuid.uuid4().hex[:8]}"
collection.add(
    ids=[unique_id],
    ...
)
```

`uuid.uuid4()`:Python 內建產生「通用唯一識別碼」的方式,每次呼叫都會產生一組幾乎不可能重複的亂碼(概念上有點像每個人的身分證字號,理論上不會有兩個人一樣)。`.hex[:8]` 取前 8 碼英數字當作尾巴接上去,不管 `document.id` 重不重複,整串 `id` 幾乎不可能撞名。

改完之後,先手動 `rm -rf chroma_db_test`(把之前被搞壞、卡在唯讀狀態的資料夾清乾淨,這是一次性的手動清理,不是常態做法),重新跑測試,**11 個測試全數通過**。

### 6. 貼指令中途被 `Ctrl+C` 中斷,以為改完了,其實沒有——用 `grep` 快速二次確認

拿掉 `shutil.rmtree` 那次,貼 heredoc 指令中途不小心被 `Ctrl+C` 中斷,以為修改完成,結果测试又出現一模一樣的唯讀資料庫錯誤。用
```bash
cat documents/tests.py | grep -A 3 "def setUp"
```
一查,發現 `shutil.rmtree` 那行根本還在——**證實了「以為做完的事,實際上沒做完」**。

**教訓(呼應這幾天一直在練習的原則)**:不管多確定自己剛剛的指令有跑完,只要牽涉到「改檔案」,養成用 `cat` 或 `grep` 之類的指令**重新讀一次實際內容**再往下走的習慣,不要只靠「印象中應該改完了」去判斷。

這次也順便學到一個更安全的改法:與其整份檔案用 `cat >` heredoc 重寫(貼的內容一多,中途被打斷或貼壞的風險就變高),**只改一兩行的小修改,改用 `sed -i "/關鍵字/d" 檔案`(刪除含有關鍵字的那一行)更精準、風險更低**：
```bash
sed -i "/shutil.rmtree/d" documents/tests.py
sed -i "/^import shutil$/d" documents/tests.py
```

### 7. `.gitignore` 要記得跟著新出現的資料夾更新

新增了 `chroma_db_test/` 這個測試專用資料夾後,`.gitignore` 也要跟著補上,避免測試產生的資料被誤 commit：
```bash
echo 'chroma_db_test/' >> .gitignore
```

---

## 過程中排查過的問題(這次特別多,整理一次除錯的完整時間軸)

1. **斷言太嚴格**(`assertEqual(len, 1)`)→ 發現 Chroma 資料會跨測試累積,改成 `assertIn`。
2. **改完斷言還是失敗,而且更嚴重**(自己剛上傳的文件都找不到)→ 用 `print` 印出上傳回傳結果,排除「上傳失敗」的可能 → 追查到 `get_chroma_collection()` 路徑寫死,測試環境跟正式環境共用同一份資料庫。
3. **加上環境區隔後,嘗試用 `shutil.rmtree` 清空測試資料夾**→ 撞上 Chroma 自己的連線快取機制,爆出「唯讀資料庫」的新錯誤。
4. **改用「id 加 uuid 保證不重複」的根本解法**,撤銷清空資料夾的做法。
5. **撤銷的指令中途被 `Ctrl+C` 打斷**,誤以為改完了,實際上沒有 → 用 `grep` 重新確認 → 改用更安全的 `sed -i` 精準刪除,而不是整份重貼。

這是一次典型的「解法引發新問題,新問題又要用另一個角度解決」的除錯過程,最後回頭看,真正治本的解法(uuid)其實是最簡單的一步,前面繞的兩個彎(改斷言、清空資料夾)都是必要的排查過程,不是白費工夫——每一步都排除了一個可能性,才能縮小範圍找到真正的根源。

---

## 還不確定、需要之後多練習的地方

1. **Chroma 的連線快取機制細節**:今天親眼見證了「刪除正在被使用的資料庫檔案會導致唯讀錯誤」,但還不清楚 `chromadb.PersistentClient` 內部具體是怎麼快取連線的(例如是不是用路徑當 key 做全域快取),這部分還只停留在「知道會出事,不知道確切原理」。
2. **`chroma_db_test/` 會不會無限累積**:雖然已經跟正式環境隔開,但每次跑測試,新的向量資料還是會一直往同一份 `chroma_db_test/` 裡加,長期下來檔案會越來越大、查詢可能越來越慢。目前先不處理,之後可以考慮定期手動清空,或研究更乾淨的測試隔離方式(例如每次測試都用全新的臨時資料夾)。
3. **測試會真的打真實 Gemini API 這件事,還沒有處理**:目前這幾個問答相關的測試,每次執行都會真的呼叫 Gemini API、消耗真實額度,之後如果測試數量變多,可能要考慮學習「mock(模擬)外部服務」的技巧,讓測試不用依賴真實網路呼叫。
4. **`sed -i` 的其他用法**:今天第一次用 `sed -i "/關鍵字/d"` 做精準刪除,還沒摸熟 `sed` 其他常見的用法(例如替換某一行的內容,而不只是整行刪除)。

---

## 下一步

1. Sprint 3 的核心功能(上傳、切段、向量化、問答、自動化測試)已經全部完成,可以考慮更新 `progress.md`,做一次 Sprint 3 的整體收尾
2. 把 Sprint 3 累積的好幾份筆記(`gemini-setup`、`embedding-chroma-verified`、`rag-pipeline-integration`、`ask-api-e2e-test`、這份 `ask-api-automated-tests`),合併整理成一份完整的 Sprint 3 筆記
3. 之後找時間實際測試 `.pdf`/`.docx` 上傳與讀取(從 Sprint 3 一開始就記錄的待辦,一直還沒處理)
4. 視進度更新履歷上的專案描述
