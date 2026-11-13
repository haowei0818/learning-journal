# 學習紀錄（Learning Journal）

非本科系轉職 Python 後端工程師的自學紀錄，主要記錄 [AI 文件管理平台](https://github.com/haowei0818/ai-document-platform)（Django REST API + RAG）從零開始的開發過程：每一步的設計思考、程式碼、踩過的坑和待釐清的問題。

## 目錄

### Sprint 0：開發環境
- [開發環境建置](2026-07-27-sprint0-environment.md)
- [Django 專案初始化](2026-07-28-sprint0-django-init.md)
- [到新電腦接續開發 SOP](2026-07-28-new-machine-setup.md)

### Sprint 1：帳號模組
- [帳號模組（Accounts）](2026-07-28-sprint1-accounts.md)
- [LoginTests 撰寫紀錄](2026-08-03-sprint1-login-tests.md)
- [MeViewTests 撰寫紀錄](2026-08-03-sprint1-me-tests.md)

### Sprint 2：文件管理
- [Document Model 設計與建立](2026-08-sprint2-document-model.md)
- [Documents API 設計與實作（Serializer / Views / URLs）](2026-09-06-sprint2-documents-api-design.md)
- [Document 後台註冊、除錯與 media 架構修正](2026-09-06-sprint2-documents-admin.md)
- [curl 實測、DocumentTests、發現並修正權限漏洞](2026-09-18-sprint2-curl-testing-documenttests.md)

### Sprint 3：AI 問答（RAG）
- [AI 問答功能完整記錄：技術選型、實作、除錯、自動化測試](2026-09-18-sprint3-rag-complete.md)
- [.pdf / .docx 格式驗證、AI 幻覺問題、Prompt Engineering](2026-09-18-sprint3-pdf-docx-and-hallucination.md)

### LLM 整合層工程化（T0～T9）✅ 已完成
- [T0～T3：LLM Service、測試隔離、刪除不殘留向量、處理狀態](2026-11-08-llm-hardening-t0-t3.md)
- [T4：暫時性錯誤自動重試、錯誤分類、統一例外](2026-11-11-t4-llm-retry.md)
- [T5：手動重新處理 API、冪等、故意破壞驗證](2026-11-11-t5-reprocess.md)
- [T6：問答時 LLM 失敗回 503、不洩漏細節](2026-11-11-t6-ask-503.md)
- [T7：LLM 呼叫可觀察、結構化 log、不記 prompt](2026-11-13-t7-llm-logging.md)
- [T8：真實 Gemini 整合測試、預設跳過、用真實錯誤驗證 T4](2026-11-13-t8-integration-tests.md)
- [T9：文件與履歷更新 + T0～T9 總結](2026-11-13-t9-docs-and-summary.md)

## 每篇筆記的結構

1. 今天完成的事（含程式碼）
2. 核心概念
3. 待釐清問題
4. 下一步
