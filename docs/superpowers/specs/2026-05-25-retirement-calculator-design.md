# 退休规划计算器 · 设计文档（v1）

**日期：** 2026-05-25（最后更新：实施完成后整理）
**作者：** Yang Yang
**参考：**

- Great Eastern Retirement Calculator (https://www.greateasternlife.com/sg/en/personal-insurance/our-products/retirement-income/retirement.html)
- `Proposal_retire_invest.xlsx` · worksheet `UT-FI-10`（Capital Drawdown 模型递推校验源）

---

## 1. 目的与目标用户

构建一份**顾问演示工具**，由 Yang Yang（GEFA 高级财务顾问）在客户会议中使用。相比 GE 原版，本工具有四方面深入：

1. **退休前 / 退休后费率分离**——pre-retirement 与 post-retirement 各自一组 return + inflation 假设
2. **可选 Waiting Period**——属于 pre-retirement 子阶段，在此期间客户**停止继续供款**，本金仍按 pre 费率继续复利
3. **goal-seeking 出 lump sum 与每月所需储蓄**——并把"通胀如何吃掉购买力"显式拆给客户看（Expense Through Time）
4. **两种取款策略 side-by-side 对比**：Capital Preservation（保本、级别股息）vs. Capital Drawdown（耗用、抗通胀），取款率均可实时拖动

**目标用户：** Yang Yang（顾问）在面对客户演示。客户不直接操作，但 UI 与解释需足够清晰让客户跟上节奏。

---

## 2. 范围（v1）

### ✅ 包含

- 3 步向导（Step 1 个人信息 / Step 2 假设 + 结果 / Step 3 策略对比）
- Pre-retirement + 可选 Waiting Period（共用 pre 费率） + Post-retirement 三段
- 3 档生活方式预设卡片（GE 原数：S$2,300 / S$2,900 / S$5,200）+ 自定义月开销输入
- Goal-Seeking 计算：客户期望开销 → 反推 lump sum + 月投资额
- Chapter 2 三组 KPI 区：
  - Expense Through Time（通胀显示）
  - Funding Gap（总需求 / 已有储蓄复利后 / 净缺口）
  - Saving Plan（月投资额 / 投资年期）
- Asset Balance + Cashflow 双图（Chapter 2 内）
- Capital Preservation + Capital Drawdown 策略对比卡 + 双图（Chapter 3）
- Yield rate 与 Withdrawal rate 实时滑块（Chapter 3）
- 浏览器原生「Print → Save as PDF」（5 页 A4 完整 report，所有图表正确缩放，详见 §8.5）
- **UI 语言：英文 + 简体中文双语并排**（每个英文文案下方紧跟弱化色阶的中文译文，详见 §9.4）

### ⏸ 不包含（留给后续版本）

- 新加坡本地政策模块（CPF LIFE / SRS / MediSave / CareShield Life）
- 多收入流叠加（年金 / 租金 / 股息混合）
- Projection 模式（"我这样供能到多少"——v1 只做 Goal-Seeking）
- 多情景并行对比（保守 / 中性 / 激进同屏）
- 客户数据保存 / 多版本历史比较
- 中英双语**切换** UI（v1 是同时并排，不是切换）

---

## 3. 整体流程

```text
[ Top stepper: 01 · 02 · 03 ]

Step 1: Profile           →    Step 2: Assumptions & Outcome    →    Step 3: Strategies
  · Profile                      · Phase I  Pre-Retirement              · Live rate sliders
    (gender, ages,                  (return, inflation)                   (CP yield / CD w-rate)
     savings)                     · Phase II Waiting Period             · CP vs CD comparison
  · Desired lifestyle              (toggle + years only;                  cards
    (3 presets + custom              uses Phase I rates)                · Capital Balance chart
     monthly expense)             · Phase III Post-Retirement            (CD trajectory)
                                    (return, inflation)                 · Annual Withdrawal chart
                                  · Outcome · Expense Through Time       (CP level vs CD growing)
                                    (3 cards: today / @ret / @end)      · Talking points
                                  · Outcome · Funding Gap                · Print/PDF
                                    (3 cards: total / ES proj / gap)
                                  · Outcome · Saving Plan
                                    (2 cards: monthly / duration)
                                  · Asset Balance chart
                                  · Cashflow chart
                                    (in [green] / out [red])
[ ← Back | Next → ]            [ ← Back | Next → ]                  [ ← Back | 🖨 Print ]
```

形态：**单 HTML 文件**，无构建步骤、无后端、无数据上传。所有计算在浏览器本地完成。

**Chapter 2 与 Chapter 3 解耦**：Chapter 2 的两张图始终展示「按规划目标取款」的理想路径（capital 在第 N 年精确归零）；Chapter 3 才展示两种策略下不同 withdrawal rate 的演变。

---

## 4. Step 1 详细规格

### 字段（按顺序）

**PROFILE 区**

| 字段 | 类型 | 默认 | 范围/校验 |
|---|---|---|---|
| Gender | Radio (Male / Female) | Male | 切换性别会自动重设 `yearsRetired`（M → 20, F → 25），覆盖手动调整值 |
| Current age | Slider | 40 | 18–80 |
| Desired retirement age | Slider | 65 | current_age + 1 to 80 |
| Years of retirement income needed | Slider | **20 (M) / 25 (F)** — gender-driven | 1–50 |
| Existing retirement savings | Currency input | S$ 100,000 | ≥ 0 |

**DESIRED LIFESTYLE 区**

- 3 个预设卡片，点击后自动填入下方 "Monthly expense" 输入框：
  - **Modest** S$2,300
  - **Comfortable** S$2,900（默认选中）
  - **Premier** S$5,200
- "Monthly expense (today's value)" 输入框，可在预设基础上自由微调

### 文案细节

- "Years of retirement income needed" 副文：「Average life expectancy in Singapore: 82 for males, 86 for females.」
- "Existing retirement savings" 副文：「All capital intended for retirement today — CPF, SRS, investments, cash reserves.」

### 导航

- 顶部 stepper：01 (active) · 02 · 03
- 底部按钮：`← Back`（禁用） · `Continue to Assumptions →`

---

## 5. Step 2 详细规格 · Assumptions & Outcome

### 5.1 三段费率（Three-Phase Model）

**① Pre-Retirement（绿色，永远展开）**

| 字段 | 默认 | 范围 |
|---|---|---|
| Investment return | 6.0% | 0–15% |
| Inflation | 2.5% | 0–10% |

**② Waiting Period（橙色，可选 toggle，默认 OFF）**

属于 pre-retirement 子阶段。**共用 Phase I 的费率**，无独立 return / inflation 输入。

| 字段 | 默认 | 范围 |
|---|---|---|
| Waiting years | 5 | 1–20 |

UI 文案：「Years **before retirement** when contributions stop — capital remains in the pre-retirement portfolio and keeps compounding.」
注释行：「Sub-phase of Pre-Retirement — uses the Phase I return and inflation rates throughout. No separate assumptions required.」

**③ Post-Retirement（蓝色，永远展开）**

| 字段 | 默认 | 范围 |
|---|---|---|
| Investment return | 4.0% | 0–15% |
| Inflation | 2.5% | 0–10% |

### 5.2 Outcome · Expense Through Time（3 KPI 卡）

显示客户期望月开销在三个时间点的名义价值，让通胀冲击显式可见：

| 卡 | 显示 | 说明 |
|---|---|---|
| Today's dollars | S$2,900 / mo | Chapter 1 输入值 |
| At retirement · age 65 | monthly_today × (1+iPre)^yearsPre | 退休时点（含 waiting） |
| At end of plan · age 85（alert 红） | × (1+iPost)^N | 计划末端（Male 默认 N=20），含 multiplier 标识（如 "3.0× today"） |

### 5.3 Outcome · Funding Gap（3 KPI 卡）

| 卡 | 显示 | 视觉 |
|---|---|---|
| Total capital requirement | LS（在退休时点） | 蓝色顶边（primary） |
| Existing savings, projected | existing × (1+rPre)^yearsPre | 蓝色顶边 |
| Net shortfall | LS − ES projected | 橙色顶边（alert） |

### 5.4 Outcome · Saving Plan（2 KPI 卡）

| 卡 | 显示 | 视觉 |
|---|---|---|
| Monthly investment from today | monthlySavings | 蓝色顶边（primary） |
| Investment duration | yearsContribution years（# contributions） | 蓝色顶边 |

注：waiting 启用时，yearsContribution = yearsPre − yearsWait，副文「contributions stop at age X · then Yy wait to retirement」。

### 5.5 Asset Balance 图

- Chart.js line chart
- X 轴：年龄（ageNow → ageEnd）
- Y 轴：资产余额（S$）
- 一条曲线：累积期上升 → 等待期继续轻微上升（无供款）→ 取款期下降至 0
- **使用 planned expense 取款**（非 Chapter 3 策略），因此 capital 精确归零于 ageEnd
- 底部三色色带 phase bar：Contribution · Wait · Decumulation

### 5.6 Cashflow 图

- Chart.js bar chart, stacked
- X 轴：年龄（ageNow → ageEnd）
- Y 轴：年度现金流（S$/year，正负双向）
- 三组堆叠 dataset：
  - **Deep forest（#2C5A3D）· One-time investment**：年 0 一根高柱 = existingSavings
  - **Sage（#5A8B6E）· Recurring investment**：contribution 年每年 = monthlySavings × 12（与 one-time 在 T=0 同时开始，BEGIN mode）
  - **Ember（#A03E2C）· Annual expense**：decumulation 年每年向下柱 = annualAtRetire × (1+iPost)^(t−1)
- Waiting 年无 bar
- 底部同款 phase bar，与 Asset Balance 共享同一时间轴

### 5.7 导航

- 顶部 stepper：✓ · 02 (active) · 03
- 底部：`← Back` · `Continue to Strategies →`

---

## 6. Step 3 详细规格 · Strategy Comparison

### 6.1 Live Adjustable Rates（顶部交互区）

- 双列布局，CP 滑块（teal）+ CD 滑块（ember）
- 拖动任一滑块 → 实时更新下方所有数字 + 两张图表
- **CP · Yield rate**：默认 4.0%，范围 0–10%，步进 0.1%
- **CD · Initial withdrawal rate**：默认 4.0%，范围 0–10%，步进 0.1%

### 6.2 Strategy 对比卡（双列）

**左 · 🌱 Capital Preservation（teal）**

| 行 | 显示 |
|---|---|
| Description | 「Draw only the yield. Principal stays intact — a legacy preserved, but income loses real value over time.」 |
| Level monthly income | LS × yield ÷ 12 |
| Real value @ end of plan | annual_income_CP / (1+iPost)^N ÷ 12（购买力警示） |
| Capital at end | 始终 = LS（100% preserved） |

**右 · 💧 Capital Drawdown（ember）**

| 行 | 显示 |
|---|---|
| Description | 「Withdraw the capital itself; income rises with inflation. Purchasing power maintained — but the well runs dry.」 |
| Initial monthly income | LS × w ÷ 12 |
| Final monthly income | W_N ÷ 12（注：W_t = W_1 × (1+iPost)^(t-1)） |
| Capital depleted at | 三种状态：<br>　— Year X · age Y（计划内耗尽）<br>　— Year N · "lasts exactly the planned span"（恰好）<br>　— S$Z surplus + 副文「Would deplete in year (N+k) (k years beyond plan)」 |

### 6.3 双图对比（底部并列）

**📊 Capital Balance — Decumulation（左图）**

- X 轴：decumulation 年 1..N
- 两条线：CP（teal 水平直线）+ CD（ember 下降曲线）

**💰 Annual Withdrawal — Decumulation（右图）**

- X 轴：decumulation 年 1..N
- 两条线：CP（teal 水平直线，= LS × yield）+ CD（ember 指数增长曲线）

### 6.4 Talking points 框（绿底 italic 句子）

> • 右图绿线：「Twenty years on, your monthly income is still the same — but a cup of coffee may cost twice as much.」
> • 右图红线：「With inflation adjustment, twenty years on you'd be drawing far more — same purchasing power as today.」
> • 左图：「The trade-off — Drawdown exhausts the principal; Preservation passes the principal on, intact.」

### 6.5 导航

- 顶部 stepper：✓ · ✓ · 03 (active)
- 底部：`← Back` · `Revise Assumptions`（回 Step 2） · `Print · Save as PDF`

---

## 7. 计算模型

### 7.1 派生量

```text
years_pre          = age_retire − age_now                          // 全 pre-retirement 跨度
years_wait         = waiting_enabled ? min(user_input, years_pre) : 0
years_contribution = years_pre − years_wait                        // 仅活跃供款年
age_contribution_end = age_now + years_contribution
N                  = years_retired
age_decum          = age_retire                                    // 取款立即从退休开始
age_end            = age_retire + N
```

**关键语义**：Waiting Period 位于 pre-retirement 跨度**内**，紧贴退休年前，**无供款**但本金仍按 r_pre 复利（i.e. r_wait = r_pre，i_wait = i_pre）。

### 7.2 ① 未来月开销（通胀到退休年）

```text
monthly_at_retire = monthly_today × (1 + i_pre)^years_pre
annual_at_retire  = monthly_at_retire × 12
```

注：Waiting Period 内同样按 i_pre 通胀，所以 yearsPre = yearsContribution + yearsWait 的累计指数等价于一段 i_pre^yearsPre。

### 7.3 ② Lump Sum 需求（Decumulation BEGIN mode / annuity due）

取款发生在每年**起始**：年初取 W_t，剩余按 r_post 增长，边界 B_N = 0。

递推：`B_t = (B_{t−1} − W_t) × (1 + r_post)`，W_t = W_1 × (1 + i_post)^(t−1)，B_0 = LS，W_1 = annual_at_retire。

闭式：

```text
if r_post ≠ i_post:
    LS = annual_at_retire × (1 + r_post) × [1 − ((1+i_post)/(1+r_post))^N] / (r_post − i_post)
else:
    LS = annual_at_retire × N
```

**Sanity check（N=1）：** B_1 = (LS − W_1)(1+r) = 0 ⇒ LS = W_1 ✓
代入闭式：`W_1 × (1+r) × (1 − q)/(r−i) = W_1 × (1+r) × ((r−i)/(1+r))/(r−i) = W_1` ✓

### 7.4 ③ 每月所需储蓄（Accumulation BEGIN mode / annuity due）

**模型**：existing savings 在 T=0 一次性入金；首笔月供也在 T=0 入金（**同时开始**，annuity due）。供款持续 yearsContribution 年（= years_pre − years_wait），随后 yearsWait 年只复利不供款。

```text
ES_at_contrib_end = existing × (1 + r_pre)^years_contribution
ES_at_retirement  = ES_at_contrib_end × (1 + r_pre)^years_wait      // r_wait = r_pre
                  = existing × (1 + r_pre)^years_pre
shortfall         = max(0, LS − ES_at_retirement)

target_at_contrib_end = shortfall / (1 + r_pre)^years_wait

m_pre = (1 + r_pre)^(1/12) − 1
M     = years_contribution × 12

// Annuity due (BEGIN) factor — 每月初存款，复利到第 M 月末:
annuity_factor = ((1 + m_pre)^M − 1) / m_pre × (1 + m_pre)

if shortfall = 0:
    monthly_savings = 0
else:
    monthly_savings = target_at_contrib_end / annuity_factor
```

### 7.5 ④ Capital Preservation 模型（Chapter 3）

```text
annual_income_CP(t)  = LS × yield_cp        // 任意 t 恒定
monthly_income_CP(t) = annual_income_CP / 12
capital_CP(t)        = LS                   // 任意 t 恒为 LS

real_income_at_end   = annual_income_CP / (1 + i_post)^N        // 末年实际购买力
```

### 7.6 ⑤ Capital Drawdown 模型（Chapter 3，BEGIN mode）

```text
W_1 = LS × withdrawal_cd                    // 用户输入；默认 4%
W_t = W_1 × (1 + i_post)^(t − 1)
B_0 = LS
B_t = (B_{t−1} − W_t) × (1 + r_post)        // 年初取款，BEGIN mode
```

与 Excel `Proposal_retire_invest.xlsx` worksheet `UT-FI-10` H 列 `=(H{t-1} − G{t}) × (1 + G23)` 一致。

**Depletion 检测（含浮点容差）：**

```text
EPS = max(W_1 × 0.001, 1)
depletion_year = first t ∈ [1, N] with B_t ≤ EPS
```

UI 三种状态（见 §6.2）：计划内耗尽 / 恰好计划末耗尽 / 计划外有盈余。

### 7.7 Asset Balance 时间轴（用于 §5.5）

**Chapter 2 charts 使用 PLANNED expense（与 Chapter 3 CD strategy 解耦）**：

```text
b = existing_savings
push({age: age_now, balance: b})

// Phase 1: Contribution (BEGIN mode, monthly compounding)
m_pre_rate = (1 + r_pre)^(1/12) − 1
for yr in 1..years_contribution:
    for m in 1..12:
        b = (b + monthly_savings) × (1 + m_pre_rate)    // BEGIN: 先加再增长
    push({age: age_now + yr, balance: b})

// Phase 2: Waiting (no contributions, compound at r_pre)
for yr in 1..years_wait:
    b = b × (1 + r_pre)
    push({age: age_contribution_end + yr, balance: b})

// Phase 3: Decumulation (PLANNED cost-of-living, BEGIN mode)
for yr in 1..N:
    E_t = annual_at_retire × (1 + i_post)^(yr − 1)
    b = (b − E_t) × (1 + r_post)
    push({age: age_decum + yr, balance: max(0, b)})
// 由 LS 的构造，B_N 在 EPS 范围内 = 0
```

### 7.8 Cashflow 年度序列（用于 §5.6）

```text
for age in age_now..age_end:
    one = (age == age_now) ? existing_savings : 0

    // Recurring 在 BEGIN mode 下从 age_now 开始，跨 years_contribution 年
    rec = (age_now ≤ age < age_contribution_end) ? monthly_savings × 12 : 0

    // Expense 在 BEGIN mode 下从 age_retire 开始，跨 N 年
    // 使用 planned cost-of-living，与 Chapter 3 CD strategy 解耦
    if age_retire ≤ age < age_retire + N:
        yr = age − age_retire + 1
        exp = −annual_at_retire × (1 + i_post)^(yr − 1)
    else:
        exp = 0

    push({age, one, rec, exp})
```

**注**：age = age_now 处同时拥有 `one`（一次性入金）和首笔 `rec`（首年 12 次月供），可视化为两色堆叠柱体——精确体现"BEGIN mode 下两者同时开始"。

---

## 8. 技术结构

### 8.1 文件结构

```text
Calculator/
├── retirement.html                         # 主交付物（单文件，~92KB，含双语 + 打印 CSS）
├── Proposal_retire_invest.xlsx             # 参考资料
└── docs/superpowers/specs/
    └── 2026-05-25-retirement-calculator-design.md   # 本文档
```

### 8.2 技术栈

- **HTML5** + **Vanilla JS**（ES2020+）
- **CSS**：内联 `<style>`，CSS Grid + Flexbox
- **Chart.js v4**（CDN `https://cdn.jsdelivr.net/npm/chart.js@4.4.0`）
- **Google Fonts**：Fraunces（variable, ital/opsz/wght/SOFT）+ IBM Plex Sans + IBM Plex Mono
- **无构建步骤、无后端、无数据上传**

### 8.3 状态管理

```js
const state = {
  // Step 1
  gender, ageNow, ageRetire, yearsRetired, existingSavings, monthlyExpenseToday,
  // Step 2 — pre + post rates only (waiting shares pre)
  rPre, iPre, waitingEnabled, yearsWait, rPost, iPost,
  // Step 3 (live)
  yieldCp, withdrawalCd,
  // UI
  currentStep,
};
```

- 任何滑块/输入变化 → `render()` 重算并重绘
- Step 2 + Step 3 滑块均触发实时更新（无 debounce，无 "Calculate" 按钮）

### 8.4 边界与校验

- 退休年龄必须 > 当前年龄
- `r_post = i_post` 时 lump sum 公式自动切换到退化形式
- CD depletion 用 EPS 容差判断（避免浮点误把 0 当成"用完之前"）
- ES projected ≥ LS 时 → 显示 "On track" / "None"，无月投资要求
- `years_wait` 自动夹紧到 ≤ years_pre

### 8.5 打印 / PDF

**目标**：从任一章节点击 `Print · Save as PDF` 按钮，生成 5 页 A4 完整 report（三章节全部 + 双图表），双语排版保留。

**触发流程**：按钮 onclick → `printReport()` → 加 `.printing` class → 两次 `requestAnimationFrame` 让布局稳定 → 对所有 4 个 Chart.js 实例调用 `.resize()` → `setTimeout 120ms` 让 canvas 重绘完成 → `window.print()` → `afterprint` 事件移除 `.printing` class。

**三处关键 CSS / JS 协同**（每一处都不可省略，否则会触发不同的失败模式）：

1. **`body.printing { ... }`**（屏幕模式覆盖）——`.step { display:block; opacity:1; transform:none }` 让所有章节同时可见；同时**预先把图表容器缩到打印尺寸**（`.timeline-chart-wrap { height:120px }`、`.chart-wrap { height:160px }`），与下面的 `@media print` 完全一致。
   - **为何必须预缩容器**：`@media print` 仅在 PDF 截屏那一刻生效，已晚于 Chart.js 完成绘制；若不预缩，canvas 会锁定在屏幕模式的 200px 高度并溢出 160px 的打印容器，造成图表与图例重叠。
2. **`@media print { ... }`**（打印模式精排）——隐藏 stepper / 按钮 / 噪点；`.step[data-step="2"], .step[data-step="3"] { page-break-before: always }` 让每章新页起；`.masthead, .phase, .kpi-row, .strat, .chart-card, .narrative { page-break-inside: avoid }` 保护卡片完整；紧凑字号；图表图例改为垂直堆叠以适配窄卡。
3. **`printReport()` 函数**——在屏幕模式下点击时启动；通过 `.printing` class 让屏幕短暂呈现"打印态"以便 Chart.js 重绘，再触发 `window.print()`。

**无头测试钩子**：URL 携带 `?report=1` 时，`DOMContentLoaded` 即添加 `.printing` class 并在 `init()` 后调用 `chart.resize()`，无需点击按钮即可让无头 Chrome 抓取完整 PDF：

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --headless=new --no-pdf-header-footer --virtual-time-budget=5000 \
  --print-to-pdf=out.pdf \
  "http://localhost:8765/retirement.html?report=1"
```

**分页布局**（默认参数 / Male）：

| 页 | 内容 |
|---|---|
| 1 | Chapter 1 全部：masthead + Profile（5 字段）+ Desired Lifestyle（3 预设 + 月开销输入） |
| 2 | Chapter 2 前半：masthead + Three-Phase Model + Expense Through Time KPI + Funding Gap label |
| 3 | Chapter 2 后半：Funding Gap KPI + Saving Plan KPI + Asset Balance 图 + Cashflow 图 |
| 4 | Chapter 3 前半：masthead + Live rate controls + Strategy 对比卡（双列） |
| 5 | Chapter 3 后半：Capital Balance 图 + Annual Withdrawal 图 + Talking points |

---

## 9. 视觉风格

**设计方向**：Editorial Wealth — Quiet Confidence（编辑感财富管理 + 静谧自信）

### 9.1 配色

- 背景：parchment `#F4EEDF`（暖米黄）
- 卡片背景：`#FAF5E8` / `#FFFAEE`（米色递进）
- 文字：ink `#1A1814`（深墨色）/ ink-soft `#6B6358` / ink-faint `#9B9388`
- Hairlines：`rgba(26, 24, 20, 0.12)` 至 `rgba(26, 24, 20, 0.32)`
- **主 accent**：oxblood `#6B1820`（章节号、primary CTA、emphasis）
- **三段色**：pre = forest `#2C5A3D` · wait = amber `#B5803A` · post = deep blue `#1F3D5C`
- **策略色**：CP = teal `#3D6661` · CD = ember `#A03E2C`

### 9.2 字体

| 角色 | 字体 | 备注 |
|---|---|---|
| Display headings（章节号、title、kicker） | **Fraunces** variable serif | opsz / wght / SOFT 三轴，可调温度 |
| Editorial italic（lede、phase 副文、annotations） | **Fraunces** italic | SOFT 100，温和 |
| Body sans | **IBM Plex Sans** | 金融行业经典 |
| Labels（uppercase tracked、KPI 标签、stepper） | **IBM Plex Mono** | 紧凑、可读 |
| 财务数字（KPI 数值、Strategy 结果、Slider 值、Input） | **IBM Plex Mono** | tabular-nums lining-nums，账本式精度 |
| 编辑性数字（masthead 章节号 "02"、Lifestyle 预设价格、罗马数字 I/II/III） | **Fraunces** | 保留编辑情感，与数据数字区分 |

### 9.3 其他细节

- 微噪点纹理（SVG turbulence filter）增强纸张质感
- 双重水平线分隔 chapter masthead
- Phase bar 用统一三色色带，Asset Balance 与 Cashflow 双图共享
- 适配最小宽度 768px（笔记本 / iPad 演示场景，不针对手机）

### 9.4 双语并排（EN + 简体中文）

**原则**：英文为视觉主线（保留原编辑感字体与字号），中文紧随其下做译注，使用弱化色阶与较小字号确保层级清晰，不喧宾夺主。

**字体栈**：`--zh: 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', 'Noto Sans CJK SC', sans-serif` —— 全系统字体回退，无需额外 webfont。

**`.zh-*` class 系列**（按语境差异化排版，避免一刀切）：

| Class | 用途 | 屏幕字号 | 颜色 |
|---|---|---|---|
| `.zh-title` | masthead 主标题中文 | 19px | ink-soft |
| `.zh-lede` | 章节引言段中文 | 13.5px | ink-faint |
| `.zh-kicker` | "Chapter One" 后的"第一章" | 10.5px | ink-soft |
| `.zh-lbl` | form 字段标签、section 标签 | 11px | ink-faint |
| `.zh-help` | 帮助文字、KPI 副文 | 11.5px | ink-faint |
| `.zh-kpi` | KPI label 中文（块级） | 10.5px | ink-faint |
| `.zh-strat` | Strategy 卡片名称、图表标题中文 | 15px | ink-soft |
| `.zh-phase` | phase 卡副标题中文 | 13px | ink-soft |
| `.zh-narr` | Talking points 中文译句 | 12.5px | ink-soft |
| `.zh-legend` | 图表图例中文 | 9px | ink-faint |
| `.zh-btn` | 按钮下方中文 | 9.5px | inherit · opacity 0.85 |

**水平 vs 垂直**：`.zh-lbl` 默认 `display: block`（中文换行至英文下方）；但在 `.section-lbl .txt`、`.years-bar .ylabel`、`.toggle .lbl`、`.narrative .nlbl` 等横向组件中通过子选择器改回 `display: inline-block; margin-left: 10px`，让中文与英文同行。

**动态文案双语化**：JS 中所有 `textContent = '...'` 改为 `innerHTML = '... <span class="zh-help">...</span>'`，覆盖 `kpiExpRetSub` / `kpiExpEndSub` / `kpiLumpSumSub` / `kpiESProjectedSub` / `kpiShortfallSub` / `kpiMonthlySub` / `kpiDurationSub` / `cdMonthlyFinalNote` / `cdDepleteNote` / `waitSub`。

**专业术语处理**：CPF / SRS / MAS 等英文缩写保留原形并在中文里加注（如"含公积金（CPF）、补充退休计划（SRS）"），符合新加坡中文金融写作惯例。

---

## 10. 验收标准（Definition of Done）

1. ✅ 双击 `retirement.html` 在 Chrome / Safari / Edge 任一现代浏览器中正常打开，无 console error
2. ✅ Step 1 → 2 → 3 三步导航顺畅，"Back" 不丢失已填数据
3. ✅ Step 2 / Step 3 任意滑块拖动时，所有数字与图表实时更新（无延迟、无 debounce）
4. ✅ **默认参数完整 KPI 校验**（Male 默认：ageNow 40, ageRetire 65, **N 20**, ES 100k, pre 6%/2.5%, post 4%/2.5%, exp 2,900, yield 4%, w 4%）：
   - Expense at retirement = S$5,376（= 2,900 × 1.025²⁵）
   - Expense at end of plan = S$8,810（= 2,900 × 1.025⁴⁵，约 3.0× today）
   - **Lump sum = ~S$1,127,947**（accum BEGIN + decum BEGIN）
   - ES projected = ~S$429,187
   - Net shortfall = ~S$698,760
   - **Monthly investment = ~S$1,028**（accum BEGIN + decum BEGIN）
   - Plan span = 40 → 85
   - **Cashflow 首根 expense bar 在 age 65（BEGIN mode）**，值 = −annualAtRetire = −S$64,517
   - **Cashflow 末根 expense bar 在 age 84**（BEGIN mode 下 yr = N 对应 age = ageRetire + N − 1）
   - **Female 切换交叉参考**（选 Female → N=25 自动重设）：LS ≈ S$1,362,332 · monthly investment ≈ S$1,373 · expense at end ≈ S$9,968 · plan span 40 → 90 · 末根 expense bar 在 age 89
5. ✅ **TVM 双 BEGIN mode 一致性**：
   - Accumulation：先加供款再增长 `b = (b + PMT)(1+m)`；闭式月供因子 `((1+m)^M − 1)/m × (1+m)`
   - Decumulation：先取款再增长 `B_t = (B_{t−1} − W_t)(1+r)`；闭式含 `(1+r_post)` 因子
6. ✅ **Chapter 2 与 Chapter 3 解耦**：拖动 Chapter 3 的 withdrawalCd 滑块，Chapter 2 的 Asset Balance / Cashflow / KPI 完全不变；Chapter 3 的两张图随之变化
7. ✅ **B_N = 0 at end of plan**：默认参数下 Asset Balance 末年余额 < S$1（实测 ~7e-8 ≈ 0）
8. ✅ **CD 策略默认行为**（Male 默认 N=20）：W_1 = LS × 4% = ~S$45,118；隐含 goal-seeking 取款率 = annualAtRetire / LS ≈ 5.72%；故默认 4% 取款率下 CD 在计划内不耗尽，约 31 年后才耗尽（depletionYearEventual ≈ 31）。Female 切换（N=25）：W_1 ≈ S$54,493；隐含率 ≈ 4.74%；depletionYearEventual ≈ 31
9. ✅ **Capital Preservation 模型**：annual_income_CP = LS × yield 恒定；capital_CP(t) = LS 恒定
10. ✅ **Capital Drawdown vs. Excel UT-FI-10**：用 Excel 输入（LS S$560,916、yield/w 4%、r/i 4%/2.5%）：
    - B_1 ≈ S$560,019 · B_30 ≈ S$36,812 · B_31 ≈ −S$10,660（depletion year ≈ 30 或 31，BEGIN mode 一致）
11. ✅ **「Print / PDF」生成的版面是 5 页 A4 完整 report**，包含三章节全部内容：
    - 无 stepper / 按钮 / 噪点 / live-rate 标签
    - 4 张图表（Asset Balance / Cashflow / Capital Balance / Annual Withdrawal）全部正确缩放，坐标轴与图例无重叠
    - 每章节自然分页（`page-break-before: always` on Chapter 2/3），卡片不跨页切断
    - 双语排版保留（英文 + 中文译文均完整呈现）
12. ✅ **双语完整性**：所有静态文案与动态生成文案（KPI 副文、按钮、图表图例、phase bar、talking points）均提供中文译文；中文以系统字体（PingFang SC / 思源黑体）回退，无需额外 webfont
13. ✅ 文件大小 < 200KB（不含 Chart.js CDN 与 Google Fonts）

---

## 11. 未决事项

无。所有设计要点已在 2026-05-25 brainstorming + 实施迭代中确认。

---

## 12. 后续版本（v2+）路线图（仅作记录）

- **v2**：CPF LIFE 月入估算（Standard / Basic / Escalating 三 plan 对比）
- **v2**：SRS 供款 + 节税 + 分 10 年提取规划
- **v3**：多收入流叠加（年金 / 租金 / 股息）
- **v3**：多情景并行对比（保守 / 中性 / 激进 side-by-side）
- **v4**：客户姓名/日期字段（PDF header）、本地存储、多版本对比
