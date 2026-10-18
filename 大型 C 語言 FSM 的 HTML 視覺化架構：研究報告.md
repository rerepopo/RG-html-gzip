# 大型 C 語言 FSM 的 HTML 視覺化架構：研究報告

> 研究日期：2026-10-13。證據標示：**【產品】** 實際產品或官方文件；**【研究】** 學術文獻；**【建議】** 我依你的資料特性做的設計推論（尚未用你的真實資料驗證）。

## 結論摘要

1. 超過數十個 State 時，「全部畫在一張圖」不該是預設入口。它適合當「局部」或「按需展開」的檢視，而不是第一眼畫面。
2. 你的資料特性（多數 State 只有 1 條出邊、少數 2 條、極少 3 條）代表圖**非常稀疏**。這是優勢：多數 State 落在「線性鏈」上，真正需要被看見的只有少數分支、匯合與迴圈。可以做出**語意不變**的縮減（結構骨架 + 可展開的鏈）。
3. 推薦架構：以「**搜尋 → 上下文 → 按需展開**」的焦點探索為核心，搭配三層導覽（系統地圖 → FSM 骨架 → 焦點視圖），並讓**表格**與**程式碼面板**永遠與圖連動。
4. 正確性來自「資料層與視圖層分離」：所有原始 State / Transition 以穩定 ID 保存，視圖只是可還原的投影；任何聚合都要揭露；**路徑追蹤一律在原始圖上計算**，不在折疊後的圖上算。
5. 技術上，圖形函式庫不是瓶頸（採用此設計後，每次渲染大約只有數十個節點）。建議 ELK.js（Worker 內做 layered 佈局）搭配 React Flow 或自訂 SVG 渲染，圖演算法自己寫；Graphviz-WASM 留作靜態匯出。需要實測的項目列在第七部分。
6. **誠實聲明**：我沒有找到「100+ State FSM 的公認最佳視覺化」，也沒有找到專門針對它的對照實驗。以下方案是把相近領域（程式碼探索工具、statechart 工具、大型圖視覺化研究）的證據組合起來，再依你的資料特性做的推論。

## 研究範圍與限制

- 查閱了產品文件（Stateflow、Sourcetrail、Stately/XState、Cytoscape.js 擴充、React Flow、ELK、Graphviz-WASM）、人機互動與圖視覺化文獻（Cockburn 等人、van Ham 與 Perer、Ghoniem 等人、Moody、UML statechart 理解實驗），以及「從 C 程式碼取出 FSM」的文獻。
- 限制：(1) 沒有取得 ELK、Cytoscape、React Flow 的中立效能基準，部分效能說法來自文件或單一來源，已標註；(2) 多數學術來源我讀到的是摘要與投影片，不是全文；(3) Sourcetrail 已停止維護，只能當設計參考；(4) 函式庫版本請在實作時查最新穩定版並固定版本號。

---

## 第一部分：業界現況

### 1.1 【產品】有實際產品或官方文件支持的做法

**MathWorks Stateflow：作者宣告的階層 + 子圖當「不透明方塊」**

- 官方文件說明可把低層細節收進 subchart，顯示成不透明方塊，且不改變 chart 行為；雙擊進入；可用 supertransition 跨層連線；編輯器提供在子圖階層間前進/後退的導覽按鈕；Model Explorer 以樹狀列出階層。
- 文件也建議：狀態巢狀超過三層時，用 subchart 降低複雜度。
- 啟示：階層之所以安全，是因為它是**作者在模型裡宣告的語意**（composite state）。你的 C 程式是平面的，工具「推導」出來的分群沒有這個保證，見第四部分。

**Sourcetrail（2021 年停止維護）：Search + Graph + Code 三個連動視圖**

- 官方文件：搜尋欄找符號；Graph 顯示「目前選取的符號與和它有關聯的符號」；Code 視圖列出選取符號的所有原始碼位置；以 Clang 為 C/C++ 建索引。
- 啟示：這是最接近你情境（不熟悉程式碼的人要探索程式）的成熟設計。它**不畫全圖**，而是「選取 → 看鄰域 → 對照程式碼」。專案已封存，不宜當依賴，但設計值得借鏡。

**Stately Studio / XState Visualizer：面向「作者撰寫的中小型 machine」**

- 功能含平移縮放、點事件模擬、命令面板；Studio 的 Discover 頁可依「10 個 state 以下」篩選。我沒有找到證據顯示它針對 100+ State 的逆向工程狀態機設計。
- 啟示：「點事件走一步」的模擬很有價值，但那是作者視角；你的使用者是讀者視角。

**Cytoscape.js 生態：compound node + expand/collapse**

- Cytoscape.js 論文稱可在一般硬體上流暢顯示數千個圖元素；expand-collapse 擴充提供折疊節點並產生 meta edge；complexity-management 擴充提供 filter / hide / collapse。
- 啟示：折疊產生的 meta edge 是「聚合邊」，它的語意風險（第四部分）要自己處理，函式庫不會替你保證。

**Graphviz（WASM 版 viz-js）：靜態佈局引擎**

- 可在瀏覽器以 WebAssembly 產生 SVG，但輸出是靜態圖，互動要自己做。有 issue 與競品作者部落格回報數百節點 DAG 在 dot 下偏慢，這些來源不中立，只供參考。

### 1.2 【研究】研究中的方法與證據

- **搜尋、顯示上下文、按需展開**（van Ham 與 Perer, IEEE InfoVis 2009）：論點是許多情境下「全圖概覽」並非使用者主要任務；改為讓使用者從感興趣的節點出發，用 degree-of-interest 取出相關子圖，再往想看的方向展開。該研究用 300 萬筆法律引用資料驗證，**不是 FSM 資料**，但互動模型直接適用。
- **Overview+Detail / Zooming / Focus+Context / Cue-based**（Cockburn、Karlson、Bederson, ACM Computing Surveys）：把「聚焦與脈絡」的做法分成空間分離、時間分離、整合式（如 fisheye）與線索式（高亮或淡化）四類。
- **Physics of Notations**（Moody, 2009）：提出「複雜度管理」（模組化、階層；經驗法則為每張圖約 7±2 個元素）與「認知整合」（用摘要圖提供全局視野、用導覽線索連接多張圖）。我讀到的是課程投影片整理，非原文。
- **Node-link 與 Matrix**：Ghoniem 等人發現圖超過約 20 個頂點時，矩陣在多數任務上優於 node-link，**唯獨路徑尋找一貫偏向 node-link**；較新的大規模實驗（Okoe 等人）則發現 node-link 在拓樸類任務上常較準確。重點：Ghoniem 測試的邊密度為 0.2～0.6，而你的 FSM 密度遠低於此（見第二部分），結論不能直接套用。
- **階層不是萬靈丹**（Manso、Morasca、Piattini 等人，UML statechart 實驗家族）：多個受控實驗**未能顯示** composite state 能明確提升 statechart 的可理解性。作者提醒結果受受試者經驗與圖規模限制，但這代表「加階層就會更好懂」不能被當作前提，必須驗證。
- **佈局品質**（DTU 碩士論文，眼動實驗）：好的佈局提升表現、降低心智負荷，線交叉的理解成本偏高。這是論文層級的證據，權重較低。

### 1.3 【研究】從 C 程式碼取出 FSM 本身並不完美

- Grosche 等人（2020）對工業級車用嵌入式 C 程式測試：預期的狀態機實作中，**74% 被完整辨識，8% 部分辨識**。他們的做法也強調模型與原始碼之間的細粒度追溯。
- 其他相關：FSMExtractor（CCS 2019，靜態分析，160 個程式評估）、StatRec（IBM，從程序式碼回復 statechart 並主張語意等價）、以符號執行（KLEE）取出狀態轉移表。
- 啟示：解析**一定會有漏抓與不確定**；視覺化若只呈現乾淨的圖，會製造不該有的確定感。這是第四部分「可信度標示」的依據。

### 1.4 對「問題 A」的直接回答

| 問題 | 回答 | 證據 |
| --- | --- | --- |
| 超過 100 個 State 還適合傳統 State Diagram 嗎？ | 不適合當預設入口；適合數十個節點內的局部檢視 | 【研究】Moody 複雜度原則；【建議】 |
| 有哪些替代方式？ | 焦點/鄰域探索、狀態轉移表、骨架 + 鏈壓縮、分層地圖、矩陣（僅限 FSM 間耦合） | 【產品】Sourcetrail、Stateflow；【研究】van Ham 與 Perer |
| 如何避免 Spaghetti？ | 靠「少畫」而非「畫得更美」：限制同時顯示的元素、長邊改用出口標記、共同目標改用徽章 | 【建議】 |
| 業界是否把 Overview / Navigation / Detail / Code 分開？ | Sourcetrail 與 Stateflow 都是如此。我沒有找到涵蓋整個業界的統計，不宜說「業界一致」 | 【產品】 |
| 值得參考的實例 | Sourcetrail（三視圖設計）、Stateflow（階層與導覽）、Cytoscape expand-collapse（機制）、van Ham 與 Perer（互動模型） | 如上 |

---

## 第二部分：問題本質

為什麼超過 100 個 State 時，單張 State Diagram 會難以理解？我認為主要因素如下，每項附上對策方向。

**1. 元素數量超過感知與工作記憶負荷【研究】。** Moody 以 7±2 作為每張圖的經驗上限；也有 UML 理解研究的回顧提到每張圖約 20～60 個元素的建議。這些都是經驗法則，不是硬門檻，但量級說明：150 個節點加 180 條邊遠超過一張圖能承載的範圍。對策：每個畫面只放「當下任務需要」的元素。

**2. 圖其實很稀疏，難讀的原因不是密度。【建議】** 以 150 個 State、平均出度約 1.2 估算，邊約 180 條，密度（邊數 / V(V-1)）約 0.008，遠低於 Ghoniem 實驗用的 0.2～0.6。真正的難讀來源是：

- **回邊（迴圈）**：分層佈局必須把迴圈邊反向，造成長距離邊橫跨許多層；
- **共同目標**：例如大量 State 的錯誤處理都回到 IDLE，產生高扇入的放射狀長邊；
- **長鏈**：一長串「1 進 1 出」的節點占滿畫面，卻幾乎沒有資訊量。

對策：針對這三種結構各自處理（出口標記、徽章、鏈壓縮），而不是套用通用的「大圖」技巧。

**3. 任務是局部的，圖是全域的。【研究】** 你列的六項需求中，搜尋、追蹤、對照程式碼、檢查轉移邏輯都是局部任務；只有「理解整體架構」需要全域視野，而且只需要**結構骨架**，不需要每個節點。van Ham 與 Perer 的論點正好對應這種情況。

**4. 平面 FSM 沒有作者階層。【產品對照】** Stateflow 的階層是作者宣告的語意；C 的平面 switch-case 沒有。自動分群若用「包含框」呈現，會暗示不存在的語意（見第四部分）。

**5. 佈局不穩定破壞心智地圖。【研究】** 折疊或展開後重新佈局，節點位置大幅跳動，使用者會失去方向。大型圖視覺化文獻把「保持心智地圖」列為關鍵議題。對策：切換檢視時盡量保留已有節點的相對位置，並以動畫過渡。

**6. 轉移條件文字塞不進邊。【建議】** 100 多條邊的條件字串無法同時可讀。對策：邊上只放短標籤，完整條件在側欄與程式碼面板按需顯示。

**7. 多 FSM 混在一起。【建議】** 「FSM 內部結構」與「FSM 之間的互動」是兩個層級的圖。畫在一起會讓兩者互相干擾。對策：系統層級單獨成圖，FSM 內部另外一層。

**8. 整齊的圖掩蓋解析不確定性。【研究】** 見 1.3，解析本身不是 100% 完整。

---

## 第三部分：方案比較

### 3.1 五種視覺化架構

**方案 A：改良版全圖（全圖 + 縮放 + 折疊）**

- 做法：沿用傳統 State Diagram，加上縮放、小地圖、compound node 折疊。
- 優點：與既有圖最接近，使用者無學習成本；實作最直接。
- 缺點：入口仍是大圖；折疊後 meta edge 有語意風險；展開/折疊後重新佈局會跳動。
- 適用：約 30～40 個 State 以內；或作為大型 FSM 的「Full」備援檢視。

**方案 B：焦點探索（Search → Context → Expand）**

- 做法：使用者先搜尋或選取一個 State，畫面只顯示它的 k 跳前驅與後繼，再按方向展開（Sourcetrail、van Ham 與 Perer 的模型）。
- 優點：認知負荷最低；與規模無關（150 或 1000 個 State 的畫面大小相同）；直接對應六項需求中的局部任務。
- 缺點：缺乏全貌，新手不知從哪裡開始；需要設計「入口」與探索歷程紀錄。
- 適用：任務導向的追蹤與除錯。

**方案 C：表格中心（狀態轉移表 / 大綱 + 連動小圖）**

- 做法：以可排序、篩選、搜尋的 State 表與 Transition 表為主，點選後在旁邊顯示局部小圖。
- 優點：最精確、零佈局成本、容易對照程式碼、可列印。
- 缺點：結構（迴圈、分支）不容易一眼看出；路徑追蹤要靠逐步跳轉。
- 適用：精確核對與 Debug；也是所有圖形方案的必要搭檔。

**方案 D：分層地圖（系統地圖 → FSM 骨架 → 焦點視圖）**

- 做法：第 0 層是 FSM 之間的地圖；第 1 層是單一 FSM 的「結構骨架」（分支、匯合、迴圈、介面 State，鏈壓縮成可展開的一條邊）；第 2 層是焦點視圖。每層之間用離散層級切換，而不是連續語意縮放。
- 優點：同時提供全貌與細節；天然對應 6～9 個 FSM；骨架可由演算法精確、可還原地產生。
- 缺點：需要骨架與分群演算法、揭露機制與驗證，工程量最大。
- 適用：你的完整目標。

**方案 E：矩陣 / DSM**

- 做法：用矩陣呈現 FSM × FSM（或 State × State）的連結。
- 優點：高密度耦合時沒有邊交叉。
- 缺點：路徑追蹤是 node-link 的強項（Ghoniem）；State × State 矩陣對稀疏 FSM 幾乎全空；使用者需要學習。
- 適用：**僅用於 FSM × FSM 的耦合概覽**（6～9 個 FSM 的矩陣很小，格子內可放連結數量）。

### 3.2 你列出的十種模式，在這個專案的適用性

| 模式 | 適用性 | 備註 |
| --- | --- | --- |
| Hierarchical Graph | 有條件使用 | 僅當階層來自**程式碼事實**（檔案、函式、真實巢狀）；啟發式分群不可用包含框表達 |
| Progressive Disclosure | 必用 | 全貌 → 骨架 → 鄰域 → 程式碼，逐層揭露 |
| Semantic Zoom | 謹慎使用 | 縮放改變資訊層級容易讓人迷路。建議用**離散層級 + 明確的層級標示**取代連續縮放 |
| Collapsible Subgraph | 部分使用 | 只用於 SCC、鏈、程式碼事實群組；必須揭露 meta edge |
| Overview + Detail | 必用 | 系統地圖 + 小地圖 + 詳細面板 |
| Focus + Context | 必用（以線索式為主） | 用高亮與淡化，**避免幾何扭曲**（fisheye 會破壞佈局穩定與可讀性，此為我的判斷） |
| State Search / Filtering | 必用 | 名稱、事件、條件文字、結構類型（分支/匯合/錯誤/未解析） |
| Transition Path Highlighting | 必用 | 前向、後向、A→B 路徑 |
| State-centric Exploration | 核心 | 以 State 為中心展開 |
| Graph + Table Hybrid | 必用 | 表格是精確性的保證 |

### 3.3 其他值得考慮的方法（皆為【建議】）

- **出口標記（off-page connector / edge stub）**：把長距離邊畫成一小段箭頭加目標名稱，點擊跳轉。每條邊仍存在、兩端仍可辨識，屬於純呈現。
- **鏈壓縮**：把「中間節點入度 = 1 且出度 = 1」的線性鏈顯示為一條可展開的邊。
- **SCC 縮合圖**：強連通分量縮合後必為有向無環圖，是**圖論事實**而非語意猜測，能呈現「主要階段 + 迴圈區域」。
- **共同目標徽章**：高入度目標（如 IDLE）的回邊改以徽章顯示計數與清單，而不畫線。
- **入口提示（Start here）**：初始 State、主要路徑、解析問題、不可達 State。
- **側車註解檔（sidecar annotation）**：人工撰寫的 State 說明放在獨立檔案，絕不覆寫解析出的事實。
- **快照比對**：不同 commit 的 FSM 差異檢視（後續版本）。

---

## 第四部分：最佳推薦方案

### 4.1 推薦：「骨架 + 焦點探索」

以方案 D（分層地圖）為骨幹，方案 B（焦點探索）與方案 C（表格）常駐搭配。這是【建議】，由下列理由組成：

- **對應你的六項需求**：理解全貌 → 骨架；定位 State → 搜尋；追蹤轉移 → 焦點視圖；對照 C 程式碼 → 程式碼面板；檢查邏輯 → 表格 + 診斷。
- **與規模無關**：每個畫面的元素數量固定在可讀範圍，150 或 500 個 State 的畫面大小相近。
- **利用你的資料特性**：出度多為 1 的稀疏圖，最適合「鏈壓縮 + 骨架」，而且這種壓縮可以由演算法精確檢查、完整還原。
- **正確性可驗證**：原始資料不動，視圖是可還原的投影（見 4.5）。
- **實作風險可分階段控制**：表格 + 搜尋 + 程式碼面板語意風險最低、價值最高，可以先做；圖形的聚合功能後做。

為什麼不只選 A：入口仍是大圖，沒有解決認知負荷。為什麼不只選 B：新手缺少全貌與入口。為什麼不只選 C：看不出結構。

### 4.2 情境 1：單一 FSM 含 100 個以上 State

1. **入口**：State 數小於約 30 時直接顯示全圖；超過時預設顯示「骨架」。這個門檻是可調參數，請用你的真實資料與第七部分的實驗決定。
2. **骨架規則（語意不變）**：保留節點 = 初始 State、終點（出度 0）、分支（出度 ≥ 2）、匯合（入度 ≥ 2）、跨 FSM 介面、解析不確定的 State、被搜尋命中或被釘選的 State。其餘「入度 = 1 且出度 = 1」的連續節點收成一條「鏈邊」：`=[ n states ]=>`，點開可就地展開成 n 個節點或垂直清單。壓縮條件由演算法檢查，不靠命名猜測，所以可精確還原。
3. **骨架大小要先量測**：骨架比例取決於你的真實資料，我無法預先保證。請先對真實 FSM 算出「骨架節點數 / 總節點數」。若骨架仍超過約 60 個節點，再疊加 SCC 縮合或「程式碼事實群組」（檔案、函式）折疊。
4. **共同目標與回邊**：高入度目標（例如 IDLE）的入邊改成徽章「→ IDLE ×37 \[清單\]」；回邊以出口標記取代長線。
5. **焦點視圖**：選取任一 State，顯示 k 跳前驅與後繼（k 可調，預設 2），其餘淡化；可逐步展開。
6. **側欄**：左側為可篩選的 State 清單（依結構類型：入口 / 分支 / 匯合 / 鏈 / 錯誤 / 未解析），右側為 Inspector；清單與圖雙向連動。

### 4.3 情境 2：6～9 個 FSM，合計 100 個以上 State

- **第 0 層：系統地圖**。節點是 FSM（FSM 是真實邊界，可以使用包含框）；邊是跨 FSM 耦合，以聚合邊顯示數量；實線 = 已確認，虛線 = 推論。可切換成 FSM × FSM 矩陣（格內放耦合數）。旁邊是 FSM 表格：State 數、Transition 數、解析率、未解析數。
- **進入某個 FSM**：回到情境 1 的做法。
- **跨 FSM 耦合的來源**：若來自明確的函式呼叫、事件傳遞或共享變數的寫入與讀取，就標示來源位置；凡是由資料流推論得到的耦合，預設標為 `inferred`。
- **跨 FSM 路徑追蹤**：追蹤到 FSM 邊界時預設停下並提示，由使用者決定是否跨越，因為跨越多半依賴推論。
- **全域搜尋**：結果以「FSM.State」列出，點選直接跳到該 FSM 的焦點視圖。

### 4.4 回答你的四個關鍵問題

**Q1. 第一次開啟、完全不熟悉 C 程式碼，先呈現什麼？**

1. 一行摘要與數字：N 個 FSM / M 個 State / T 條 Transition / 解析覆蓋率 / 未解析數。
2. 多 FSM 時是系統地圖，單一 FSM 時是骨架。
3. 「Start here」：初始 State、主要路徑（標示為「推導」）、解析問題、不可達 State。
4. 圖例（實線 / 虛線 / 點線、鏈邊、徽章各代表什麼）。

理由：先給全局摘要，再讓使用者往下鑽（Moody 的認知整合概念）；**不一開始就畫 150 個節點**。

**Q2. 點選某個 State 應該看到什麼？（依重要性排序）**

1. 名稱、所屬 FSM、類型徽章（entry / branch / join / hub / error / unresolved）、可信度。
2. 意義：程式碼註解、側車註解檔的人工說明（兩者分開標示）。
3. 入邊與出邊表：對象、事件或條件、求值順序、可信度、行號。
4. 程式碼片段：該 case 區塊，轉移行高亮。
5. 診斷：隱含 stay（沒有任何條件成立時留在原狀態）、沒有 else、未解析的賦值。
6. 跨 FSM 關聯。

**Q3. 如何往前或往後追蹤？**

- 在 State 上有「後繼」「前驅」（每按一次擴展一跳）、「向前追蹤 depth n」「向後追蹤 depth n」、「找 A → B 路徑」（再選第二個 State）。
- 結果以路徑清單呈現（最短優先，顯示「x / y 條」，上限可調），圖上只高亮路徑、淡化其他；遇到迴圈標記「已訪問」。
- 有上一步 / 下一步歷程，狀態寫進 URL hash 以便分享。
- 每條邊的條件逐一列出，**不合併、不做可行性判斷**（那需要符號執行，屬後續工作）。

**Q4. 某個 FSM 有 150 個 State，如何既看得到整體又不被淹沒？** 同時使用：骨架（整體結構）+ 左側 150 項可篩選清單（全部都找得到）+ 小地圖 + 焦點視圖（目前關心的區域）+ 出口標記與徽章（處理長邊與共同目標）。使用者同時看到的是「約數十個骨架節點 + 一個焦點鄰域」，不是 150 個節點。

### 4.5 問題 C：兼顧視覺化與正確性

**(a) 簡化的語意分級**

| 等級 | 例子 | 是否改變語意 | 規則 |
| --- | --- | --- | --- |
| L0 純呈現 | 佈局、顏色、縮放、淡化；出口標記（兩端仍可辨識）；邊標籤截斷（全文在側欄） | 否 | 可預設啟用 |
| L1 可還原聚合 | 鏈壓縮；SCC 或程式碼事實群組折疊；共同目標徽章；平行邊合併（×2） | 不改變，但可能誤導 | 預設啟用，但必須揭露：顯示被隱藏的數量與清單、一鍵展開，且不得用來回答可達性 |
| L2 可能改變語意 | 把啟發式分群當階層；傳遞約簡（移除「多餘」邊）；合併相似 State；把 meta edge 當成「A 可到 B」；自動猜測缺失的轉移；把推論邊畫成實線 | 是 | 預設禁止；若提供，必須標為「建議」並與事實視覺上區隔 |

**(b) 哪些資訊可以隱藏、但必須可還原**

- 可隱藏：鏈中節點、共同目標的入邊、邊上的完整條件、程式碼細節、解析正常的診斷。每一項都有「展開 / 清單」入口與數量。
- 不得隱藏：未解析與推論標記、孤立或不可達 State 的存在（可以摺疊，但計數永遠顯示）、隱含 stay。

**(c) 保留所有原始 State 與 Transition**

- 雙層模型：規範層保存 State 與 Transition（穩定 ID）；視圖層的每個元素記錄 `represents: [ids]`。
- 可自動測試的不變量：(1) 每個原始 Transition 在任一視圖中恰好被一個視圖元素表示（直接邊、鏈邊成員、meta edge 成員、徽章成員或隱藏清單）；(2) 全部展開後節點數與邊數等於原始；(3) 路徑與可達性搜尋只讀規範層。
- 穩定 ID 由「FSM 名 + 狀態常數」與「轉移來源位置」組成，避免重新解析後 ID 漂移；重新命名的處理列為後續工作。

**(d) 轉移條件與觸發事件**

- 資料欄位：trigger（事件、輸入、旗標）、guard（原文 + 正規化文字）、actions、order（C 的 if / else if 求值順序即優先序）、隱含 stay。
- 顯示：邊標籤只放短標籤（事件或截斷條件，約 20 字內）；完整原文與求值順序在側欄；多出邊的 State 在 Inspector 依求值順序列出。「無條件」字樣只在解析器確定時使用，否則顯示「條件未解析」。

**(e) 從圖形節點定位到 C 程式碼**

- 每個 State、Transition、guard 都帶 `loc: {file, startLine, endLine, startCol, endCol}` 與快照識別（commit hash 或檔案雜湊）。
- 程式碼面板於建置時預先嵌入相關片段並語法高亮；點邊時高亮 `state = X` 賦值行與 guard 行；提供「複製 path:line」，以及依團隊工具而定的「在編輯器開啟」連結（例如 VS Code 的 `vscode://file/` URL scheme，使用前請先確認團隊環境）。
- HTML 與目前程式碼版本不同時，顯示快照識別與產生時間，避免對到錯的行。

**(f) 辨識解析不完整、推論不確定或無法確認的轉移**

- `confidence` 三級：`confirmed`（直接樣式，例如 case 內對 state 變數賦常數）；`inferred`（經巨集展開、常數傳遞、函式回傳、跨函式賦值）；`unresolved`（賦值來源是變數、函式指標或查表，目標未知）。
- 視覺：實線 / 虛線 / 點線加「?」；`unresolved` 指向獨立的「Unknown target」佔位節點；篩選器可一鍵列出所有 inferred 與 unresolved。
- 覆蓋率儀表：已解析 case 數 / 總 case 數、未解析寫入點數、不可達 State、無出邊的非終點 State；系統地圖上每個 FSM 顯示解析率。
- 原則：寧可顯示「不知道」，也不要靜默漏掉。Grosche 等人的結果（74% 完整、8% 部分）顯示漏掉是常態。

**(g) 避免自動分群暗示不存在的階層**

1. 分群依來源分三類並使用不同視覺語言：(事實) 來自程式碼的檔案、函式、enum 區段註解、真實巢狀 → 實線框 + 來源說明；(圖論事實) SCC、鏈 → 專屬圖示並註明「由轉移關係計算」；(啟發式) 名稱前綴、社群偵測 → 淡色光暈 + 「建議分組」徽章，**不使用包含框**，預設關閉。
2. 包含框只保留給**真實邊界**（FSM 本身、程式碼宣告的巢狀）。
3. 折疊後的 meta edge 必須標示真實端點（A → 群組內 x；群組內 y → B），不得顯示成 A → B；hover 時列出底層轉移。**路徑與可達性永遠在原始圖上計算**。這是聚合最常見的陷阱：群組有入邊也有出邊，並不代表入口走得到出口。
4. 佈局造成的假象：分層佈局的「由上而下」不代表時間順序；回邊要明確標示，避免讀者把視覺方向當成語意。

---

## 第五部分：具體 UI 設計

以下為純文字 Wireframe，**State 名稱與數字皆為示意**，UI 文字以英文呈現以維持對齊。

### 5.1 系統總覽畫面（多 FSM）

左側是系統地圖，右側是 FSM 表與入口提示，底部是覆蓋率。重點：第一眼看到的是 FSM 之間的關係與解析品質，不是 State。

```
+--------------------------------------------------------------------------+
| Search states/FSMs/events (/): [_______________]  ecu-fw @a1b2c3d        |
+--------------------------------------------+-----------------------------+
| SYSTEM MAP (L0)             [Map|Matrix]   | FSM        St  Tr  Parsed   |
|                                            | PowerMgr   18  21  100%     |
|  +----------+  12 links  +----------+      | CommLink   42  51   97%     |
|  | PowerMgr |----------->| CommLink |      | Diag       27  30   88% !   |
|  |  18 st   |<-----------|  42 st   |      | Storage    11  12  100%     |
|  +-----+----+   3 links  +-----+----+      | ...                         |
|        | 5                     | 9         |                             |
|        v                       v           | START HERE                  |
|  +----------+             +----------+     | > Entry: PowerMgr.INIT      |
|  |   Diag   |             | Storage  |     | > Cross-FSM links: 29       |
|  |  27 st   |             |  11 st   |     | > Parse issues: 14  [view]  |
|  +----------+             +----------+     | > Unreachable states: 3     |
|                                            |                             |
|  solid = confirmed  dashed = inferred      | Sort/filter the table to    |
|  click FSM = open  |  hover = preview      | find an FSM, then open it.  |
+--------------------------------------------------------------------------+
| Coverage: 4 FSM / 98 st / 114 tr | 14 unresolved | (sample numbers)      |
+--------------------------------------------------------------------------+
```

### 5.2 單一 FSM 畫面（150 個 State 的骨架視圖）

左：可篩選清單（150 項全都找得到）；中：骨架（`=[ n states ]=>` 是可展開的鏈，`hub` 是共同目標徽章）；右：Inspector 與圖例；下：程式碼面板（預設收合）。

```
+--------------------------------------------------------------------------+
| < System / CommLink | [Search: ________] View:[Skeleton|Full|Table] k=[2]|
+------------------+------------------------------------+------------------+
| STATES (150)     | SKELETON  (38 of 150 shown)        | INSPECTOR        |
| [____filter___]  |                                    |                  |
| [x] entry        |        (INIT)                      | (no selection)   |
| [x] branch       |          |                         |                  |
| [x] join         |     =[ 7 states ]=>                | Click a state or |
| [ ] chain        |          v                         | a =[ n ]=> run   |
| [ ] error        |      (CHK_LINK)--fail-->(ERROR)    | to inspect it.   |
| [ ] unresolved   |          | ok                      |                  |
| -------------    |     =[ 12 states ]=>               | LEGEND           |
| INIT             |          v                         | ( ) state        |
| CHK_LINK  *      |      (WAIT_ACK)--timeout-->(RETRY) | =[n]=> chain of n|
| SEND_SYN         |          | ack                 |   |       states     |
| WAIT_ACK         |          v                 back to | ---   confirmed  |
| RETRY            |     (ESTABLISHED)         WAIT_ACK | - -   inferred   |
| ESTABLISHED      |   hub: 37 edges -> INIT  [list]    | ...   unresolved |
| ...              |                                    | ->INIT x37 hub   |
| (list <-> graph  | minimap [ .::. ]  fit | 1:1        | badge            |
|  stay in sync)   |                                    |                  |
+------------------+------------------------------------+------------------+
| CODE (collapsed)  comm_fsm.c:212-248   [open in editor] [copy path:line] |
+--------------------------------------------------------------------------+
```

### 5.3 單一 State 詳細資訊

注意三件事：入邊與出邊分欄；出邊依 C 的求值順序編號，並明列「隱含 stay」；每條轉移連到具體行號。

```
+--------------------------------------------------------------------------+
| STATE  CommLink.WAIT_ACK   [branch] [in-loop] [confirmed]                |
| enum 0x14 | defined comm_fsm.c:57 | handler comm_fsm.c:212-248           |
+--------------------------------------------------------------------------+
| Meaning (code comment): 'Waiting for ACK after SYN'                      |
| Your note (sidecar file): (none)  [+ add note]                           |
+------------------------------------+-------------------------------------+
| INCOMING (3)                       | OUTGOING (2)  checked in order      |
| from       via        conf.        | # to      when          conf.       |
| SEND_SYN   always     conf.        | 1 ESTAB   rx_ack==1     conf.       |
| RETRY      timeout    conf.        | 2 RETRY   t>TMO&&n<3    conf.       |
| INIT       reset(hub) infer.       | - (stay)  none true     implicit    |
+--------------------------------------------------------------------------+
| CODE comm_fsm.c       [open in editor] [copy path:line]                  |
| 212  case ST_WAIT_ACK:                                                   |
| 213    if (rx_ack == 1) {                                                |
| 214      state = ST_ESTAB;         <-- transition #1 (L214)              |
| 215    } else if (t > TMO && n < 3) {                                    |
| 216      state = ST_RETRY;         <-- transition #2 (L216)              |
| ...                                                                      |
+--------------------------------------------------------------------------+
| DIAGNOSTICS: 1 note - no else branch: implicit 'stay' (self-loop)        |
+--------------------------------------------------------------------------+
```

### 5.4 Transition 路徑追蹤畫面

左側是路徑清單（最短優先，可 hover 預覽），右側圖只顯示追蹤結果加一跳上下文，其餘 141 個 State 淡化隱藏。底部列出逐邊條件，並明講「未合併、未做可行性判斷」。

```
+--------------------------------------------------------------------------+
| TRACE from [CHK_LINK] [Forward|Backward|Path A->B] to [ESTABLISHED]      |
| depth [4]  stop at: [x] FSM boundary [ ] error states   history < 3/5 >  |
+------------------------------+-------------------------------------------+
| PATHS (3 found, 3 shown)     | GRAPH (trace + 1-hop context)             |
| #1 length 4 (shortest)       |                                           |
|   CHK_LINK > SEND_SYN >      |  (CHK_LINK)--ok-->(SEND_SYN)              |
|   WAIT_ACK > ESTAB           |      |                  |                 |
| #2 length 7 (via RETRY)      |    fail            (WAIT_ACK)             |
|   ...                        |      v              |      \\              |
| #3 length 9 (via RETRY x2)   |   (ERROR)         ack    timeout          |
|   ...                        |                     v        v            |
|                              |                 (ESTAB)   (RETRY)~~       |
| hover path = highlight       |                                           |
| click state = inspect        |  dimmed: 141 other states hidden          |
+------------------------------+-------------------------------------------+
| Path #1 edge conditions: ok -> always -> rx_ack==1     [copy] [export]   |
| Note: conditions listed per edge, NOT combined (no feasibility check).   |
+--------------------------------------------------------------------------+
```

### 5.5 互動規則摘要

- `/` 聚焦搜尋；方向鍵或 `j` / `k` 在清單與候選間移動；`Enter` 設為焦點；`Esc` 清除焦點。
- `f` 向前追蹤、`b` 向後追蹤、`p` 選第二個 State 找路徑；`Alt+←` / `Alt+→` 歷程。
- 切換視圖（骨架 / 全圖 / 表格）時，保留已存在節點的相對位置並用動畫過渡，降低心智地圖被破壞的程度。
- 所有狀態（FSM、焦點 State、追蹤條件、視圖模式）寫入 URL hash，可分享、可重現。

---

## 第六部分：實作建議

### 6.1 整體管線

```
C source --> Parser / Extractor --> FSM IR (JSON, canonical)
                                        |
                                        +--> Analysis: validate, SCC, chains,
                                        |      hubs, reachability, metrics
                                        v
                                  View-model builder
                          (system map / skeleton / focus)
                                        |
                                        v
                       Layout (ELK, Web Worker, cached) --> HTML UI
                                        ^
      code snippets + source map + snapshot id embedded in the HTML
```

原則：**規範層（IR）只由解析器寫入，UI 只讀**；視圖層從規範層導出，每個視圖元素都記錄它代表哪些原始 ID。

### 6.2 C 程式碼解析

- **兩條路徑**：(1) Clang AST（libclang）：精確，可展開巨集與型別，需要 compile\_commands.json 與可編譯的環境；Sourcetrail 與多數研究使用 Clang 前端。(2) tree-sitter-c：不需編譯設定、對壞掉的建置環境較有韌性，但只到語法層、不展開巨集。這個取捨為【建議】。
- **建議**：能取得編譯參數時以 Clang 為主，tree-sitter 作備援；由巨集展開或條件編譯得來的轉移標為 `inferred`。
- **要辨識的實作樣式**：switch(state) 的 case、if-else 鏈、表驅動（轉移表陣列）、函式指標表（每個 State 一個 handler）、巨集包裝。表驅動最容易精確；函式指標表常導致 `unresolved`。
- **State 變數的辨識**：建議讓使用者在設定檔指定（state 變數、enum 型別、FSM 入口函式），工具再驗證。KLEE 相關研究也是假設使用者提供 state variable；全自動辨識（如 FSMExtractor）可列為選項，但誤判必須揭露。
- **輸出必附**：原始碼位置、可信度、解析診斷。

### 6.3 FSM 資料模型（IR）

```ts
interface FsmIR {
  schema: 'fsm-ir/1';
  snapshot: { commit: string; generatedAt: string };
  fsms: Fsm[];
  couplings: Coupling[];     // cross-FSM links
  diagnostics: Diagnostic[];
}
interface Fsm {
  id: string; stateVar?: string; entry?: StateId;
  states: State[]; transitions: Transition[];
  implicitStay: { state: StateId; reason: string }[];
}
interface State {
  id: StateId;           // stable, e.g. 'CommLink.ST_WAIT_ACK'
  name: string; value?: string;
  doc?: string;          // from code comments only
  loc: SourceRange;      // definition (enum / case label)
  handler?: SourceRange; // the case block / function
}
interface Transition {
  id: string;            // stable
  from: StateId; to: StateId | 'UNKNOWN';
  trigger?: string;      // event / input / flag
  guard?: { raw: string; normalized?: string };
  actions?: string[];
  order?: number;        // evaluation order in an if/else chain
  confidence: 'confirmed' | 'inferred' | 'unresolved';
  loc: SourceRange;      // the assignment statement
}
interface Coupling {
  from: string; to: string;  // FSM ids
  kind: 'call' | 'event' | 'shared_var';
  confidence: 'confirmed' | 'inferred';
  evidence: SourceRange[];
}
```

人工說明放在獨立的 sidecar 檔（YAML 或 JSON），只疊加顯示，不寫回 IR。

### 6.4 分析層（建置時）

- **驗證**：孤立 State、不可達 State、無出邊的非終點、重複轉移、同一 case 內多重賦值。
- **結構**：SCC（Tarjan）、鏈偵測（入度 = 1 且出度 = 1，且不是入口 / 介面 / 不確定）、分支與匯合、回邊（DFS）、共同目標（入度達使用者設定的門檻或分位數）。
- **路徑**：在規範層用 BFS 與 DFS。簡單路徑數量可能指數成長，所以先給最短路徑，再列 k 條，並明示「已顯示 x 條，另有更多」，附上限與逾時保護。
- **預先計算**：骨架、SCC、鏈、共同目標在建置時算好；焦點與路徑查詢（圖很小）在瀏覽器即時算。

### 6.5 佈局

- 骨架與焦點視圖使用 ELK 的 layered 演算法。理由：它支援 port、邊標籤、compound 與各種約束，且 elkjs 可在 Web Worker 中執行，不會阻塞介面（依 elkjs 文件）。
- 我沒有找到中立的 ELK 效能基準。由於這個設計下每次佈局的節點數預期只有數十到約一百，延遲應可接受，但**請以第七部分的基準測試確認**。
- 穩定性：快取每個視圖的座標；展開或收合時固定已存在節點的位置，只對新增部分佈局。ELK 提供與互動式佈局相關的策略，採用前請對照官方 reference 驗證能否滿足需求。
- 邊路由：使用 ELK 輸出的 bend points 自行繪製邊，邊標籤位置也取自 ELK。

### 6.6 圖形與渲染技術比較

評估的前提：採用 4.x 的設計後，**每次渲染只有數十到約一百個節點**，所以渲染效能不是主要決策因素；決定性的是節點內容的豐富度、邊路由與標籤、對聚合語意的控制，以及維護成本。

| 技術 | 角色與優點 | 缺點與限制 | 維護成本 |
| --- | --- | --- | --- |
| ELK.js | 專職佈局（layered 等）；支援 port、邊標籤、compound；可在 Worker 執行 | 不負責渲染；JSON 格式與選項較多，學習曲線陡 | 中（選項多，需讀 reference） |
| Cytoscape.js | 完整圖模型 + canvas 渲染；內建鄰居、路徑等圖查詢；有 expand-collapse 等擴充；論文稱可流暢顯示數千元素 | 節點內容較難做得豐富；canvas 文字與無障礙較弱；與 ELK 的邊路由整合程度需自行確認 | 低到中 |
| React Flow | DOM 節點，可做資訊豐富的 State 卡片與自訂邊；內建平移縮放、小地圖、選取 | 無內建自動佈局（需外接 ELK）；官方效能指南強調要 memoize、避免在元件內讀取整個 nodes 陣列、可用 onlyRenderVisibleElements；React 與建置依賴 | 中 |
| D3.js | 自由度最高，適合客製視圖（矩陣、鏈 ribbon、小地圖） | 互動、選取、無障礙都要自己做 | 高 |
| Graphviz（viz-js WASM） | 成熟的靜態佈局；可在瀏覽器產生 SVG | 無內建互動；有 issue 與非中立部落格回報數百節點 DAG 偏慢；對聚合與互動沒有幫助 | 低（但能力有限） |
| 其他 | Sigma.js 等 WebGL 方案針對數千節點以上，本設計不需要；yFiles 等商用方案本次未查證，不作評論 |  |  |

### 6.7 我的選擇與理由

- **圖邏輯自行實作**：模型、SCC、鏈、焦點展開（DOI）、路徑搜尋用純 TypeScript，與任何渲染函式庫解耦，才能完全控制聚合語意並寫單元測試。
- **佈局**：ELK.js，放在 Web Worker。
- **渲染**：React Flow（自訂 State 卡片與自訂邊元件，使用 ELK 的 bend points）；若團隊不想引入 React，就改用自訂 SVG + d3-zoom。
- **靜態匯出與基準對照**：Graphviz-WASM，或直接輸出 SVG。
- **替代方案**：若你想最快做出可用原型，且可接受節點內容較簡單，Cytoscape.js 是合理選擇。
- **不是因為熱門**：我的理由是 (a) 需要豐富的 State 卡片；(b) 需要 ELK 的邊路由與標籤；(c) 需要完全控制聚合語意；(d) 能建置成單一 HTML 檔（建置工具請自行評估，本次未查證特定外掛）。
- 以上是【建議】，在第七部分的基準測試前請視為假設。

### 6.8 分階段路線

1. **Phase 0 量測**：量真實 FSM 的出度分布、鏈比例、SCC 大小、共同目標；建立 3 組基準資料（10 / 150 / 300 State，另加多 FSM）與合成資料。
2. **Phase 1**：IR + 表格 + 搜尋 + 程式碼面板 + 診斷與覆蓋率（價值最高、語意風險最低）。
3. **Phase 2**：焦點視圖 + 路徑追蹤。
4. **Phase 3**：骨架 + 鏈壓縮 + 徽章 + 出口標記。
5. **Phase 4**：系統地圖 + 矩陣 + 跨 FSM 耦合。
6. **Phase 5**：驗證實驗（第七部分），依結果調整門檻與預設值。

每個階段結束都要跑 4.5(c) 的不變量測試。

---

## 第七部分：驗證方法

目標：比較不同 UI 是否真的提升工程師的**理解效率與正確性**，而不是看起來比較漂亮。

### 7.1 實驗設計

- **受試者**：不熟悉該程式碼的工程師。採組內設計（每人使用多種 UI），以拉丁方平衡 UI 與資料集的順序，降低學習效應。先 pilot 4～6 人調整任務，再決定正式人數；我不預設人數或效果量。
- **條件**：B0 = 現有完整展開的 Diagram（基準線）；B1 = 只有表格 + 程式碼（無圖）；B2 = 靜態 Graphviz SVG（可選）；T = 本方案。也可做消融（拿掉骨架、拿掉揭露機制）來看各機制的貢獻。
- **資料**：小型（少於 10）、中大型（150 與 300）、多 FSM（6～9 個，合計超過 100）。每組資料都加入**植入缺陷**（故意改一條轉移的目標、移除一條轉移、加一個不可達 State、加一個 unresolved 賦值）作為正確性檢測。
- **任務**（每項都有標準答案）：
  - T1 定位：找出處理某條件的 State，並指出程式碼行。
  - T2 鄰域：列出某 State 的所有前驅、後繼與條件。
  - T3 路徑：A 能否到達 B？列出路徑。要包含「答案是不能」的題目，並刻意設計一題「群組有入邊也有出邊，但實際走不通」，專門測試聚合造成的假象。
  - T4 反查：哪些 State 可以走到 ERROR？
  - T5 檢查：找出植入的缺陷。
  - T6 全貌：看 5 分鐘後用自己的話描述系統架構（FSM 數量、主要流程、跨 FSM 互動），由不知道條件的評分者對照標準答案打分。
  - T7 不確定性：指出哪些轉移是推論或未解析的。

### 7.2 指標

| 面向 | 指標 | 說明 |
| --- | --- | --- |
| 效率 | 任務完成時間；首次正確答案時間；到「能描述全貌」的時間 | 以中位數與分佈比較，不只看平均 |
| 正確性 | 任務正確率；錯誤類型；植入缺陷的偵測率與誤報率 | **本專案最關鍵**，尤其是「聚合造成的錯誤信念」比例 |
| 校準 | 自信度評分與實際正確率的關係 | 檢視是否因圖太整齊而過度自信 |
| 負荷 | NASA-TLX；SUS | 主觀感受 |
| 行為 | 點擊與互動次數、回溯次數、平移縮放次數、搜尋使用率、程式碼面板開啟率 | 看哪些機制真的被使用 |
| 理解品質 | T6 全貌題的評分 | 概念層面的理解 |
| 可追溯性 | 從圖跳到程式碼的成功率與耗時 | 對照原始碼的能力 |
| 眼動（可選） | 注視分布、回看次數 | 判斷是否迷失 |

### 7.3 不需要人的保真度測試

- **不變量**：每個原始 Transition 恰好被表示一次；全部展開等價於原圖；路徑查詢結果與直接在 IR 上計算一致。
- **解析品質**：用已知答案的 C 程式（含 switch、表驅動、函式指標、巨集）計算轉移的 precision 與 recall，並把 `unresolved` 的比例與「漏抓」的比例分開報告。
- **佈局指標（輔助，不作為主要結論）**：邊交叉數、邊總長、彎折數、重新佈局的位移量（穩定性）。這些與理解效率沒有保證的對應，只當輔助。
- **效能**：用 10 / 150 / 300 / 600 個節點、出度 1～2 的合成圖，量測佈局耗時、互動延遲（輸入到畫面更新）與記憶體。這同時回答第六部分尚未確認的 ELK 與渲染層效能問題。

### 7.4 上線後遙測（可選）

記錄搜尋到找到 State 的時間、追蹤深度分佈、程式碼面板使用率、未解析徽章被點擊的次數，用來判斷骨架門檻與預設值是否合理。

### 7.5 判準（事先定義，避免事後解釋）

方案 T 要同時滿足：任務正確率（尤其 T3、T5）不低於基準；並且在完成時間或 T6 全貌分數上明顯較好。若「聚合造成的錯誤信念」比例高於基準，視為失敗，應回頭修正揭露機制。具體的顯著性門檻請在 pilot 後再訂。

---

## 待實測與尚未確認事項

- ELK 在你的資料上的佈局耗時，以及展開時的位置穩定程度（我沒有找到中立基準）。
- 骨架壓縮比與骨架節點數（取決於真實資料）。
- 含「全域重置到 IDLE」這類共同目標邊時，SCC 可能整個合成一大塊；必須先用共同目標徽章處理，再算 SCC。
- 解析器在你的 C 程式樣式（switch、表驅動、函式指標）上的覆蓋率。
- 團隊編輯器的 deep link 慣例。
- 各函式庫的最新穩定版：實作時查詢並鎖版。
- 「composite state 不一定提升可理解性」的實驗證據來自小型 UML 圖與學生或有限的受試者，是否適用於你的情境，也需要在你的實驗中驗證。

---

## 參考資料

**產品與官方文件**

- MathWorks, Stateflow subcharts: https://www.mathworks.com/help/stateflow/ug/using-subcharts-to-encapsulate-modal-logic.html
- MathWorks, Refactor Stateflow charts using the API: https://www.mathworks.com/help/stateflow/api/refactor-stateflowcharts-using-the-api.html
- Sourcetrail documentation (community fork copy): https://github.com/petermost/Sourcetrail/blob/master/DOCUMENTATION.md
- Sourcetrail, discontinued in 2021: https://en.wikipedia.org/wiki/Sourcetrail ; archived repository: https://github.com/coatisoftware/sourcetrail
- Stately Studio docs: https://stately.ai/docs/studio ; XState Visualizer: https://xstate.js.org/docs/visualizer
- Cytoscape.js expand-collapse: https://github.com/iVis-at-Bilkent/cytoscape.js-expand-collapse ; complexity-management: https://github.com/iVis-at-Bilkent/cytoscape.js-complexity-management
- Cytoscape.js paper: https://pmc.ncbi.nlm.nih.gov/articles/PMC4708103
- elkjs README (Web Worker usage): https://classic.yarnpkg.com/en/package/elkjs
- React Flow performance guide: https://reactflow.dev/learn/advanced-use/performance
- Viz.js API (Graphviz WASM): https://viz-js.com/api/ ; Graphviz issue on slow dot layout: https://gitlab.com/graphviz/graphviz/-/issues/2343 ; non-neutral blog on DAG layout speed: https://sia.hackernoon.com/heres-why-you-shouldnt-use-graphviz-for-dags-what-you-should-use-instead

**人機互動與視覺化研究**

- van Ham & Perer, Search, Show Context, Expand on Demand (IEEE InfoVis 2009): https://dig.cmu.edu/publications/2009-doigraphs.html
- Cockburn, Karlson, Bederson, A Review of Overview+Detail, Zooming, and Focus+Context Interfaces (ACM Computing Surveys): https://www.microsoft.com/en-us/research/wp-content/uploads/2008/01/cockburn-ComputingSurveys09.pdf
- Ghoniem, Fekete, Castagliola, node-link vs matrix readability: https://hal.archives-ouvertes.fr/hal-00343819
- Okoe et al., Node-Link or Adjacency Matrices: Old Question, New Insights: https://arxiv.org/pdf/1709.00293.pdf
- Moody, Physics of Notations (secondary: course slides): https://cs.uwaterloo.ca/%7Ejmatlee/Teaching/CS846/Schedule/Jan30/Kevin.pdf
- Manso, Morasca, Piattini et al., composite states in UML statecharts: https://neverworkintheory.org/2011/08/17/three-empirical-evaluations-of-uml.html ; https://irinsubria.uninsubria.it/handle/11383/1675846
- Baltsen & Christoffersen, layout quality and UML comprehension (DTU thesis): https://www2.imm.dtu.dk/pubdb/pubs/6813-full.html

**從 C 程式碼取出 FSM**

- Grosche, Igel, Spinczyk, Transformation- and Pattern-based State Machine Mining from Embedded C Code (ICSOFT 2020): https://www.scitepress.org/Papers/2020/97698
- Chen et al., Automated finite state machine extraction (FSMExtractor, 2019): https://pure.psu.edu/en/publications/automated-finite-state-machine-extraction/
- Abadi et al., StatRec (IBM Research): https://research.ibm.com/publications/automatic-recovery-of-statecharts-from-procedural-code
- Yoshida et al., Extracting a Micro State Transition Table Using KLEE: https://srg.doc.ic.ac.uk/klee21/talks/Yoshida-MSTT.pdf
