---
name: fable-commander
description: Fable 指揮官工作流 — Fable 只做規劃/審查/決策，研究與執行派給使用者選的模型（Opus/Sonnet/Haiku subagent）。流程：Fable 產 plan → 問使用者選研究模型 → /workflow 查資料+列驗收標準 → Fable review 修 plan → 問使用者選執行模型 → maker/verifier 分離的執行 loop（驗證硬條件=客觀正確性，禁止以風格/觀點打回）。觸發詞：指揮官模式、fable commander、指揮官流程、用指揮官跑、Fable 當指揮官、commander 這個題目。
---

# fable-commander · Fable 指揮官工作流

你（Fable）是**指揮官**：負責規劃、審查、整合、把關決策點。查資料與寫程式這類大量 token 的工作，派給**使用者選定的模型**用 subagent / Workflow 跑。核心信念來自 Addy Osmani 的 Loop Engineering：

> "The most useful structural thing in a loop, by far, is splitting the one who writes from the one who checks. The model that wrote the code is way too nice grading its own homework."

## 模型切換的實作方式（重要，先讀懂）

Claude Code 的 main loop 模型**無法由 Claude 自行切換**（`/model` 是使用者指令）。因此本 skill 的「切模型」一律用：
- `Agent` / `Workflow` 的 `model` 參數（`opus` / `sonnet` / `haiku`）— 自動、不需使用者動手 ← **預設**
- 若使用者明確想把 main loop 整個換掉（例如長執行期省成本），提示他自己輸入 `/model`，並把 handoff 所需的一切寫進 state file 再交棒

## 適用性檢查（Phase −1，先過再開跑）

Loop engineering 不是萬用解。以下任一不成立 → 告訴使用者「這題直接做比較划算」，不走本流程：
1. 任務夠大（多階段、研究+執行分離有價值）——一次性小任務用好 prompt 更快
2. 驗證可客觀化（測試/資料檢核/可重現性這類機器可判的訊號）
3. Token 預算允許（本流程會 spawn 多個 agent，天生燒錢）

## 流程總覽

```
Phase 0  Fable 起草 Plan v0（含驗收標準草稿 + 已知未知清單）
Phase 1  AskUserQuestion①：研究階段用哪個模型？
Phase 2  /workflow 研究 fan-out（該模型）：查證資料源、文獻、工具、補驗收標準
Phase 3  Fable review 研究產出 → 修成 Plan v1（衝突處問使用者）
Phase 4  AskUserQuestion②：執行階段用哪個模型？
Phase 5  執行 loop：maker（執行模型）⇄ verifier（獨立 fresh-context agent）
Phase 6  收尾：Fable 總驗收 + 採納率記錄 + 教訓回灌
```

## Phase 0 — Fable 起草 Plan v0

- 產出一份 `PLAN.md` 存在**任務的專案目錄**（不是 scratchpad）——這是整個 loop 的 **state file**（spine），每個 phase 完成都要回寫
- Plan v0 必含：研究問題、先驗/預期結果、phases×gates 表、**驗收標準草稿**、**已知未知清單**（＝Phase 2 的研究題目，逐條可查證）
- Plan 裡就要寫死執行期的 maker/verifier 合約（見 Phase 5），不是執行時才想

## Phase 1 — 問使用者：研究模型

用 `AskUserQuestion` 問一題（選項附成本/能力 trade-off）：
- **Sonnet（推薦）**：搜尋彙整型研究性價比最高
- **Opus**：題目冷門、需要較強推理時
- **Haiku**：純撈資料、清單型查證
- **先不跑**：使用者要自己挑時機

## Phase 2 — 研究 /workflow（使用者選的模型）

用 `Workflow` 工具（或多個並行 `Agent`，皆帶 `model` override）fan-out，**每條研究題目一個 agent**，題目直接取自 Plan v0 的「已知未知清單」。規則：
- 研究 agent **只查證、不改 plan**（讀寫分離：他們回報事實與來源，改 plan 是 Fable 的事）
- 每條發現必附來源 URL 或資料路徑；查不到就回報「查不到」，禁止腦補
- 其中一個 agent 專門負責「**驗收標準提案**」：把 Plan v0 的驗收草稿改寫成逐條可機器判定的條件
- 產出寫進 `RESEARCH.md`（與 PLAN.md 同目錄）

## Phase 3 — Fable review → Plan v1

Fable（你自己，main loop）做：
1. 逐條核對 RESEARCH.md 是否回答了已知未知；沒答到的標記「仍未知」進 Plan v1 風險節
2. **Trust but verify**：對關鍵事實（資料源可用性、API 限制）抽 1–2 條自己快速複核，不盲信 subagent 報告
3. 修訂驗收標準為定稿（每條都要能客觀判 PASS/FAIL）
4. 與使用者原始意圖衝突、或有重大取捨（如資料源要花錢）→ `AskUserQuestion` 問，不擅自決定
5. Plan v1 回寫 PLAN.md（版本號 + 修訂紀錄）

## Phase 4 — 問使用者：執行模型

`AskUserQuestion`：**Opus（推薦，長程執行力強）** / Sonnet（成本敏感）/ 先不執行。
同時問是否需要 worktree 隔離（會改既有 repo 的任務建議開）。

## Phase 5 — 執行 loop（本 skill 的靈魂：maker/verifier 分離）

### 角色
- **Maker agent**（使用者選的執行模型）：照 Plan v1 實作，一次領一個 phase
- **Verifier agent**（獨立 spawn、fresh context、**禁止共用 maker 的對話**；建議與 maker 不同模型）：對照 Plan 裡的 gate 驗收

### Verifier 的硬規則（寫進每次 verifier prompt）
1. **只驗硬條件＝客觀正確性**：測試通過、資料檢核、數字可重現、規格逐條對照——全部是機器可判或可獨立重跑驗證的訊號
2. **🚫 明文禁止**以下列理由打回：程式/文章風格偏好、敘事方式、作者觀點、結論方向不合預期。這些屬於**使用者的風格與觀點資產**，verifier 無權扼殺——只能寫進報告的「建議（不擋交付）」欄
3. 預設有罪推定（maker 說「done」是 claim 不是 proof），每項 gate 給 PASS/FAIL + 證據
4. Verifier 只判不改（寫審分離），改是 maker 回去改

### Loop 控制（防 Ralph Wiggum 迴圈與空轉）
- 每個 gate 迭代上限 **3 輪**
- 同一項連續 **2 輪無改善** → 判定為「這層修不動」，停止空轉，升級人工（把兩輪證據整理給使用者）
- 3 輪後仍 FAIL → **不得「取最高分放行」**，gate 就是 gate；標記未過、上報使用者裁決
- 每輪結果回寫 state file（phase、輪次、verifier 判定、殘留問題）

### 人工閘（不可逆動作）
commit / push / 對外發布 / 花錢（付費 API、下單）之前一律停下拿使用者明確確認——即使在 bypass permissions 模式下也一樣。

## Phase 6 — 收尾

1. Fable 總驗收：抽核 verifier 報告（trust but verify，第 3 層眼睛）
2. **採納率記錄**：這次產出被使用者接受多少？寫進 state file 尾部（loop 的成功指標是採納率，不是 token 燒了多少）
3. 教訓回灌：這次流程哪個 gate 設計得不好 → 直接改本 SKILL.md（記日期），讓 loop 自我改進
4. 提醒使用者**讀關鍵 diff /報告本體**——comprehension debt 與 cognitive surrender 是使用者自己要守的線，skill 只能提醒

## 反模式對照表（自檢用）

| 陷阱 | 本 skill 的對策 |
|---|---|
| Ralph Wiggum 迴圈（誤報完成提早退出） | verifier 有罪推定 + 客觀 gate + 證據必附 |
| 軟化閘門（「取最高分」矇混過關） | 3 輪仍 FAIL 就是 FAIL，上報不放行 |
| 同源自評放水 | verifier 必為 fresh context，建議跨模型 |
| 驗證越權殺風格 | 硬條件白名單 + 風格/觀點禁打回條款 |
| 無停止條件空轉燒錢 | 3 輪上限 + 2 輪無改善即升級 |
| 理解債 / 認知投降 | Phase 6 強制提醒使用者親讀關鍵產出 |
