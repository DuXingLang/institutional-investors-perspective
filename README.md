<a id="top"></a>

<!-- Language tabs -->

<p align="center">
  <a href="#中文"><strong>中文</strong></a>
  &nbsp;|&nbsp;
  <a href="#english">English</a>
</p>

---

<a id="中文"></a>

# 机构投资者视角 · 主力操盘研判 Skill

> 把 Agent 放进机构主力的位置：先判断自己处于什么操盘阶段，再推演下一步动作，最后用历史数据校验。

本 Skill 面向 A 股个股研究。当用户提供一只股票时，Agent 不再只回答「明天涨不涨」，而是切换到「已经持有大量筹码的机构主力」视角，围绕筹码、成本、抛压、资金效率与退出流动性进行分析，输出今天与明天的可执行交易预案。

它是一套**纯提示词工作流**：核心内容只有 `SKILL.md`，不携带行情接口、脚本、模型或数据文件。实时行情、历史 K 线、资金流向、筹码分布、板块与新闻数据，均通过运行环境中已安装的 Skill / MCP 获取；数据缺失时，Agent 必须明确标注并降低置信度。

> **重要边界**：所谓「主力行为」「主力成本」都是基于公开行情特征的行为推演与假设，不代表能够识别真实机构账户、真实持仓或真实操盘意图。本项目不构成投资建议，也不保证收益。

**快速导航：** [概述](#cn-overview) · [核心能力](#cn-features) · [工作流程](#cn-workflow) · [三大引擎](#cn-engines) · [联合决策](#cn-decision) · [输出报告](#cn-report) · [安装与使用](#cn-usage) · [数据要求](#cn-data) · [边界与风控](#cn-guardrails) · [工程化扩展](#cn-extension) · [目录结构](#cn-structure)

<a id="cn-overview"></a>

## 概述

普通投资者关心的是：这只股票明天涨还是跌？

机构主力关心的是：

- 我手里可能有多少筹码？
- 我的建仓成本大概在哪里？
- 当前是盈利还是浮亏？
- 上方还有多少套牢盘和抛压？
- 继续拉升需要多少资金，划不划算？
- 现在是该吸筹、洗盘、拉升，还是派发？
- 如果现在退出，流动性够不够？

本 Skill 让 Agent 沿着这些问题推演，并用三个引擎形成闭环：

1. **主力行为识别**：判断现在可能在做什么。
2. **主力操盘情景模拟**：推演接下来可能怎么做。
3. **历史相似走势回测**：验证过去类似情况下实际发生了什么。

<a id="cn-features"></a>

## 核心能力

| 能力             | 回答的问题                                     | 主要输出                                               |
| ---------------- | ---------------------------------------------- | ------------------------------------------------------ |
| 主力行为识别     | 当前最可能处于哪个操盘阶段？                   | 九类行为的匹配评分、量价/资金/价格证据、识别置信度     |
| 持仓与成本模型   | 主力成本区、套牢区、兑现区可能在哪里？         | 推测成本区间、当前相对成本位置、支撑与压力、可信度     |
| 操盘情景模拟     | 吸筹、洗盘、拉升、高位换手、派发，哪种更占优？ | 各策略成功概率、预期收益、潜在回撤、综合评分           |
| 历史相似走势回测 | 历史上类似结构之后发生了什么？                 | 相似样本、次日与 3/5/10 日统计、MFE/MAE、偏差检查      |
| 联合决策         | 三个引擎结论是否一致？冲突如何处理？           | 0～100 综合评分、冲突说明、置信度调整                  |
| 交易预案         | 今天、明天具体怎么做？                         | 高开/平开/低开三情景推演、触发条件、支撑压力、失效条件 |

<a id="cn-workflow"></a>

## 工作流程

```text
用户提供股票
    ↓
识别股票代码与所属市场
    ↓
发现并调用已安装的行情 Skill / MCP
    ↓
实时行情 / 历史 K 线 / 资金 / 筹码 / 板块 / 新闻
    ↓
数据完整性检查与标准化
    ↓
主力行为识别引擎
    ↓
机构主力持仓与成本模型
    ↓
主力操盘情景模拟引擎
    ↓
历史相似走势回测引擎
    ↓
联合评分、冲突处理与概率校准
    ↓
生成今天及明天的操盘计划
    ↓
输出标准研判报告
```

1. **识别标的**：把股票名称或代码解析为代码、市场与所属板块；存在歧义时先向用户确认。
2. **发现工具**：检查当前环境已安装的行情、资金、筹码、板块与回测工具，不假设某个工具一定存在。
3. **采集数据**：至少获取最近 120 个交易日日 K（尽量 240 个），记录查询时间、数据源与数据周期。
4. **行为识别**：从量价、成交量与换手、筹码结构、资金流向、技术趋势、板块与市场六个维度评分。
5. **情景模拟**：在假设持仓与成本区间下，模拟五种操盘策略 × 三种市场环境。
6. **历史回测**：在 2～5 年历史数据中检索相似样本，统计后续表现并做偏差控制。
7. **联合决策**：加权汇总三个引擎，识别冲突，校准置信度。
8. **输出报告**：按固定十节结构给出今日复盘与明日预案。

如果用户提供了自己的真实持仓成本和仓位，应单独建立用户持仓模型，不与假设的机构持仓混为一谈。

<a id="cn-engines"></a>

## 三大分析引擎

### 1. 主力行为识别引擎

目标不是靠 K 线形态「猜主力」，而是通过量价特征、资金变化、筹码交换、分时行为和市场环境，识别当前最可能发生的资金行为，并输出匹配程度而非确定性结论。

| 行为类型 | 识别目标                         |
| -------- | -------------------------------- |
| 底部吸筹 | 是否在低位持续积累筹码           |
| 震荡吸筹 | 是否通过横盘逐步收集筹码         |
| 试盘     | 是否在测试上方抛压与市场承接能力 |
| 洗盘     | 是否通过回调清理短线浮筹         |
| 震仓     | 是否通过剧烈波动改变筹码结构     |
| 启动拉升 | 是否开始推动股价脱离整理区       |
| 主升浪   | 是否进入持续上涨阶段             |
| 高位换手 | 是否在高位交换筹码               |
| 派发出货 | 是否出现持续减持或退出迹象       |

**量价五态**

| 量价形态 | 可能解释                                 | 关键确认                       |
| -------- | ---------------------------------------- | ------------------------------ |
| 缩量下跌 | 抛压衰减、观望、技术回调或洗盘中的低交换 | 是否放量反包、是否守住支撑     |
| 放量上涨 | 资金进攻、新资金进入、试盘成功或主升启动 | 上涨位置、收盘位置、量能持续性 |
| 放量下跌 | 恐慌换手、趋势破坏、高位撤退或消息冲击   | 是否失守关键支撑、后续能否收复 |
| 放量滞涨 | 上方抛压较重、存在高位派发风险           | 上影线、冲高回落次数、收盘强弱 |
| 缩量横盘 | 蓄势整理、等待方向或关注度下降           | 波动率、量能是否重新放大       |

缩量下跌不自动等于洗盘，放量上涨也不自动等于主升浪；必须结合位置、收盘强弱和后续持续性判断。

**异常行为检测**：尾盘突然放量拉升、早盘冲高回落、盘中跳水后收复、关键价位反复大单成交、放量突破后迅速跌回、连续多日异常量、价涨而资金指标偏弱、价跌而成交持续萎缩、高位放量长上影、低位放量长下影。只有行情接口提供盘口或逐笔数据时才识别，且不凭挂单推断真实资金意图。

**成本与筹码模型**：结合成交密集区、价格区间成交量分布、阶段换手率、OBV、成交额与股本关系、筹码分布，划分潜在建仓区、成本密集区、当前价格区、上方套牢区与潜在兑现区。没有可靠筹码数据时，必须降低推断置信度，不能把全部历史成交都归给机构。

**行为评分权重**

| 维度           | 权重 |
| -------------- | ---- |
| 量价关系       | 25%  |
| 成交量及换手率 | 20%  |
| 筹码结构       | 15%  |
| 资金流向       | 15%  |
| 技术趋势       | 15%  |
| 板块与市场环境 | 10%  |

`BehaviorScore = Σ(各维度标准化评分 × 对应权重)`

九类行为分别计算 0～100 分，取最高分作为主要假设，同时给出第二可能行为、主要证据、可能推翻判断的信号与置信度。

### 2. 主力操盘情景模拟引擎

核心假设是：Agent 已经持有该股票大量筹码。引擎分别模拟不同操盘策略下未来可能如何演变，以及主力应如何调整操作。

| 策略     | 假设                                 | 观察重点                                       |
| -------- | ------------------------------------ | ---------------------------------------------- |
| 继续吸筹 | 当前价格仍有建仓价值                 | 量能是否收敛、下跌是否有承接、价格中枢是否抬高 |
| 主动洗盘 | 筹码已积累，但短线浮筹过多           | 回踩均线、跌破支撑后能否收复、是否放量破位     |
| 主动拉升 | 已具备启动条件，希望提升持仓价值     | 突破有效性、上方套牢压力、量能持续、板块配合   |
| 高位换手 | 已有明显涨幅，观察市场承接           | 换手是否健康，还是已出现派发特征               |
| 逐步派发 | 达到收益目标或环境恶化，需要降低持仓 | 是否持续弱势、反弹是否缺量、风险收益是否恶化   |

同时至少模拟三种市场环境：

- **强势上涨**：大盘走强、板块上涨、成交额增加、承接增强。
- **震荡整理**：大盘波动有限、板块分化、个股量能下降。
- **市场下跌**：大盘走弱、板块资金流出、承接能力下降。

计算上优先调用已有量化回测或模拟工具；没有专用工具时，可用历史波动率、ATR 区间、历史相似走势条件分布、市场状态条件分析与蒙特卡洛模拟构建简化模型。不允许仅凭 LLM 自由想象价格路径。

**情景评分权重**

| 评价维度     | 权重 |
| ------------ | ---- |
| 预期收益空间 | 25%  |
| 成功概率     | 25%  |
| 下行风险     | 20%  |
| 成交量支持   | 10%  |
| 板块环境     | 10%  |
| 筹码结构     | 10%  |

最终输出五种策略的成功概率、预期收益、潜在风险与综合评分，并回答：如果我是主力，最可能选哪种策略、为什么、什么条件会让我改变策略。

### 3. 历史相似走势回测引擎

不只问「模型觉得明天会不会涨」，而是在历史行情中寻找与当前技术结构、量价特征、筹码状态和市场环境相似的案例，统计这些案例之后的真实表现。

**比较特征**

- **价格结构**：5/10/20/40/60 日涨跌幅、距阶段高低点幅度、均线排列、整理区间、突破或回踩形态。
- **成交量**：量比、5/10/20 日均量、量能变化率、换手率变化、成交额变化、OBV 变化。
- **技术指标**：MACD、KDJ、RSI、BOLL、ATR、均线斜率、价格偏离均线程度。
- **市场环境**：大盘与板块趋势、市场成交额、涨跌家数、涨跌停数量、连板高度、风险偏好。
- **主力行为特征**：缩量整理、放量突破、放量滞涨、连续资金流入、高位换手、疑似吸筹或洗盘。

**相似度权重**

| 相似度维度   | 权重 |
| ------------ | ---- |
| 价格结构     | 25%  |
| 量价关系     | 25%  |
| 技术指标     | 15%  |
| 市场环境     | 20%  |
| 主力行为特征 | 15%  |

不同量纲的指标必须先标准化，再计算加权距离：

```text
D(X, Y) = Σ wᵢ × distance(xᵢ, yᵢ)
Similarity = 1 / (1 + D)
```

选取相似度最高的 20～50 个历史样本；样本不足时必须说明，不得强行凑数。统计口径包括：次日涨跌与横盘概率、均值与中位数涨跌幅、平均最高涨幅、平均最低跌幅、2/3/5/10 日累计表现、涨停比例、最大有利波动（MFE）与最大不利波动（MAE）。横盘阈值需事先声明，例如收盘涨跌幅绝对值不超过 0.5%。

**质量控制**

- 历史样本不得包含当前交易日之后的数据，不得使用未来数据构建当前特征。
- 相邻交易日形成的高度重叠样本不能简单视为独立案例。
- 同时保留上涨、横盘与下跌案例，不能只挑成功样本。
- 考虑同板块、同市场环境的其他股票，以及不同市场阶段的差异。
- 进行样本外验证与 Walk-forward 验证，防止幸存者偏差。
- 历史条件频率不等于经过校准的未来概率。

<a id="cn-decision"></a>

## 联合决策

三个引擎不能各自输出结论，必须联合分析。

| 模块         | 权重 |
| ------------ | ---- |
| 主力行为识别 | 30%  |
| 主力情景模拟 | 25%  |
| 历史相似走势 | 30%  |
| 实时市场环境 | 15%  |

```text
FinalScore = 0.30 × BehaviorScore
           + 0.25 × ScenarioScore
           + 0.30 × HistoricalScore
           + 0.15 × MarketScore
```

所有子评分统一到 0～100 分。评分用于排序和决策辅助，不直接等于上涨概率；模块之间可能使用相同特征，存在信息重复；权重应通过样本外验证调整；数据缺失时必须重新归一化或降低整体置信度。

**冲突处理**：当三个引擎结论不一致时，不能简单取平均值。应检查各自依赖的证据、数据时间是否一致、板块环境是否变化、是否有重大消息冲击，以及哪个模型更适用于当前市场状态。若冲突无法消解，降低置信度，优先给出防守型策略。

<a id="cn-report"></a>

## 输出报告

每次分析固定输出以下十节：

1. 股票基本信息
2. 当前行情分析
3. 主力行为识别结果
4. 主力操盘情景模拟
5. 历史相似走势回测
6. 三引擎联合研判
7. 如果我是机构主力，今天会怎么操作？
8. 如果我是机构主力，明天会怎么操作？
9. 明天走势概率预测
10. 最终交易决策

其中第 9 节要求上涨、横盘、下跌三种概率合计为 100%，并给出预计涨跌幅区间、收盘价区间、涨停概率与判断置信度。第 10 节必须回答：是否适合加仓、什么情况减仓、什么情况退出、当前最大风险是什么，并用一句话给出核心判断。

报告必须明确区分：

- **客观事实**：行情、成交量、换手率、资金与板块数据。
- **量化推断**：技术指标、量价信号、行为匹配评分。
- **情景模拟**：明确假设下的可能路径。
- **历史统计**：相似样本的后续表现。

<a id="cn-usage"></a>

## 安装与使用

1. 将本目录放入 Agent 运行环境所配置的 skills 目录，确保 `SKILL.md` 可被加载；如果使用 SkillHub，可按目录 slug 安装。或者直接跟你的Agent说：

   `帮我安装这个skill：https://github.com/DuXingLang/institutional-investors-perspective`
2. 在对话中直接提出分析请求，例如「用机构主力视角分析 600000」或「判断这只票主力是在吸筹还是出货」。
3. 可选输入：你的真实持仓成本与仓位、观察周期、风险偏好、重点关注的价格位。真实持仓会被单独建模，不与假设的机构持仓混合。

示例请求：

```text
用机构主力视角分析 600000：
假设我已持有较大比例筹码，成本区间 10.2–10.8 元。
请判断当前操盘阶段，并给出今天与明天的操盘预案。
帮我从主力的视觉分析一下XXXX这只股票今天及明天的走势。
```

**元数据与技能发现**：`SKILL.md` 顶部已内置 YAML frontmatter，包含 `name`、`description`、`version`、`license`、`compatibility`、`triggers` 与 `metadata` 字段。`description` 同时写明能力边界与适用场景，`triggers` 覆盖「机构主力视角分析」「主力在吸筹还是出货」「帮我看看主力成本」等中文说法与英文说法，便于 Agent 在扫描技能列表时完成匹配。若你的运行时只读取特定字段（例如仅 `name` 与 `description`），其余字段可安全忽略。

**数据前提**：本 Skill 不自带行情源。需要至少接入实时行情与历史 K 线工具；资金、筹码、板块和新闻数据越完整，结论越可靠。缺少必要数据时，Agent 应明确标注缺口、降低置信度，而不是编造数据。

<a id="cn-data"></a>

## 数据要求

| 数据类别   | 最低要求                                                                        | 主要用途             |
| ---------- | ------------------------------------------------------------------------------- | -------------------- |
| 实时行情   | 最新价、开高低、昨收、涨跌幅、成交量额、换手率、量比、振幅、盘口                | 当日强弱与交易时机   |
| 历史日 K   | 至少 120 个交易日，尽量 240 个；含 OHLCV、换手率、成交额、复权价                | 结构、量价与指标计算 |
| 长周期历史 | 2～5 年，条件允许可更长                                                         | 相似走势检索与回测   |
| 技术指标   | MA5/10/20/30/60、MACD、KDJ、RSI、BOLL、ATR、OBV、VWAP、量均线                   | 趋势、位置与波动     |
| 资金与筹码 | 超大/大/中/小单、主力净流入、3/5/10 日变化、筹码分布、获利盘、平均成本          | 行为识别与成本推断   |
| 市场与板块 | 上证指数、深证成指、创业板指、行业/概念板块、成交额、涨跌家数、涨跌停、连板高度 | 环境校准             |
| 新闻与公告 | 公司公告、业绩预告、行业政策、机构调研、舆情                                    | 事件冲击识别         |

所有数据必须记录查询时间、数据源和数据周期。盘中分析不得把延迟行情描述为实时行情；数据缺失必须标注，禁止编造行情、资金流向、回测结果和统计概率。

<a id="cn-guardrails"></a>

## 边界与风控

- 行为标签是量价特征的规则化匹配，不是识别到了真实机构账户、订单或意图。
- 资金流向是统计口径，不等同于机构交易流水。
- 主力成本区间是假设性推断，不是真实持仓成本。
- 情景模拟不是预测；历史相似样本的统计频率也不等于未来概率。
- A 股存在 T+1、涨跌停、停牌、跳空、流动性与交易费用约束；盘中最高价不等于可成交的卖出价。
- 收盘后不虚构盘中计划，应明确区分当日复盘与次日预案。
- 数据质量决定置信度上限，不能用低质量数据产生的高分冒充高置信结论。
- 不保证收益，不诱导高风险交易，始终保留「观望 / 不参与」选项。
- 本 Skill 是研究与决策辅助工具，不是自动交易系统，也不构成投资建议。

<a id="cn-extension"></a>

## 工程化扩展

`SKILL.md` 第十节给出了四个建议的 MCP 工具接口，用于把量化计算与 LLM 叙事分离。当前仓库尚未附带这些工具的代码实现，以下为设计蓝图：

| 建议工具                           | 职责                                                          |
| ---------------------------------- | ------------------------------------------------------------- |
| `detect_institutional_behavior`    | 九类行为评分、推测成本区间、支撑压力、证据、置信度、数据质量  |
| `simulate_institutional_scenarios` | 三情景概率、价格区间、推荐策略、风险等级                      |
| `backtest_similar_patterns`        | 相似样本、未来 1/3/5/10 日统计、MFE/MAE、验证标记             |
| `generate_stock_decision`          | 汇总三引擎与市场环境，输出决策、概率、今日/次日计划与风险因素 |

完整字段与示例 JSON 见 `SKILL.md` 第十章。实现时应由工具负责数值计算与概率校准，LLM 负责解释与决策表达，避免多个工具各自输出未经协调的概率。

<a id="cn-structure"></a>

## 目录结构

```text
institutional-investors-perspective/
├── README.md   # 中英双语说明（本文件）
├── SKILL.md    # Skill 主规范：YAML frontmatter、工作流、评分模型、报告模板与执行规则
└── LICENSE     # MIT 许可证
```

`SKILL.md` 是唯一的行为规范；本 README 仅作说明。如两者冲突，以 `SKILL.md` 为准。

<p align="right"><a href="#english">English ↓</a> &nbsp;·&nbsp; <a href="#top">回到顶部 ↑</a></p>

---

<a id="english"></a>

# Institutional Investors' Perspective · Institutional Playbook Skill

> Put the agent in the seat of a dominant institutional holder: identify the current operating stage, reason about the next move, and validate the judgment against history.

This skill is built for single-stock research in the A-share market. When the user supplies a stock, the agent does not stop at “will it rise tomorrow?” It steps into the role of a large institutional holder that already owns a meaningful position and reasons about chips, cost, overhead supply, capital efficiency, and exit liquidity. The result is an actionable plan for today and the next trading day.

It is a **prompt-only workflow**: the payload is just `SKILL.md`—no market-data adapters, scripts, models, or datasets. Quotes, historical bars, fund flow, chip distribution, sector data, and news must come from Skills or MCP tools already installed in the runtime. When data is missing, the agent must flag the gap and lower its confidence.

> **Important boundary:** “Institutional behavior” and “institutional cost” are behavioral inferences and assumptions derived from public market features. They do not identify real institutional accounts, holdings, or trading intent. This project is not investment advice and does not guarantee returns.

**Quick navigation:** [Overview](#en-overview) · [Capabilities](#en-features) · [Workflow](#en-workflow) · [Three engines](#en-engines) · [Joint decision](#en-decision) · [Output report](#en-report) · [Install and use](#en-usage) · [Data requirements](#en-data) · [Limits and controls](#en-guardrails) · [Engineering extensions](#en-extension) · [Repository layout](#en-structure)

<a id="en-overview"></a>

## Overview

Retail investors ask: will this stock rise or fall tomorrow?

An institutional operator asks:

- How many shares might I already control?
- Where is my build-up cost?
- Am I sitting on a profit or a loss?
- How much trapped supply sits above the current price?
- How much capital would a further markup require, and is it worth it?
- Should I accumulate, wash out, mark up, or distribute now?
- If I exit here, is there enough liquidity?

The skill makes the agent reason through these questions and closes the loop with three engines:

1. **Institutional-behavior detection** — what is likely happening now?
2. **Playbook scenario simulation** — what might the operator do next?
3. **Historical similar-pattern backtest** — what actually happened after comparable setups?

<a id="en-features"></a>

## Capabilities

| Capability                          | Question answered                                                                    | Main output                                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| Institutional-behavior detection    | Which operating stage is most likely now?                                            | Match scores for nine behaviors, price-volume/fund/price evidence, confidence |
| Position and cost model             | Where might the cost zone, trapped zone, and exit zone be?                           | Estimated cost range, relative cost position, support/resistance, confidence  |
| Playbook scenario simulation        | Accumulation, washout, markup, high-level turnover, or distribution—which dominates? | Success probability, expected return, downside, composite score               |
| Historical similar-pattern backtest | What happened after comparable structures?                                           | Similar samples, next-day and 3/5/10-day statistics, MFE/MAE, bias checks     |
| Joint decision                      | Do the engines agree? How are conflicts resolved?                                    | 0–100 composite score, conflict notes, confidence adjustment                  |
| Trading plan                        | What should be done today and tomorrow?                                              | Gap-up/flat/gap-down scenarios, triggers, levels, invalidation conditions     |

<a id="en-workflow"></a>

## Workflow

```text
User supplies a stock
    ↓
Resolve ticker and market
    ↓
Discover and call installed market-data Skills / MCPs
    ↓
Quotes / historical bars / fund flow / chips / sector / news
    ↓
Data completeness checks and normalization
    ↓
Institutional-behavior detection engine
    ↓
Institutional position and cost model
    ↓
Playbook scenario simulation engine
    ↓
Historical similar-pattern backtest engine
    ↓
Joint scoring, conflict resolution, probability calibration
    ↓
Today and next-day trading plan
    ↓
Standard research report
```

1. **Resolve the instrument**: map the name or ticker to a code, market, and sector; confirm with the user when ambiguous.
2. **Discover tools**: inspect installed quote, fund-flow, chip, sector, and backtest tools; never assume a tool exists.
3. **Collect data**: at least 120 trading days of daily bars (240 preferred), recording query time, source, and period.
4. **Detect behavior**: score six dimensions—price-volume, volume/turnover, chip structure, fund flow, trend, and sector/market context.
5. **Simulate scenarios**: model five strategies across three market regimes under explicit position and cost assumptions.
6. **Backtest history**: retrieve similar samples from 2–5 years of history, measure forward performance, and control for bias.
7. **Joint decision**: weight the engines, identify conflicts, and calibrate confidence.
8. **Deliver the report**: produce a today/next-day plan in the fixed ten-section format.

If the user provides a real position and cost basis, model it separately; never mix it with the hypothetical institutional position.

<a id="en-engines"></a>

## The three engines

### 1. Institutional-behavior detection

The goal is not to guess intent from candlestick shapes. The engine evaluates price-volume behavior, fund-flow changes, chip turnover, intraday behavior, and market context, then reports how well each behavior matches—not a claim of certainty.

| Behavior            | What it looks for                                |
| ------------------- | ------------------------------------------------ |
| Bottom accumulation | Sustained chip accumulation at low prices        |
| Range accumulation  | Gradual chip collection through sideways trading |
| Probing             | Testing overhead supply and market absorption    |
| Washout             | Clearing short-term float through pullbacks      |
| Shakeout            | Reshaping the chip base through violent swings   |
| Launch              | Pushing price out of a consolidation zone        |
| Main markup wave    | A sustained advancing phase                      |
| High-level turnover | Exchanging chips near highs                      |
| Distribution        | Persistent reduction or exit behavior            |

**Five price-volume states**

| State                              | Possible interpretation                                                                     | Key confirmation                                       |
| ---------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Declining volume, falling price    | Selling pressure fading, wait-and-see, technical pullback, or low turnover during a washout | Does volume expand on a reversal? Is support held?     |
| Expanding volume, rising price     | Active buying, new capital, successful probe, or start of a markup                          | Price location, close strength, follow-through         |
| Expanding volume, falling price    | Panic turnover, trend damage, high-level exit, or news shock                                | Is key support lost? Can it be recovered?              |
| Expanding volume, stalling price   | Heavy overhead supply; risk of high-level distribution                                      | Upper shadows, repeated intraday reversals, weak close |
| Contracting volume, sideways price | Accumulation, waiting for direction, or fading attention                                    | Volatility, whether volume expands again               |

Declining volume on a pullback does not automatically mean a washout, and expanding volume on a rally does not automatically mean a markup. Location, close strength, and persistence matter.

**Anomaly checks**: late-session volume spikes, morning spikes that fade, intraday dives that recover, repeated large trades at key prices, failed breakouts that fall back into the range, consecutive days of abnormal volume, rising price with weak fund-flow indicators, falling price with shrinking volume, high-level long upper shadows, and low-level long lower shadows. Only use order-book or tick data when the data source provides it, and never infer intent from resting orders alone.

**Cost and chip model**: combine volume-heavy price zones, volume distribution by price, stage turnover, OBV, turnover amount versus share count, and chip-distribution data to distinguish a potential build-up zone, cost-dense zone, current price zone, overhead trapped zone, and potential exit zone. Without reliable chip data, confidence must be reduced; never attribute all historical volume to institutions.

**Behavior scoring weights**

| Dimension                 | Weight |
| ------------------------- | ------ |
| Price-volume relationship | 25%    |
| Volume and turnover       | 20%    |
| Chip structure            | 15%    |
| Fund flow                 | 15%    |
| Technical trend           | 15%    |
| Sector and market context | 10%    |

`BehaviorScore = Σ(normalized dimension score × weight)`

Score all nine behaviors from 0 to 100. Report the top behavior as the primary hypothesis, the runner-up, the supporting evidence, the signals that would invalidate the call, and the confidence level.

### 2. Playbook scenario simulation

The core assumption is that the agent already holds a large position. The engine simulates how different operating strategies could unfold and how the operator should adjust.

| Strategy              | Assumption                                            | What to watch                                                                   |
| --------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------- |
| Continue accumulating | The current price still offers build-up value         | Volume contraction, dip absorption, rising price center                         |
| Active washout        | Chips are held, but short-term float is excessive     | Moving-average retests, support recovery, whether a breakdown comes with volume |
| Active markup         | Conditions for a launch are in place                  | Breakout validity, overhead supply, volume follow-through, sector support       |
| High-level turnover   | The stock has already risen; test market absorption   | Whether turnover is healthy or already distribution-like                        |
| Gradual distribution  | Target return reached or the environment deteriorated | Persistent weakness, low-volume rebounds, worsening risk/reward                 |

At least three market regimes must be simulated:

- **Strong rally**: broad market strength, sector gains, higher turnover, better absorption.
- **Range-bound**: limited index movement, sector divergence, declining stock-level volume.
- **Market decline**: broad weakness, sector outflows, weaker absorption.

Prefer existing quantitative or simulation tools. Without them, build simplified models from historical volatility, ATR bands, conditional distributions of similar setups, market-regime conditioning, and Monte Carlo simulation where data permits. Do not let the LLM freely imagine price paths.

**Scenario scoring weights**

| Dimension              | Weight |
| ---------------------- | ------ |
| Expected upside        | 25%    |
| Probability of success | 25%    |
| Downside risk          | 20%    |
| Volume support         | 10%    |
| Sector environment     | 10%    |
| Chip structure         | 10%    |

Output success probability, expected return, potential risk, and composite score for all five strategies. Then answer: if I were the operator, which strategy would I most likely choose, why, and what would make me change course?

### 3. Historical similar-pattern backtest

Rather than asking whether the model “feels” the stock will rise, the engine finds historical cases with similar technical structure, price-volume features, chip state, and market context, then measures what happened next.

**Features compared**

- **Price structure**: 5/10/20/40/60-day returns, distance from swing highs/lows, moving-average alignment, consolidation range, breakout or retest pattern.
- **Volume**: volume ratio, 5/10/20-day average volume, volume change rate, turnover change, turnover-amount change, OBV change.
- **Technical indicators**: MACD, KDJ, RSI, BOLL, ATR, MA slopes, deviation from moving averages.
- **Market context**: index and sector trends, market turnover, advance/decline counts, limit-up/limit-down counts, consecutive-board height, risk appetite.
- **Behavioral features**: volume contraction, volume breakout, stalling on volume, sustained inflows, high-level turnover, suspected accumulation or washout.

**Similarity weights**

| Dimension                 | Weight |
| ------------------------- | ------ |
| Price structure           | 25%    |
| Price-volume relationship | 25%    |
| Technical indicators      | 15%    |
| Market context            | 20%    |
| Behavioral features       | 15%    |

Standardize features with different units before computing weighted distance:

```text
D(X, Y) = Σ wᵢ × distance(xᵢ, yᵢ)
Similarity = 1 / (1 + D)
```

Select the 20–50 most similar historical samples. If too few samples exist, say so; never pad the sample. Statistics include next-day up/sideways/down probabilities, mean and median returns, average maximum gain, average maximum drawdown, 2/3/5/10-day cumulative performance, limit-up ratio, maximum favorable excursion (MFE), and maximum adverse excursion (MAE). Define the sideways threshold in advance—for example, an absolute close-to-close move of no more than 0.5%.

**Quality controls**

- Historical samples must not include data after the current trading day, and current features must not use future data.
- Heavily overlapping observations from adjacent trading days are not independent cases.
- Keep up, sideways, and down outcomes; do not cherry-pick successes.
- Include other stocks in the same sector or market regime, and account for different market phases.
- Run out-of-sample and walk-forward validation; guard against survivorship bias.
- Historical conditional frequencies are not calibrated future probabilities.

<a id="en-decision"></a>

## Joint decision

The three engines must not issue independent conclusions; they are combined.

| Module                              | Weight |
| ----------------------------------- | ------ |
| Institutional-behavior detection    | 30%    |
| Playbook scenario simulation        | 25%    |
| Historical similar-pattern backtest | 30%    |
| Live market context                 | 15%    |

```text
FinalScore = 0.30 × BehaviorScore
           + 0.25 × ScenarioScore
           + 0.30 × HistoricalScore
           + 0.15 × MarketScore
```

All sub-scores are normalized to 0–100. The score supports ranking and decision-making; it is not a probability of gains. Modules may reuse the same features, creating information overlap. Weights should be tuned through out-of-sample validation. When data is missing, renormalize the weights or lower overall confidence.

**Conflict handling**: when the engines disagree, do not simply average them. Examine the evidence behind each conclusion, whether the data timestamps align, whether the sector environment has changed, whether a material news event intervened, and which model fits the current regime. If the conflict cannot be resolved, lower confidence and favor a defensive plan.

<a id="en-report"></a>

## Output report

Every analysis follows this ten-section structure:

1. Basic stock information
2. Current market analysis
3. Institutional-behavior detection result
4. Playbook scenario simulation
5. Historical similar-pattern backtest
6. Joint three-engine assessment
7. If I were the institutional operator, what would I do today?
8. If I were the institutional operator, what would I do tomorrow?
9. Next-day probability forecast
10. Final trading decision

Section 9 requires up, sideways, and down probabilities that sum to 100%, plus expected return ranges, a closing-price range, limit-up probability, and forecast confidence. Section 10 must answer whether adding exposure is appropriate, when to reduce, when to exit, and what the biggest risk is, ending with a one-sentence core judgment.

Reports must distinguish:

- **Observed facts**: quotes, volume, turnover, fund-flow, and sector data.
- **Quantitative inference**: indicators, price-volume signals, behavior match scores.
- **Simulation**: possible paths under explicit assumptions.
- **Historical statistics**: the forward performance of similar samples.

<a id="en-usage"></a>

## Install and use

1. Place this directory in the skills directory configured for the agent runtime so that `SKILL.md` can be loaded. If you use SkillHub, install it by directory slug. Or just tell your agent:

   `Install this skill: https://github.com/DuXingLang/institutional-investors-perspective`
2. Ask for an analysis directly, for example “Analyze 600000 from the institutional operator's perspective” or “Is the operator accumulating or distributing this stock?”
3. Optional inputs: your real cost basis and position size, holding horizon, risk preference, and key price levels. A real position is modeled separately from the hypothetical institutional position.

Example request:

```text
Analyze 600000 from the institutional operator's perspective.
Assume I already hold a large position with a cost range of 10.2–10.8.
Identify the current operating stage and give me a plan for today and tomorrow.
From the operator's perspective, analyze how stock XXXX will trade today and tomorrow.
```

**Metadata and skill discovery**: `SKILL.md` starts with YAML frontmatter declaring `name`, `description`, `version`, `license`, `compatibility`, `triggers`, and `metadata`. The description states both the capability boundary and the situations it applies to, and `triggers` covers Chinese and English phrasings such as “用机构主力视角分析”, “主力在吸筹还是出货”, and “institutional operator perspective”, so an agent scanning the skill list can match the request. Runtimes that read only specific fields—typically `name` and `description`—can safely ignore the rest.

**Data prerequisite**: the skill does not bundle a market-data source. At minimum, connect real-time quote and historical-bar tools; richer fund-flow, chip, sector, and news data improves reliability. When required data is missing, the agent must flag the gap and lower confidence rather than fabricate it.

<a id="en-data"></a>

## Data requirements

| Data category        | Minimum requirement                                                                                                                          | Primary use                           |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Real-time quotes     | Last price, open/high/low, previous close, change, volume/turnover, turnover rate, volume ratio, amplitude, order book                       | Intraday strength and timing          |
| Daily history        | At least 120 trading days, 240 preferred; OHLCV, turnover rate, turnover amount, adjusted prices                                             | Structure, price-volume, indicators   |
| Long history         | 2–5 years, longer when available                                                                                                             | Similar-pattern search and backtests  |
| Technical indicators | MA5/10/20/30/60, MACD, KDJ, RSI, BOLL, ATR, OBV, VWAP, volume MAs                                                                            | Trend, location, volatility           |
| Fund flow and chips  | Super-large/large/medium/small orders, main-force net inflow, 3/5/10-day changes, chip distribution, profitable-position ratio, average cost | Behavior detection and cost inference |
| Market and sectors   | SSE Composite, SZSE Component, ChiNext, industry/concept sectors, turnover, advance/decline, limit-ups/downs, board height                   | Regime calibration                    |
| News and filings     | Company announcements, earnings previews, policy, institutional research, sentiment                                                          | Event-shock detection                 |

Record the query time, source, and period for every dataset. Intraday analysis must not describe delayed quotes as real-time. Missing data must be flagged; never fabricate quotes, fund flows, backtest results, or probabilities.

<a id="en-guardrails"></a>

## Limits and risk controls

- Behavior labels are rule-based matches of price-volume features, not evidence of real institutional accounts, orders, or intent.
- Fund-flow data is an aggregate statistic, not an institutional transaction ledger.
- The inferred cost range is a hypothesis, not the operator's actual cost basis.
- Scenario simulation is not a forecast; historical sample frequencies are not future probabilities.
- A-shares are subject to T+1 settlement, price limits, suspensions, gaps, liquidity constraints, and transaction costs. An intraday high is not necessarily an executable exit price.
- After the close, do not invent intraday plans; separate the day's review from the next-day plan.
- Data quality caps confidence. A high score built on poor data must not be presented as a high-confidence conclusion.
- Do not guarantee returns or encourage high-risk trading; always keep a “wait / no trade” option.
- This skill is a research and decision-support tool, not an automated trading system, and it is not investment advice.

<a id="en-extension"></a>

## Engineering extensions

Section 10 of `SKILL.md` proposes four MCP tool interfaces that separate quantitative computation from LLM narrative. The implementations are not included in this repository; the interfaces are design blueprints:

| Proposed tool                      | Responsibility                                                                                     |
| ---------------------------------- | -------------------------------------------------------------------------------------------------- |
| `detect_institutional_behavior`    | Nine-behavior scores, estimated cost range, support/resistance, evidence, confidence, data quality |
| `simulate_institutional_scenarios` | Three-regime probabilities, price ranges, recommended strategy, risk level                         |
| `backtest_similar_patterns`        | Similar samples, 1/3/5/10-day forward statistics, MFE/MAE, validation flags                        |
| `generate_stock_decision`          | Combine engines and market context into a decision, probabilities, today/next-day plan, and risks  |

Full fields and example JSON are in Section 10 of `SKILL.md`. In an implementation, tools should own numerical computation and probability calibration, while the LLM explains results and communicates decisions—avoiding multiple tools that each emit uncoordinated probabilities.

<a id="en-structure"></a>

## Repository layout

```text
institutional-investors-perspective/
├── README.md   # Bilingual documentation (this file)
├── SKILL.md    # Primary skill specification: YAML frontmatter, workflow, scoring models, report template, execution rules
└── LICENSE     # MIT license
```

`SKILL.md` is the single source of behavioral truth. This README is documentation only; if the two conflict, `SKILL.md` prevails.

<p align="right"><a href="#中文">中文 ↑</a> &nbsp;·&nbsp; <a href="#top">Back to top ↑</a></p>
