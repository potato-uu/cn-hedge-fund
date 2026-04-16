<div align="center">

# cn-hedge-fund

「*证据只算一次，决策只信因子，大师只管解释。*」

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE) [![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-c0392b.svg)](#) [![Prompt-only](https://img.shields.io/badge/prompt--only-no%20runtime-8e44ad.svg)](#)

**中国 A 股 / 港股 / 美股四层架构分析 skill — Evidence → Factor → Decision → Narration**

纯 prompt 驱动，无外部代码依赖，数据通过 WebSearch 获取。

[快速开始](#使用方式) · [四层架构](#四层架构v2-重构) · [开源 core vs 私有 runtime](#architecture-open-core-vs-private-runtime) · [免责声明](#免责声明)

</div>

---

## 效果示例

```
用户  › 分析一下 300750

skill › Step 0 Instrument Router：A 股个股 / 科技成长 / 动力电池
        → 开启：PE/PEG/DCF/ROE/FCF/技术面/A 股 playbook
        → 关闭：ETF 指标 / 生科 runway / 银行 NIM

      Factor Layer：
        Quality      +2   ROE 18%，FCF/净利 0.9
        Growth       +1   YoY 22%，毛利率稳定
        Valuation    +1   PE 历史分位 35% / PEG 0.8
        Trend        +2   布林中轨向上 + MA 多头
        Liquidity    +1   北向持续加仓
        Regime       +2   新能源供需出清末段
        Tail Risk     0   海外业务 28%
        加权总分：+1.37 → 🟢 BUY

      Decision：信号 BUY / 置信度 0.72 / 风险预算上限 15%

      Narration：
        巴菲特：Quality=+2 Valuation=+1，在我框架下是"合理价格买好公司"
        芒格：业务能讲清，ROIC 持久 → "简单好生意"
        德鲁肯米勒：Regime=+2 Trend=+2 → "产业趋势加速，集中度机会"
        塔勒布：Tail Risk=0，但提醒——高估值+高预期的标的，一次失望就崩
        A 股短线：布林中轨向上 + 量价齐升 → "主升浪早期"
```

---

## 为什么重构（v2）

旧版"8 大师 + 4 分析师"分别给 BUY/HOLD/SELL + 置信度，然后投票加权。

问题：
- **Double counting**：同一份财报被 Buffett 看一遍、Munger 看一遍、基本面分析师再看一遍，"共识"是幻觉
- **凯利公式失效**：散户没有 N≥50 同类交易样本库，凯利公式无法计算
- **A 股偏置**：技术面 / 情绪模块强行套在美股和 ETF 上会失真
- **人格投票机制不稳定**：8 个人格权重相等就变成平均数，加权则人为偏见

v2 的修正：
1. **Step 0 Instrument Router** — 先分类资产类型，再开关相应指标模块
2. **Factor Layer 唯一决策源** — 证据只打一次分，7 因子加权输出信号
3. **人格降级为 evidence selector + narrator** — 告诉采集层他们关注哪些数据，Factor 打完分再用他们的框架解释
4. **Risk Budget 替代 Kelly** — 无样本库时用固定仓位上限（核心 ETF 25% / 行业 ETF 15% / 龙头 10% / 一般 5-8% / 小盘 0-3%）
5. **置信度 4 子项分解** — min(data_completeness, source_quality, factor_consistency, market_fit)（取瓶颈而非乘积，避免过度悲观）

---

## 四层架构（v2 重构）

```
Step 0  Instrument Router          ← 分类 → 决定开哪些模块
─────────────────────────────────
Evidence Layer                     ← 原始事实（财报/公告/价格/资金流/新闻）
─────────────────────────────────
Factor Layer                       ← 7 因子打分，Decision 唯一来源
  Quality / Growth / Valuation
  Trend / Liquidity / Regime / Tail Risk
─────────────────────────────────
Decision Layer                     ← 信号 + risk budget 仓位
─────────────────────────────────
Narration Layer                    ← 8 位大师解释（不投票）
```

---

## 使用方式

### 通用 AI 助手（ChatGPT / Claude.ai / Gemini / 其他）

把 `SKILL.md` + `references/*.md` 的内容贴入你的 AI 对话作为系统 / 上下文提示，然后问：

- `分析一下 300750`
- `用这套框架分析 NVDA 和 9988.HK`
- `当前 A 股主线是什么`

### Claude Code / 支持 skill 机制的助手

```bash
git clone https://github.com/<your-user>/cn-hedge-fund.git ~/.claude/skills/cn-hedge-fund
```

触发词激活 skill，按四层架构输出报告。

### 其他 AI 编程工具（Cursor / Cline / OpenClaw）

把 `SKILL.md` 作为系统提示或 rule 文件引入，`references/` 作为附加知识库。

---

## 文件结构

```
cn-hedge-fund/
├── SKILL.md                         # 入口（触发词 + 四层流程）
├── README.md                        # 本文件
├── LICENSE                          # MIT
└── references/
    ├── personas.md                  # 8 位大师 — evidence selector + narrator（不投票）
    ├── analysts.md                  # Factor Layer 7 因子详细规则 + Decision 加权
    ├── playbook.md                  # A 股短线实战手册（2500+ 行）
    └── main-line-workflow.md        # 主线推演子流程（赛道级分析）
```

---

## Architecture: Open Core vs Private Runtime

本仓库只是整个系统的 **公开 core**。真实的实盘运行还有一层 **私有 runtime**，两层之间的界线是刻意的：

```
┌─────────────────────────────────────────────┐
│ Public Core（本仓库 cn-hedge-fund）         │
│   - 四层架构：Evidence → Factor → Decision   │
│     → Narration                              │
│   - 7 因子打分规则 + 跨市场开关矩阵         │
│   - 8 位大师 narration 模板                  │
│   - A 股 playbook（2500+ 行蒸馏方法论）     │
│   - 主线推演子流程                           │
└─────────────────────────────────────────────┘
                    ↑  调用 / 引用
┌─────────────────────────────────────────────┐
│ Private Runtime（不在本仓库）               │
│   - RAG 检索层（私有语料库）                │
│   - 持仓监控器（个人账户规则）              │
│   - 早报管道（每日 08:00 launchd + Gmail）  │
│   - 主线推演 evidence 采集（5 路并行）      │
│   - 个人人格扩展（可选）                    │
└─────────────────────────────────────────────┘
```

**为什么要分：**
- Public core 是**可审计、可复用**的方法论 —— 任何人拿去都能跑起来，不依赖私人数据
- Private runtime 是**有状态、有副作用**的工程层 —— 和个人持仓、私有 RAG、定时任务强耦合，不适合公开
- 分层后 core 的变化（比如因子权重调整）会被 runtime 显式引用而不是隐式继承 —— drift 可追踪

**如果你要自建 runtime：**
建议保留 public core 的四层契约和 7 因子定义，在本地 override skill 里只注入个人数据（持仓 / RAG / 定制人格），不要把决策逻辑 fork 出去 —— 否则升级时会冲突。

```
~/.claude/skills/cn-hedge-fund-<yourname>/
├── SKILL.md                        # 触发词与本 skill 协同
└── references/
    └── holdings.md                 # 个人持仓 + 成本基础 + 运行时钩子
```

override skill 不重复 core 内容，只在 trigger 中追加"优先引用持仓对应的 ETF"、"附加私有 RAG 查询"等 context。

---

## 决策思维元原则（8 框架）

分析开始前的过滤器，保留自 v1：林奇卖赢家 / 芒格 inversion / 德鲁肯米勒集中度 / 第一性原理 / Risk Budget（禁用凯利）/ 三层因子独立 / 具体化（金额+触发）/ 行为金融校准（单次 ≤5%）。

完整定义见 `SKILL.md` § 决策思维元原则。

---

## A 股实战手册（playbook.md）

完整方法论见 `references/playbook.md`，涵盖：

- **第一章 宏观与事件驱动**：Fed 降息 / 央行逆回购 / 通胀滞胀 / 战争 / 关税 / 五年规划 / 非农 CPI GDP PMI
- **第二章 行业周期**：生命周期 + 8 行业剧本（新能源 / AI / 创新药 / 军工 / 白酒 / 金融 / 房地产 / 周期）+ 财报季
- **第三章 估值方法论**：历史分位 / PEG / PB / 估值切换 / AIAE / 巴菲特指标 / ERP / 泡沫识别
- **第四章 技术面操盘**：布林带 / 量价 / 炮轰市场 / RSI / 跳空 / 天量天价 / 二四六止盈
- **第五章 仓位管理**：二四六止盈法 / 定投启停 / 金字塔建仓
- **第六章 投资心理学**：10 个陷阱 + 6 阶段情绪周期

---

## 致谢

- 框架设计启发自 [virattt/ai-hedge-fund](https://github.com/virattt/ai-hedge-fund)
- `playbook.md` 的 A 股实战框架蒸馏自公开社区的复盘讨论，已去身份化，仅保留方法论

---

## 免责声明

本 skill 基于公开信息和启发式阈值生成分析，**不构成投资建议**。阈值和权重为设计假设，不是回测优化结果；本 skill 无样本库，所有分数需结合当前市场环境判断。单一指标失效时降级处理。

投资有风险，决策请自行判断并承担后果。

---

## License

MIT
