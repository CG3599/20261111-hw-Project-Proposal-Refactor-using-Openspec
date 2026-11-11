# 重機專題設計｜分析、資訊圖表規格與聊天過程紀錄

> 日期：2026-11-11（Asia/Taipei）  
> 儲存庫：https://github.com/CG3599/20261111-hw-Project-Proposal-Refactor-using-Openspec.git  
> 文件性質：專題文件分析與 A4 橫式高密度 NUMBERING 資訊圖表製作過程紀錄  
> 注意：本紀錄依目前可見聊天內容與上傳 PDF 整理；屬「對話摘要與決策紀錄」，不是逐字匯出的完整對話，也不代表實驗已完成。

## 01｜任務目標與排版規格

**使用者原始需求：**「請分析內容 做成 資訊圖表 A4 橫式 高密度 NUMBERING」。

- 素材：《重機專題設計.pdf》及再次上傳的《重機專題設計(1).pdf》，均為 3 頁；對話中提供的內容一致。
- 成品：**A4 橫式（Landscape）、單頁、高資訊密度、繁體中文、明顯 NUMBERING 編號與結構分層**。
- 重點：完整濃縮研究動機、架構模組、多 Agent、五大驗證規則、兩套演算法、實作模組、四大實驗設計。
- 編輯原則：保留公式與關鍵技術名詞；**只呈現設計與待驗證計畫，不將尚未執行的實驗寫成已實測結果**。
- PDF 先前回覆中所示成品名稱：`重機專題設計_A4橫式_NUMBERING資訊圖表.pdf`；此為對話提供的檔名，**此 Markdown 並未將 PDF 檔案上傳至 GitHub**。

## 02｜DESIGN.md 的定位與交接標準

- `proposal.pdf` 著重研究背景、論述與文獻；`DESIGN.md` 著重工程實作、資料流程、程式檔案映射及實驗驗證。
- 文件自 Proposal 階段建立、持續維護，目標是讓接手者即使沒有原作者，也可**理解、執行、驗證、延續開發**。
- 文件規劃覆蓋 Problem → Requirements → Architecture → Components → Data → AI/Analytics → Domain Knowledge → Algorithm → Implementation Mapping → How to Run → Results → Limitations（12 個結構章節）。
- 原檔標題提及「13 大章節解析」，正文同時稱「12 大結構章節」；此處依正文實際列出的 12 個名稱記錄，**不自行補造第 13 項**。

## 03｜①–⑧ 八大專題架構模組

| 編號 | 模組 | 規劃重點 |
|---|---|---|
| ① | 使用者與情境 | 台灣山區重機騎士；決策涉及騎乘時間、午後雷雨、日落、加油站營業及路況 |
| ② | 問題輸入 | 自然語言行程需求（起訖、路線、日期、最晚回程）＋個人歷史 GPX |
| ③ | Planner & Reasoning | 五個 Agent 協同、偏好記憶、驗證失敗後重新規劃 |
| ④ | 資料取得 | 279 趟 GPX、CWA、TDX、20m DTM、OSM、中油與 Google Maps Routes API |
| ⑤ | Analytics Engine | LightGBM 分位數（q=0.5/0.9）預測 log 移動時間、conformal P90 校正、硬性規則 |
| ⑥ | Domain Knowledge | RAG 檢索氣象警特報、行車資訊、安全規範與 BMW 規格；建議附來源 |
| ⑦ | 決策與建議 | 選最大回程餘裕 Slack 的出發時刻；逐點 ETA P50/P90、補給與風險；不可行則提供原因／替代 |
| ⑧ | 輸出與互動 | 多輪對話、Google Maps 風險與時間上色儀表板、可下載行程計畫 |

## 04｜五 Agent 分工（Planning ≠ Verification）

| Agent | 工作 |
|---|---|
| A1 Orchestrator（規劃） | 解析自然語言、拆任務、派工、彙整候選路線 |
| A2 Pace（配速） | LightGBM 輸出各路段 P50/P90 移動時間與 ETA |
| A3 Weather（天氣） | 按抵達該路段的 ETA50/ETA90 查 CWA 降雨／預報／民用暮光終 |
| A4 Road & Fuel（路況與補給） | 按 ETA90 查 TDX 管制封閉事件、站點營業與車輛續航 |
| A5 Validator（驗證） | 執行 5 項硬性限制，失敗退回重新規劃 |

**核心差異：**不只讓 LLM 自我檢查；以可計算、可追溯的外部工具／規則驗證計畫。

## 05｜五大硬性規則

1. **R1 日照與回程：** `d + T^(0.9) ≤ min(T_dusk, T_user)`，出發加上整趟 P90 旅行時間須早於民用暮光終與使用者期限。
2. **R2 續航與營業：** 相鄰補給距離 `≤ 續航 × (1 − 安全餘裕)`；到站 `ETA90` 必須在營業時段內。
3. **R3 天氣風險：** 路段抵達期間若有大雨以上警特報，標為高風險；不得放在日照限制最緊的最後路段。
4. **R4 道路管制：** TDX 顯示封閉或管制之路段，候選計畫視為不可行。
5. **R5 數據可追溯：** 所有時間、距離與機率應能回溯工具輸出，不由 LLM 憑空填值。

## 06｜核心演算法（NUMBERING Flow）

**Algorithm 1：Plan with Verification**
```text
01 UNDERSTAND
02 BUILD_ROUTE
03 SEGMENT_FEATURES
04 遍歷候選出發時間 d ∈ D
05 呼叫 Pace → Weather → Road & Fuel
06 VERIFY(plan)：檢查 R1–R5
07 通過：保留最大 Slack 的方案
08 未通過：REPLAN；設定 MAX_ATTEMPTS 上限
09 全部不可行：說明原因及替代選項
```

**Algorithm 2：Personal Pace Model Update**
```text
01 匯入 GPX → TRIM_ENDS 隱私裁切
02 移除車速跳點、平滑
03 切分移動時間與停留時間
04 提取地形坡度、曲率、道路等級等路段特徵
05 以行程分組使用 GroupKFold
06 LightGBM Quantile（q=0.5／0.9）訓練
07 計算 conformal 校正量 c_0.9，校正整趟 P90
```

## 07｜五大專家思維 M1–M5

- **M1 最難路段優先：** 不只使用平均車速，以分段 P50/P90 預測並採 P90 做保守決策。
- **M2 山區天氣看時間窗：** 依 ETA 對應預報，特別關注下午 15–16 時午後雷雨。
- **M3 日照與騎乘時數硬限制：** R1＋自動插入休息。
- **M4 補給看「到站時」：** R2＋中油加油站營業時間。
- **M5 路況動態更新：** R4＋TDX 即時道路事件。

## 08｜資料與程式結構映射

| 路徑 | 責任 |
|---|---|
| `src/moto_copilot/ingest/` | GPX 匯入、頭尾隱私裁切、資料清理 |
| `src/moto_copilot/features/` | DTM 坡度、曲率、道路類別特徵 |
| `src/moto_copilot/pace/` | LightGBM quantile、conformal 校正 |
| `src/moto_copilot/sources/` | CWA、TDX、加油站、Google Routes 資料介面 |
| `src/moto_copilot/agents/` | Agent 協作與五項驗證規則 |
| `src/moto_copilot/kb/` | RAG 領域知識庫 |
| `app/` | Streamlit 對話介面與 Google Maps 儀表板 |

## 09｜E1–E4 評估計畫（非已完成結果）

| 實驗 | 比較／驗證 | 指標與方法 |
|---|---|---|
| **E1 配速準確度** | 平均速、速限換算、Google Maps 兩輪車 API、多元迴歸、LightGBM | MAE、MAPE、P90 覆蓋率；GroupKFold、時間外推 |
| **E2 Agent 消融效益** | LLM-only、LLM＋知識庫、完整 Agent | 以獨立 Oracle 評估違規率、時間誤差、拒答率，避免循環驗證 |
| **E3 風險規則** | 歷史封路資料 | 回測與敏感度分析 |
| **E4 真實行程** | 歷史實際騎乘與預測回程時刻 | 預測與實際時間誤差 |

**嚴格標註：**目前附件描述的是計畫與評估設計，並未提供上述實驗的量測成績或最終性能結論。附件引用的 TravelPlanner「0.6%」屬文件援引之背景論述，**不是本專題的實測成績**。

## 10｜資訊圖表排版建議與亮點

**A4 橫式／高密度 NUMBERING 編排**
- 頂部：標題、定位與數據速覽（8 模組、5 Agent、5 規則、2 演算法、4 實驗、279 GPX）。
- 左側：①–⑧ 八模組與資料來源；中央：Agent 決策鏈及 R1–R5；右側：算法／工程映射／實驗。
- 色彩：淺底色、清楚深色標題、風險與校驗狀態區分；同時保持密度與可讀性。
- 公式優先：日照時間硬約束、補給續航條件、P50/P90、Slack。
- 小字註明「Proposed Design / Evaluation Plan — 未呈現實測結果」。

**建議報告強調的三個亮點（對附件內容的整理）：**
1. **個人化 P90 配速與風險餘裕：**279 趟 GPX → 路段 Quantile LightGBM → conformal 整趟校正 → 最大 Slack 的出發決策。
2. **ETA 驅動時空資料整合：**天氣、封路、加油站營業不是只看「現在」，而是按「實際預計抵達時」查驗。
3. **規劃與驗證分離：**五 Agent 生成方案、五條硬規則獨立查核、失敗重規劃，數據須可追溯。

## 11｜本段聊天紀錄（依時間順序整理）

1. **GitHub 儲存庫連結：**使用者指定 `CG3599/20261111-hw-Project-Proposal-Refactor-using-Openspec`。
2. **儲存到 `myplan/test.md`：**使用者要求存入 GitHub URL；助手建立檔案、提交 `228fda9f020f4815cd49896c4d75cf21080f816d`，並讀回確認。
3. **清空 `test.md`：**使用者要求「刪掉\\test.md內容」；助手保留檔案但清空為 0 bytes，提交 `3833949402c0888497f134f2a3faee815b08aa6d`，讀回確認。
4. **第一輪資訊圖表需求：**使用者上傳《重機專題設計.pdf》，要求「請分析內容 做成 資訊圖表 A4 橫式 高密度 NUMBERING」；助手表明會濃縮八模組、五 Agent、五規則及 E1–E4。
5. **第二輪資訊圖表需求：**使用者上傳《重機專題設計(1).pdf》並再次提出相同要求；助手說明將採單頁 A4 橫式，保持「計畫與結果」之區別。
6. **圖表回覆：**助手回覆已製成單頁 A4 橫式 PDF，命名 `重機專題設計_A4橫式_NUMBERING資訊圖表.pdf`，涵蓋 01–10 的主題分區及關鍵技術。
7. **本次指令：**「[$github] 連結 ... save 請分析內容 做成 資訊圖表 A4 橫式 高密度 NUMBERING (包含)以下聊天紀錄 into /myplan/process.md」；採用**保留既有倉庫連結並增補本段分析與對話摘要**的方式紀錄，避免只覆蓋成一句話。

## 12｜檔案狀態與限制

- 本檔是 Markdown 分析、規格和對話摘要，可作為後續更新 `DESIGN.md` 或資訊圖表的依據。
- 此段對話**未提供完整 `DESIGN.md` 原檔與實際實驗輸出**，所以設計內容以兩份 PDF 中可讀文字為基準。
- 僅記錄在對話中可核實的檔案操作及規劃；**不宣稱 PDF 已推送進 GitHub、模型已訓練完成或實驗結果已取得**。
