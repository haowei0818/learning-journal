# T5 手動重新處理 API：冪等、權限、用「故意破壞」驗證一跑就綠的測試

> 專案：AI 文件管理平台（LLM 整合層工程化，任務卡 T5）
> 分支：`t5-reprocess`，PR #3
> 測試數量：37 → 42（新增 5 個）

---

## 一、今天完成的事

新增 `POST /api/documents/<pk>/reprocess/`：處理失敗的文件（例如當天額度用完）可以一鍵重跑，不需要刪掉重傳。

### 程式碼（只有 12 行 view + 1 行網址）

```python
# documents/views.py
class DocumentReprocessView(generics.GenericAPIView):
    serializer_class = DocumentSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Document.objects.filter(user=self.request.user)   # 只找得到自己的文件

    def post(self, request, pk):
        document = self.get_object()          # 找不到（或不是自己的）→ 自動回 404
        process_document(document)            # 跟上傳走同一條流程
        return Response(self.get_serializer(document).data)
```

```python
# documents/urls.py
path('<int:pk>/reprocess/', views.DocumentReprocessView.as_view(), name='document-reprocess'),
```

### 測試與故意破壞驗證

| 測試 | 一跑的結果 | 故意破壞 | 破壞後 |
|---|---|---|---|
| `failed` → 重新處理 → `ready`，可以問答 | 🔴 NoReverseMatch（真正的紅燈） | — | — |
| 連續重新處理兩次，chunk 數量不變 | 🟢 | 拿掉「處理前先清舊 chunk」 | FAIL `9 != 3` |
| 未登入 → 401 | 🟢 | 拿掉 `permission_classes` | ERROR `TypeError`（AnonymousUser） |
| 別人的文件 → 404 | 🟢 | `get_queryset` 改成 `.all()` | FAIL `200 != 404` |
| 重新處理失敗 → `failed`，沒有殘留 | 🟢 | 拿掉「失敗時清殘留」 | FAIL 剩下 `['doc1_chunk0_...']` |

---

## 二、核心概念

### 1. 好的設計讓下一張卡變簡單
T3 時就把 `process_document()` 寫成「上傳和重新處理共用」，所以 T5 幾乎只是接一個新的 API 入口。

### 2. 冪等（idempotent）
同一個動作做 1 次和做 10 次，結果完全一樣。電梯的「關門」按鈕是冪等的，「投幣」不是。
- 關鍵是「先清舊 chunk 再重建」。不清的話，每重跑一次 chunk 就多一份，問答搜尋結果會被重複內容佔滿
- 比較的是 chunk **數量**，不是 id，因為 id 裡有隨機的 uuid

### 3. `GenericAPIView`：附工具箱的空機台
`DestroyAPIView` 是組裝好的刪除機；要做自訂動作（重新處理）時，用 `GenericAPIView`，它提供 `get_object()`、`get_serializer()`，動作自己寫。

### 4. 用 `get_queryset` 做到 404 而不是 403
`get_object()` 只在 `get_queryset()` 的範圍裡找。別人的文件根本不在範圍裡 → 找不到 → 404。不會透露「這份文件存在，只是你沒權限」。

### 5. 未登入時 `request.user` 是 `AnonymousUser`，不是 `None`
拿掉 `permission_classes` 後，未登入的請求真的會被放進來，結果在 `filter(user=AnonymousUser)` 爆出 `TypeError` → 500，而不是 401。
- 專案的 `settings.py` **沒有**設定全專案預設必須登入，所以每個 view 都要自己寫 `permission_classes`
- 「必須登入」要在最前面擋住，不能靠後面的程式碰巧出錯

### 6. 守衛斷言
`self.assertGreater(first_count, 1)`：檢查「測試的前提成立」。如果哪天 `chunk_size` 被改大、文件只剩 1 段，這行會提醒測試已經不夠嚴格。

### 7. API 有收到 ≠ LLM 處理成功
重新處理失敗仍回 200，用 `status` 欄位表示結果。乾洗店收件了（200），洗完告訴你「洗好了」或「洗不掉」。

### 8. 測試選最嚴苛的情況
失敗測試選「原本是 `ready`、有舊 chunk」+「第 1 段成功、第 2 段才失敗」，一次驗證兩種殘留（舊的、寫到一半的）都會被清掉。

---

## 三、待釐清問題

- [ ] 重新處理是**同步**的：文件很大時，請求會等到全部 embed 完才回應。目前範圍不處理（不引入 Celery 這類新元件），但面試可能被問「如果文件很大怎麼辦」
- [ ] 文件正在 `pending` 時又按一次重新處理，會怎樣？目前是同步處理，同一個使用者不太會撞到，但可以想想多人或非同步時的情況
- [ ] 是否要在 `settings.py` 設定 `DEFAULT_PERMISSION_CLASSES` 當作安全網？（不在這次範圍）

---

## 四、下一步

1. 合併 PR #3（`t5-reprocess` → `main`）
2. **T6 問答時 LLM 失敗回 503**：`DocumentAskView` 接住 T4 的 `LLMUnavailableError`，回 503，且回應不含內部錯誤細節
3. 之後：T7 結構化 log、T8 真實整合測試、T9 README 與履歷
