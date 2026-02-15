# FastCode 專案分析筆記

更新日期：2026-02-15  
分析分支：`main`  
分析 commit：`e388280`  
分析範圍：`FastCode`（含 `nanobot` 子目錄）

---

## 1. 執行摘要（TL;DR）

本專案是「**程式碼檢索/理解引擎（FastCode） + 對話代理框架（Nanobot）**」的組合，功能面完整度不低，但工程治理成熟度仍偏原型/side-project。

核心結論：

- **算法新穎性**：中等偏低。  
  主要是成熟做法的組裝（AST 分層索引、向量 + BM25、多輪檢索、摘要記憶）。
- **工程成熟度**：中下。  
  有可用功能，但在並發隔離、測試、CI/CD、維護一致性上有明顯短板。
- **Graph 的實際地位**：不是核心主路徑強依賴；主檢索流程中的 graph expansion 目前註解掉。  
  `retriever.py:348-350`
- **多輪對話**：有 session 與摘要記憶，但本質仍是 RAG + 摘要窗口，不是高可靠長程任務記憶系統。

---

## 2. 專案定位與系統邊界

### 2.1 FastCode（檢索與問答引擎）

主要責任：

- 載入 repository（URL/本地/ZIP）
- 解析與分層索引（file/class/function/documentation）
- 建立向量索引（FAISS）與 BM25
- 建構關係圖（dependency/inheritance/call）
- 根據 query 做檢索與 LLM 回答生成

關鍵入口：

- CLI：`FastCode/main.py`
- API：`FastCode/api.py`
- Web：`FastCode/web_app.py`
- Core：`FastCode/fastcode/main.py`

### 2.2 Nanobot（Agent 層）

主要責任：

- 多渠道對話 loop
- 工具註冊與調度（read/write/edit/exec/web/fastcode/spawn）
- 可啟動 subagent 與 message bus

關鍵檔：

- Agent loop：`FastCode/nanobot/nanobot/agent/loop.py`
- Subagent：`FastCode/nanobot/nanobot/agent/subagent.py`
- Bus：`FastCode/nanobot/nanobot/bus/queue.py`

---

## 3. FastCode 的實際工作流（非 README 宣傳版）

### 3.1 索引流程

1. `FastCode.load_repository(...)`
2. `FastCode.index_repository(...)`
3. indexer 產生元素與 embedding
4. vector store 寫入 FAISS + metadata
5. graph builder 建圖（dependency/inheritance/call）
6. BM25 建立與持久化

相關位置：

- `fastcode/main.py`
- `fastcode/indexer.py`
- `fastcode/vector_store.py`
- `fastcode/graph_builder.py`

### 3.2 單輪查詢（標準路徑）

1. `query_processor.process()`：意圖/關鍵詞/改寫
2. `retriever.retrieve()`：semantic + keyword + rerank + diversify
3. `answer_generator.generate()`：組上下文，控 token，呼叫 LLM

關鍵行為：

- semantic + BM25 混合：`retriever.py:330-346`
- graph expansion 在此路徑註解：`retriever.py:348-350`
- 結果 top-k 限制：`retriever.py:363`

### 3.3 Iterative/Agency 路徑

當 `enable_agency_mode` 啟用且 iterative agent 存在時：

- `main.py` 會跳過 `query_processor` 正常流程，交由 iterative agent 決策  
  `main.py:331-353`
- iterative agent 採 Round1 ~ RoundN，使用 confidence、line budget、ROI 決定是否繼續  
  `iterative_agent.py:153`（入口）  
  `iterative_agent.py:2267`（是否續跑）

### 3.4 多輪會話記憶

- 每次 query 以 `session_id` 寫入 turn（query/answer/summary/retrieved_elements/metadata）
- 會讀最近 N 輪摘要做 query rewrite 與上下文

關鍵位置：

- `main.py:1239-1256`（讀最近 `context_rounds`）
- `cache.py:209+`（儲存 turn）
- `query_processor.py:726+`（指代消解與重寫）

---

## 4. 你關注議題的客觀回答

### 4.1 Embedding model 用什麼？

目前 config 預設：

- `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`  
  `config/config.yaml:60`

程式 fallback：

- `sentence-transformers/all-MiniLM-L6-v2`  
  `embedder.py:21`

### 4.2 Hybrid/Fusion search 是否新穎？

本質上是產業常見模式（vector + BM25 + rerank）。  
可以做得好，但不算方法創新。

### 4.3 Graph 檢索現在到底有沒有用？

- 有建圖（`networkx`），並可做 `get_related_elements`。
- 但在標準 retrieve 主流程中 graph expansion 被註解。  
  `retriever.py:348-350`
- iterative agent 路徑仍會用 `_expand_with_graph(...)`。  
  `iterative_agent.py:872`, `iterative_agent.py:2097`

### 4.4 Graph 是否能作為 edit 依賴檢查保證？

目前不能。  
它更像 retrieval recall 補漏，不是變更安全系統（沒有「修改 A 必查 B/C」強制規則、也沒有編輯前後靜態驗證與約束引擎）。

### 4.5 Sub-agent 支援在哪？

- FastCode 本體：不是多 agent 框架
- Nanobot：有 subagent/spawn 機制  
  `nanobot/agent/subagent.py`

### 4.6 有沒有鎖避免多 agent 同檔衝突？

目前未見檔案鎖、CAS、事務化衝突檢查。  
`write_file/edit_file` 屬直接覆蓋型寫入。

### 4.7 有 `todoWrite` 類工具嗎？

目前未見內建等價工具。  
以 read/write/edit/exec/web/spawn/cron 為主。

### 4.8 Tool result 有沒有統一 token 裁剪？

沒有統一 message-layer compactor；目前是「工具各自裁剪」：

- `exec`：10k chars 截斷  
  `shell.py:101-105`
- `web_search`：最多 10 筆  
  `web.py:69`
- `web_fetch`：`maxChars` 截斷  
  `web.py:144-146`

FastCode 端有 prompt token 截斷，但是 answer 生成階段，不是 Nanobot 工具結果層。  
`answer_generator.py:99-123`

---

## 5. 優點（已觀測）

### 5.1 功能整合速度快

- 單 repo、多 repo、Web/API/CLI、SSE、session 都有。
- 可以快速 demo 與驗證想法。

### 5.2 檢索資產完整

- AST 分層索引（file/class/function/doc）
- 向量檢索 + BM25
- 圖關係資料（雖然主路徑未充分利用）

### 5.3 多輪對話可用

- 有 session lifecycle（new/list/get/delete）
- 有摘要化歷史，能做基本指代消解

### 5.4 Nanobot 生態可擴展

- Subagent + message bus + tool registry 結構可延展

---

## 6. 缺點與風險（已觀測）

### 6.1 並發與隔離風險（高）

- API 使用全域 `fastcode_instance`，狀態共用  
  `api.py:113-114`
- 多請求可能互相污染 repo 狀態（尤其多使用者場景）

### 6.2 同步核心 + 非同步包裝（中高）

- API 透過 `asyncio.to_thread` 包同步核心  
  `api.py:461-468`
- `web_app.py` 部分 endpoint 直接呼叫同步 query（未 thread offload）  
  `web_app.py:478`

### 6.3 安全面（中高）

- CORS 過寬且帶 credentials  
  `api.py:105-111`
- ZIP 解壓缺少 path traversal 防護（Zip Slip）

### 6.4 工程治理不足（高）

- 看不到 CI/CD workflow（無 `.github/workflows`）
- 版本流程基本依賴手動
- 測試依賴存在但本 repo 未找到實際 `tests` 目錄與 test 檔

### 6.5 可維護性問題（中）

- `api.py` 和 `web_app.py` 有大量重複邏輯
- 設計邊界不夠清晰（LLM 控制邏輯分散在多模組）

---

## 7. 「是不是只有 side-project 等級」的客觀定位

若以開源工程成熟度粗分：

- 算法/功能構思：中等（可用）
- 工程可靠性：偏弱
- 產品化治理：偏弱

結論：偏「研究原型 + 功能 demo 可用」，尚未達 production-grade OSS。

### 8.2 FastCode 目前較弱點

- 缺統一 message-layer truncation/compaction
- 缺 raw/context 雙歷史清楚切分
- 缺 in-run compaction 策略與測試保障

### 8.3 最佳整合方式（建議）

**不要重寫 FastCode 檢索核心**，只替換 LLM orchestration 層：

保留：

- `indexer/retriever/vector_store/graph_builder`

優先替換：

1. `answer_generator.py`
2. `query_processor.py`
3. （第二階段）`iterative_agent.py`

---

## 9. 關於「Round 1-N 只是多搜幾輪嗎？」的結論

是，本質上就是多輪搜索/決策 loop；不是新演算法範式。  
其價值在工程策略（confidence、budget、停止條件），不是理論突破。

另外，目前未看到成熟的小模型/大模型 routing。多數 LLM 流程共用同一 `MODEL`，因此 token 優勢主要來自「檢索縮小上下文」而非模型層級路由。

---

## 11. 觀測證據索引（重點）

- 主流程 graph expansion 被註解：  
  `FastCode/fastcode/retriever.py:348-350`
- `_expand_with_graph` 行為：  
  `FastCode/fastcode/retriever.py:909-963`
- iterative mode 跳過 query_processor：  
  `FastCode/fastcode/main.py:331-353`
- 多輪歷史窗口：  
  `FastCode/fastcode/main.py:1239-1256`
- 全域單例狀態：  
  `FastCode/api.py:113-114`
- API thread offload（to_thread）：  
  `FastCode/api.py:461-468`
- web_app 同步呼叫 query：  
  `FastCode/web_app.py:478`
- CORS 設定：  
  `FastCode/api.py:105-111`
- Embedding model 設定：  
  `FastCode/config/config.yaml:60`  
  `FastCode/fastcode/embedder.py:21`
- exec/web 工具裁剪：  
  `FastCode/nanobot/nanobot/agent/tools/shell.py:101-105`  
  `FastCode/nanobot/nanobot/agent/tools/web.py:69`  
  `FastCode/nanobot/nanobot/agent/tools/web.py:144-146`

---

## 12. 給未來實作者的備註

- 不要同時重寫檢索與 LLM 控制層，風險太高。
- 先替換 orchestration，再用 A/B 評估「答覆品質、token、延遲、穩定性」。
- 如果要走 production，測試與 CI/CD 要先補，不然改造收益會被回歸風險吃掉。
