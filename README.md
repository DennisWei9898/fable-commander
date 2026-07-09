<div align="center">

# fable-commander

**Make your top model the commander, not the coder — it plans and reviews; the models you choose do the research and the work.**
**讓最強的模型當指揮官，而不是碼農 —— 它只規劃與審查，研究和執行派給你選的模型。**

A Claude Code skill that turns your session into a commander workflow: plan → delegated research → review → maker/verifier execution loop.
一個 Claude Code skill，把你的 session 變成指揮官工作流：規劃 → 委派研究 → 審查 → maker/verifier 分離的執行迴圈。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](#license)
![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-8A2BE2.svg)
![Rubric](https://img.shields.io/badge/rubric-Loop%20Engineering-orange.svg)
![Pattern](https://img.shields.io/badge/pattern-maker%2Fverifier-blue.svg)

**[English](#english)** · **[繁體中文](#繁體中文)**

</div>

---

<a name="english"></a>

## English

I built this because running everything on the most expensive model is a **double waste**: it burns tokens on grunt work a cheaper model does fine, and — worse — the model that wrote the code is way too nice grading its own homework. The idea comes straight from Addy Osmani's [Loop Engineering](https://addyosmani.com/blog/loop-engineering/):

> "The most useful structural thing in a loop, by far, is splitting the one who writes from the one who checks."

So this skill makes your top model (Fable, or whatever you run the main loop on) act as the **commander**: it drafts the plan, asks *you* which model should do the research, reviews what comes back, asks you which model should execute, and then runs a maker/verifier loop where the checker is always a fresh, independent context.

### One model doing everything vs. the commander workflow

| | One model end-to-end | Commander workflow |
|---|---|---|
| Self-grading | The maker approves its own work → misses slip through | Verifier is a separate fresh-context agent, guilty-until-proven |
| Ralph Wiggum loops | "Done!" declared early, nobody checks | Objective gates + evidence required, 3-round cap, no "best of N" pass |
| Token cost | Top model burns tokens on lookups and boilerplate | Research & execution go to the model *you* pick (Opus/Sonnet/Haiku) |
| Your voice | Reviewer rewrites your style along the way | Verifier checks **hard conditions only** — style and opinions are off-limits |

### The workflow

```
Phase 0  Commander drafts Plan v0 (acceptance criteria + known-unknowns)
Phase 1  Ask the user: which model for research?
Phase 2  Research fan-out on that model (facts + sources only, no plan edits)
Phase 3  Commander reviews → Plan v1 (trust but verify)
Phase 4  Ask the user: which model for execution?
Phase 5  Execution loop: maker (chosen model) ⇄ verifier (fresh context)
Phase 6  Wrap-up: final audit + adoption-rate record + lessons fed back
```

Built-in guardrails: iteration cap of 3 rounds per gate, escalate to human after 2 rounds without improvement, a hard human gate before anything irreversible (commit / push / publish / spend), and an explicit ban on the verifier rejecting work for style or opinion — it can only flag those as non-blocking suggestions.

### Install

```bash
mkdir -p ~/.claude/skills/fable-commander
cp SKILL.md ~/.claude/skills/fable-commander/SKILL.md
```

Then say: **"用指揮官模式跑這個題目"** / **"run this in commander mode"**.

> Note: the skill text is written in Traditional Chinese (the SKILL.md *is* the product — a fully specified workflow). Claude follows it regardless of the language you converse in.

### See it in action

- [`examples/PLAN-sample.md`](examples/PLAN-sample.md) — a real-shaped state file after a full Phase 0–6 run: acceptance criteria, gate-by-gate rounds, adoption rate.
- [`examples/verifier-report-sample.md`](examples/verifier-report-sample.md) — what a verifier report looks like: PASS/FAIL with evidence, plus the "suggestions (non-blocking)" lane that protects your style.

### When *not* to use it

The skill checks this itself (Phase −1): if the task is small, the verification can't be made objective, or the token budget is tight — it will tell you to just do the task directly. Loop engineering is not a universal answer.

### Advisor mode: a lighter alternative

Claude Code also ships an experimental `/advisor <model>` command: your main model runs cheap, and Claude automatically consults a stronger advisor at key decision points (before committing to an approach, on repeated errors, before declaring done). It's a good fit for exactly the tasks Phase −1 already tells you to skip the full workflow for.

It's not a substitute for the commander workflow, though — the advisor gives a second opinion to the *same* executor, which usually just follows it; there's no independent verifier, no objective gate, no stop condition. It also has two known rough edges worth knowing before you rely on it: (1) if your session (or any subagent inside it) has loaded a deferred tool via `ToolSearch` — which stock Claude Code routinely does — advisor calls afterward deterministically fail ([GitHub #73923](https://github.com/anthropics/claude-code/issues/73923), closed as not planned); (2) with a frontier model as the main model, very long transcripts (~100K+ tokens) can silently disable the advisor too ([GitHub #67609](https://github.com/anthropics/claude-code/issues/67609)).

See the "Advisor 模式" section in [`SKILL.md`](SKILL.md) for full guidance, including the trap of running this skill's verifier while an advisor is still attached — it erodes the independence the whole workflow depends on.

### Related

Sister project: [loop-engineering-reviewer](https://github.com/DennisWei9898/loop-engineering-reviewer) — run your loop with the commander, then audit the loop itself against the seven Loop Engineering rules.

### Credit

The structural beliefs (maker/verifier split, objective gates, stop points, adoption rate as the success metric) come from Addy Osmani's [Loop Engineering](https://addyosmani.com/blog/loop-engineering/). This skill is one opinionated way to put them to work inside Claude Code.

---

<a name="繁體中文"></a>

## 繁體中文

做這個 skill 的起點是一個挫折：把所有事都丟給最貴的模型，是**雙重浪費** —— 查資料、寫樣板這種粗活燒它的 token 很不划算；更糟的是，寫程式的模型自己驗收自己，永遠改作業改得太仁慈。理念直接來自 Addy Osmani 的《[Loop Engineering](https://addyosmani.com/blog/loop-engineering/)》：

> 「迴圈裡最有用的結構性設計，就是把『寫的人』和『查的人』分開。」

所以這個 skill 讓你的最強模型（Fable，或任何跑 main loop 的模型）只當**指揮官**：起草計畫 → 問**你**研究要派哪個模型 → 審查研究產出 → 問你執行要派哪個模型 → 跑 maker/verifier 分離的執行迴圈，驗收永遠由獨立 fresh-context 的 agent 來做。

### 一個模型做到底 vs 指揮官工作流

| | 一個模型從頭做到尾 | 指揮官工作流 |
|---|---|---|
| 自評放水 | maker 驗收自己的作業 → 漏洞放行 | verifier 獨立 fresh context，預設有罪推定 |
| Ralph Wiggum 迴圈 | 提早喊「做完了」沒人查 | 客觀 gate + 證據必附，3 輪上限，禁止「取最高分」放行 |
| Token 成本 | 最貴的模型燒在查資料和粗活上 | 研究與執行派給**你選的**模型（Opus/Sonnet/Haiku） |
| 你的風格 | 審查順手把你的文風改掉 | verifier **只驗硬條件**——風格與觀點明文禁止打回 |

### 流程總覽

```
Phase 0  指揮官起草 Plan v0（驗收標準 + 已知未知清單）
Phase 1  問使用者：研究階段用哪個模型？
Phase 2  研究 fan-out（該模型）：只回報事實與來源，不改 plan
Phase 3  指揮官 review → Plan v1（trust but verify）
Phase 4  問使用者：執行階段用哪個模型？
Phase 5  執行迴圈：maker（執行模型）⇄ verifier（獨立 fresh context）
Phase 6  收尾：總驗收 + 採納率記錄 + 教訓回灌
```

內建護欄：每個 gate 迭代上限 3 輪、連續 2 輪無改善就升級人工、不可逆動作（commit / push / 發布 / 花錢）前一律過人工閘、明文禁止 verifier 以風格或觀點理由打回——那些只能寫進「建議（不擋交付）」欄。

### 安裝

```bash
mkdir -p ~/.claude/skills/fable-commander
cp SKILL.md ~/.claude/skills/fable-commander/SKILL.md
```

之後說：**「用指揮官模式跑這個題目」**即可觸發。

### 實際長什麼樣

- [`examples/PLAN-sample.md`](examples/PLAN-sample.md) —— 一份跑完 Phase 0–6 的範例 state file：驗收標準、逐 gate 輪次紀錄、採納率。
- [`examples/verifier-report-sample.md`](examples/verifier-report-sample.md) —— verifier 報告長相：PASS/FAIL + 證據，以及保護你風格的「建議（不擋交付）」欄。

### 什麼時候不該用

skill 自己會先檢查（Phase −1）：任務太小、驗證無法客觀化、或 token 預算緊——它會直接告訴你「這題直接做比較划算」。Loop engineering 不是萬用解。

### Advisor 模式：更輕量的替代方案

Claude Code 另外還有一個實驗性的 `/advisor <model>` 指令：主模型跑便宜的，Claude 在關鍵決策點（提交方案前、錯誤重複、宣告完成前）自動諮詢更強的 advisor。這正好適合 Phase −1 判定「不值得跑完整流程」的那些任務。

但它取代不了指揮官工作流——advisor 是給同一個執行者的第二意見，執行者通常照做；沒有獨立 verifier、沒有客觀 gate、沒有停止條件。而且有兩個值得先知道的已知坑：(1) 如果你的 session（或任何 subagent）曾用 `ToolSearch` 載入過 deferred 工具——標準 Claude Code 常態性會這樣做——之後的 advisor 呼叫會確定性失敗（[GitHub #73923](https://github.com/anthropics/claude-code/issues/73923)，Anthropic 已標記不打算修）；(2) 用 frontier 模型當主模型時，很長的對話（約 100K+ tokens）也可能讓 advisor 悄悄失效（[GitHub #67609](https://github.com/anthropics/claude-code/issues/67609)）。

完整說明見 [`SKILL.md`](SKILL.md) 的「Advisor 模式」節，包含「開著 advisor 跑本 skill 的 verifier」這個陷阱——它會侵蝕整個工作流賴以運作的獨立性。

### 姊妹作

[loop-engineering-reviewer](https://github.com/DennisWei9898/loop-engineering-reviewer) —— 用 commander 跑迴圈，再用 reviewer 拿七條 Loop Engineering 量尺回頭體檢迴圈本身。

### 致謝

結構信念（maker/verifier 分離、客觀 gate、停止點、採納率作為成功指標）全部來自 Addy Osmani 的《[Loop Engineering](https://addyosmani.com/blog/loop-engineering/)》。本 skill 只是把它落地成 Claude Code 裡一套有立場的工作流。

### 授權

MIT —— 自由使用、修改、分享。

---

## Contact · 合作聯絡

Open to collaboration on AI agent workflows.
對 AI agent 工作流的合作提案持開放態度，歡迎聯繫。

📧 [dennis.xd.wei@gmail.com](mailto:dennis.xd.wei@gmail.com) · 💼 [LinkedIn](https://www.linkedin.com/in/dennis-wei-47393a14a/)

---

<div align="center">
<sub>Structural beliefs · 結構信念來源：Addy Osmani <a href="https://addyosmani.com/blog/loop-engineering/">Loop Engineering</a></sub>
</div>
