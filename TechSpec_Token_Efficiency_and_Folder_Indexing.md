# TechSpec: Token 節省與資料夾結構索引搜尋設計

## 1. 文件目標

本文件定義 FastCode 在大型程式碼庫問答場景中的兩個核心能力：

1. 為什麼能有效節省 token 成本
2. 如何以資料夾結構（directory structure）為主軸建立索引，達到高品質搜尋

本 TechSpec 同時描述：

- 現行實作（As-Is）
- 工程上可驗收的設計要求（To-Be）

---

## 2. 背景與問題

在 code Q&A / code navigation 場景中，主要成本來自：

- 無差別讀取大量檔案
- 把整個檔案內容直接塞進 prompt
- 缺乏分層檢索，導致每輪都重複搜尋

核心目標是「先用低成本訊號縮小範圍，再投入高成本上下文」，以最大化「每 token 的資訊價值」。

---

## 3. 設計原則

1. Value-first：先拿便宜且高訊號資訊（結構、簽名、摘要），再讀大段程式碼。
2. Layered Retrieval：先 repo/資料夾，再檔案，再 class/function。
3. Budget-aware：每輪必須檢查 token/行數預算與邊際收益。
4. Deterministic Safety：路徑解析、repo 篩選、去重與裁剪應可重現。

---

## 4. 現行系統如何省 token（As-Is）

### 4.1 索引前即減量（Ingestion-time Reduction）

- `fastcode/loader.py` 在掃描階段排除 ignore patterns 與不支援副檔名。
- 超大檔案（預設 > 5MB）直接略過，避免把低價值大檔帶入後續流程。

效果：減少 embedding 與後續檢索候選池體積。

### 4.2 階層化索引代替整檔閱讀（Hierarchical Units）

- `fastcode/indexer.py` 建立 `file/class/function/documentation` 多層元素。
- class/function 具備 `signature/summary/start_line/end_line`，可先做「結構級」判斷。

效果：查詢多數情境可先命中細粒度元素，不需整檔注入。

### 4.3 兩階段檢索縮小搜尋範圍（Repo -> File/Element）

- `fastcode/retriever.py` 先用 repo overview（vector + BM25）選 repo。
- 再在入選 repo 內進行 semantic/BM25 混合檢索與重排。

效果：多 repo 場景可先裁掉不相關倉庫，避免跨 repo 大量誤檢索。

### 4.4 限額與混合排序控制候選量

- `_semantic_search`、`_keyword_search` 使用 top-k 限制。
- 最終 `max_results` 截斷結果數量，避免 context 無上限增長。

效果：控制 prompt 上下文成長速度。

### 4.5 迭代式成本控制（Iterative Agent）

- `fastcode/iterative_agent.py` 使用 `confidence_threshold`、`max_iterations`、`adaptive_line_budget`。
- 以 ROI（confidence gain / lines）與 stagnation 準則決定是否繼續。
- `_smart_prune_elements` 以相關度、來源、粒度、大小做優先級裁剪。

效果：避免「再多看一點」的無限擴張，將檢索收斂在成本可控區間。

### 4.6 Prompt 級 token 預算保護

- `fastcode/answer_generator.py` 計算 prompt tokens。
- 若超過 `max_context_tokens - max_tokens - reserve_tokens_for_response`，則截斷 context。

效果：避免超窗與高額失敗請求，保證回答可生成。

### 4.7 多輪對話摘要降低歷史上下文成本

- 多輪模式將歷史抽成 `<SUMMARY>`，下輪優先使用摘要而非完整歷史文本。

效果：在長對話中維持可用記憶，同時壓低 token 消耗。

---

## 5. 資料夾結構索引如何提升搜尋品質（As-Is）

### 5.1 目錄結構作為第一層語意

- `fastcode/repo_overview.py` 的 `parse_file_structure()` 建立：
  - `directories`
  - `languages`
  - `file_types`
  - `key_files`
- 產生 `structure_text` 與 summary，存入 `VectorStore` 的 repo overviews。

價值：查詢尚未定位到 symbol 前，可先用專案結構做粗定位。

### 5.2 路徑正規化與安全邊界

- `fastcode/path_utils.py` 統一路徑解析、重疊目錄修正、安全檢查。
- `AgentTools` 的 `list_directory/search_codebase/read_file_content` 僅在 repo root 內操作。

價值：避免路徑歧義導致漏檢或誤檢，並確保可重現。

### 5.3 全域 map 讓「名稱 -> 檔案」可解析

- `fastcode/global_index_builder.py` 建立：
  - `file_map`: abs_path -> file_id
  - `module_map`: dotted module -> file_id
  - `export_map`: module -> symbol -> node_id

價值：將檔案結構與符號位置綁定，改善跨檔案追蹤。

### 5.4 Agent 的資料夾導向探索

- 迭代流程中先看 directory tree，再決定 `list_directory/search_codebase`。
- tool call 歷史去重，避免重複探索同一路徑。

價值：以結構化探索替代盲搜，提高命中率與穩定性。

---

## 6. To-Be 規格要求（建議落地）

### 6.1 Token 效率 KPI

必追指標：

1. `retrieved_lines_per_answer`
2. `prompt_tokens_per_answer`
3. `confidence_gain_per_1000_lines`（ROI）
4. `repo_selection_prune_ratio`（進入第二階段前被裁掉 repo 比例）

### 6.2 Folder Index 品質 KPI

1. `directory_level_hit_rate`：查詢是否可先命中正確目錄群
2. `first_pass_file_precision@k`
3. `symbol_resolution_success_rate`（module/export map 命中率）
4. `path_normalization_error_rate`

### 6.3 驗收標準（初版）

- 多 repo 查詢中，第二階段平均 repo 數 <= 5。
- 相同測試集下，與無分層 baseline 比較：
  - prompt tokens 下降 >= 35%
  - 答案定位正確率不下降（或提升）
- 迭代流程中，超過 line budget 的回合必觸發 smart prune。

---

## 7. 風險與對策

1. 風險：過度裁剪導致遺漏關鍵上下文。
   - 對策：保留 top element + 最低 coverage；低信心時允許一輪探索。

2. 風險：路徑格式不一致造成 tool call 無效。
   - 對策：統一走 `PathUtils` 解析與記錄 resolved parameters。

3. 風險：repo overview 品質不穩定影響第一階段選擇。
   - 對策：LLM 選 repo 失敗時回退 embedding+BM25，再回退 user scope。

---

## 8. 實作對應檔案

- `fastcode/loader.py`
- `fastcode/indexer.py`
- `fastcode/repo_overview.py`
- `fastcode/vector_store.py`
- `fastcode/retriever.py`
- `fastcode/iterative_agent.py`
- `fastcode/answer_generator.py`
- `fastcode/path_utils.py`
- `fastcode/global_index_builder.py`

