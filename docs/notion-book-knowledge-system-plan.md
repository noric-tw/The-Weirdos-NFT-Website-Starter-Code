# Notion 串接書籍知識管理系統：可執行製作評估（強化版）

> 目標：在 **Notion 資料庫** 內完成「新增書籍、筆記、回顧、AI 摘要與問答（RAG）、跨書比較」；不做心智圖與評論。

---

## 1. 需求對應與範圍

### 你已確認的需求
- 系統運作位置：Notion 資料庫
- 儲存位置：雲端（跨裝置同步）
- 必要功能：
  1. 新增書籍
  2. 新增筆記
  3. 回顧功能（複習）
  4. AI 摘要與問答
  5. 同類型、不同書籍觀點比較
- 不需要：心智圖、評論功能

### 建議範圍切分
- **MVP 必做**：Books/Notes/Review 三庫 + RAG 問答 + 今日回顧清單
- **v1 增強**：跨書比較報告 + 自動摘要回填 + 品質監控（命中率/引用率）

---

## 2. 系統架構（Notion-first）

```text
Notion DB (Books/Notes/Review)
    ↕ Notion API
Backend Service (FastAPI / Next.js)
    ├─ Sync Worker (增量同步/去重)
    ├─ Embedding Worker (chunk + 向量化)
    ├─ RAG API (問答/摘要)
    └─ Compare API (跨書觀點比較)
           ↕
    Vector DB (pgvector / Pinecone / Weaviate)
           ↕
      LLM + Embedding Model
```

### 為什麼這樣配
- Notion 保留你熟悉的編輯與資料視圖。
- AI 模組外掛，不綁死在 Notion UI，未來可接 Web/Line Bot。
- 成本可控：先用單一模型與小型向量庫，後續再擴充。

---

## 3. Notion 資料庫設計（可直接建）

## 3.1 Books
| 欄位 | 型別 | 說明 |
|---|---|---|
| Title | Title | 書名 |
| Author | Text | 作者 |
| Category | Multi-select | 類型（商業/心理/科技...） |
| Status | Select | 想讀/在讀/讀完 |
| Start Date | Date | 開始日期 |
| Finish Date | Date | 完讀日期 |
| Source | Select | Kindle/紙本/電子書/其他 |
| Key Takeaways | Rich text | AI 回填總結 |
| Notes | Relation -> Notes | 關聯筆記 |
| Reviews | Relation -> Review | 關聯回顧 |

## 3.2 Notes
| 欄位 | 型別 | 說明 |
|---|---|---|
| Note Title | Title | 筆記標題 |
| Book | Relation -> Books | 所屬書籍 |
| Chapter | Text | 章節 |
| Original Quote | Rich text | 原文節錄 |
| My Insight | Rich text | 個人理解 |
| Tags | Multi-select | 主題標籤 |
| Importance | Select | 高/中/低 |
| Created Time | Created time | 建立時間 |
| Last Edited Time | Last edited time | 更新時間 |
| Embedding Status | Select | pending/done/error |
| Chunk Count | Number | 分塊數 |
| Source URL | URL | 來源連結（可選） |

## 3.3 Review
| 欄位 | 型別 | 說明 |
|---|---|---|
| Title | Title | 例如：原子習慣 D7 |
| Book | Relation -> Books | 回顧書籍 |
| Notes to Review | Relation -> Notes | 要回顧的筆記 |
| Stage | Select | D1/D7/D30/D90 |
| Review Date | Date | 應回顧日期 |
| Completed | Checkbox | 是否完成 |
| Review Summary | Rich text | 當次回顧摘要 |
| Next Review Date | Formula/Date | 下次日期 |
| Score | Number | 吸收程度 1~5 |

---

## 4. 功能落地方式

## 4.1 新增書籍與筆記
- 直接用 Notion Template：`New Book`、`New Note`。
- 若你有 Readwise/Kindle，後續用排程匯入（非 MVP 必做）。

## 4.2 回顧（Spaced Repetition）
- 每新增筆記，系統自動建立 4 筆 Review：D1 / D7 / D30 / D90。
- 每日 07:00 排程抓「Review Date = 今天 且 Completed = false」。
- 在 Notion 建立「今日回顧」View，手機可直接勾選完成。

## 4.3 RAG 問答（根據筆記回答）
1. 增量抓取 Notes（依 `Last Edited Time`）。
2. 將文字切成 300–500 字 chunk（可重疊 50 字）。
3. 產生 embedding 寫入向量庫；metadata 要含：
   - `book_title`, `author`, `category`, `tags`, `note_id`, `updated_at`
4. 問答時先檢索 Top-k（建議 k=6~10），再讓 LLM 僅依檢索內容回答。
5. 回答格式強制附「引用來源」（書名 + Note URL）。

## 4.4 跨書觀點比較
- 輸入：主題詞（例如「長期主義」）+ 類型篩選（例如商業）。
- 檢索多本書相關段落後，輸出固定結構：
  1. 共識（哪些書一致）
  2. 分歧（哪些書觀點相反）
  3. 適用情境（什麼條件下採用哪個觀點）
  4. 你的可執行行動（3 條內）

---

## 5. API 規格建議（MVP）

- `POST /sync/notion`：增量同步 Notion -> 內部儲存
- `POST /embeddings/rebuild`：重建指定書籍/筆記向量
- `POST /ai/ask`：RAG 問答
  - input: `question`, `book_ids?`, `category?`, `tags?`
  - output: `answer`, `citations[]`
- `POST /ai/compare`：跨書比較
  - input: `topic`, `category?`, `book_ids?`
  - output: `consensus[]`, `differences[]`, `actions[]`, `citations[]`
- `POST /review/generate`：為新筆記建立 D1/D7/D30/D90

---

## 6. 推薦技術選型（務實）

### 最快可做組合
- Backend：FastAPI（Python）
- Queue/排程：GitHub Actions cron（先簡單）
- Vector DB：Supabase pgvector（低成本）
- LLM/Embedding：同一供應商先上線，降低整合成本

### 若你偏 JS 生態
- Backend 改 Next.js Route Handlers 也可，架構不變。

---

## 7. 開發里程碑（1~2 週可上線）

### Week 1（MVP）
- 建好三個 Notion DB + Template
- 完成 Notion token 串接與增量同步
- 完成 embedding pipeline + `POST /ai/ask`
- 完成今日回顧 View + 自動建立 D1/D7/D30/D90

### Week 2（v1）
- 完成 `POST /ai/compare`
- 回答顯示 citations（可點回 Notion）
- 增加失敗重試、去重與刪除同步機制

---

## 8. 成本、風險、治理

### 成本
- Notion：沿用既有方案
- Vector DB：小量通常可低月費
- LLM：按 token 計費（問答頻率是主因）

### 風險與對策
1. **筆記結構不一致** -> 用模板強制欄位（Book/Tags/Insight）
2. **回答幻覺** -> 限制「僅依檢索片段回答」+ 必須附引用
3. **資料同步不一致** -> 每晚全量校驗 + 日間增量同步
4. **重複內容過多** -> 以 hash 去重 chunk

### 安全建議
- Notion Integration Token 放在雲端 secret manager。
- API 加最小權限與請求速率限制。

---

## 9. 你現在可以立刻做的 5 件事
1. 在 Notion 建立 `Books / Notes / Review` 三個資料庫與欄位。
2. 先只選一個類別（例如商業）匯入 5 本書做測試。
3. 每本書至少建立 15 則筆記（要有 Tags + My Insight）。
4. 做第一版 RAG 問答，先要求「回答必附 2 則引用」。
5. 再加跨書比較，驗證是否能產出「共識/分歧/行動」。

---

## 10. 結論
你的需求非常適合採用 **Notion-first + 外掛 AI 服務層**：
- 保留 Notion 低摩擦輸入與跨裝置同步。
- 以最小成本先做可用 MVP（新增、回顧、問答）。
- 再逐步升級到跨書比較與品質治理。

若要快速落地，建議你先鎖定：
- 三庫建模（Books/Notes/Review）
- RAG 問答附引用
- 每日回顧清單

做到這三件，系統就已經有實際價值。
