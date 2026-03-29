[English](#english) | 中文

# Product Review

AI agent skill：操控模拟器体验 App，生成 3-12 个拟人用户独立测试，输出结构化报告。

## 一个人的视角永远不够

你写过产品体验报告吗？一个人点遍所有功能，写下"界面清晰、操作流畅"——然后发给团队。

问题是：一个 25 岁的产品经理和一个 55 岁的退休教师打开同一个 App，看到的是两个完全不同的产品。前者跳过引导直奔核心功能，后者卡在注册页面 3 分钟。你的报告里只有一个人的视角。

这个 skill 做的事：**让 AI agent 操控模拟器体验 App，然后生成多个背景完全不同的虚拟用户，让他们各自使用产品、各自吐槽、各自打分。** 一个 55 岁的王大妈会说"这个按钮啥意思"，一个 28 岁的量化交易员会说"下单路径比 IBKR 多了两步"。你拿到的不是一份报告，是一场迷你用户调研。

## 快速开始

```bash
# 1. 安装
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review

# 2. 添加 mobile-mcp（控制模拟器）
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest

# 3. 启动 iOS Simulator 或 Android Emulator，打开 Claude Code，运行：
/product-review suggestion 小红书
```

前置条件：Claude Code + 模拟器环境（iOS 需要 Xcode，Android 需要 Android Studio）。

## 四种报告

```bash
/product-review ue-report 小红书      # UX 五层模型深度分析
/product-review manual core WeChat    # 核心功能使用指南
/product-review manual mece Notion    # MECE 穷尽式完整手册
/product-review suggestion 抖音       # 拟人用户反馈 + 优化建议
```

| 命令 | 你能拿到什么 | 适合谁 |
|------|------------|--------|
| `ue-report` | 战略-范围-结构-框架-表现五层分析，附用户旅程情绪曲线 | 做竞品研究的产品经理 |
| `manual core` | 3-5 个核心功能的操作指南 + Top 10 FAQ | 需要快速了解产品的团队 |
| `manual mece` | 功能树 + 每个按钮/设置项的穷尽文档 + 覆盖率报告 | 客服知识库、合规文档 |
| `suggestion` | 多 persona 反馈聚合 + P0/P1/P2 优化建议 + ROI 评估 | 做产品决策的人 |

## Persona 模拟：三种模式

```bash
/product-review suggestion --mode=A 抖音             # 全自主：每个 persona 独立操控 App
/product-review suggestion --mode=B 小红书           # 混合（默认）：3 个核心 persona 操控 + 其余评审
/product-review suggestion --mode=C WeChat           # 全评审：最快，基于已有探索数据
/product-review suggestion --personas=8 抖音          # 指定 persona 数量（默认按产品复杂度 3-12 个）
```

| 模式 | 做了什么 | 花多久 |
|------|---------|--------|
| A 全自主 | 每个 persona 在模拟器上实际操作 App，按自己的性格和动机走不同路径 | 最长 |
| B 混合 | 3 个最关键的 persona 实操（新手/目标用户/挑剔用户），其余基于截图评审 | 中等 |
| C 全评审 | 所有 persona 看 Step 5 的探索截图和操作记录，给出评价 | 最快 |

## 每个 Persona 长什么样

不是"新手用户"三个字。是一个有名有姓、有故事的人：

> **Persona #3: 王淑芬**，55 岁，退休小学教师，住在成都。女儿在加拿大工作，推荐她用这个 App 买点美股 ETF 养老。她只会用微信和支付宝，App Store 都要女儿远程帮她搜。对"限价单"和"市价单"完全没概念。每次打开 App 不超过 5 分钟，加载超过 3 秒就退出。

每个 persona 有 20+ 结构化属性（年龄、职业、技术水平、领域经验、动机、耐心程度、情绪触发器...）、一份行为指令集（遇到弹窗怎么反应、看到长列表翻几屏）、和一段 200 字背景叙事。

多样性靠属性正交采样保证——任意两个 persona 至少 3 个核心维度不同，极端用户（最低技术、最高领域专家）强制覆盖。

## Experience Bank：用得越多越准

第一次 review 和第十次 review 的质量不应该一样。

每次 review 结束后，系统自动提取 6 类可复用经验：哪些 persona 属性组合产出了最好的洞察、这类 App 哪些功能区域容易被忽略、哪些探索策略浪费了时间。写入 `reviews/_experience_bank/`，带置信度评分。

下次 review 同类产品时，系统检索相关经验注入 persona 生成环节。置信度 >= 0.5 才注入，随验证次数提升，随时间和反例衰减。

效果：金融 App review 了 3 次之后，系统知道"风控规则理解度"是必须覆盖的 persona 维度；社交 App review 了 2 次之后，系统知道"设置页面的隐私选项经常藏在第三层"。Skill 在使用中自然进化。

## 流程全景

```
探索 ──→ 画像 ──→ 模拟 ──→ 报告 ──→ 沉淀
  |         |        |        |        |
  |         |        |        |        └─ 提取经验写入 experience bank
  |         |        |        └─ 生成报告（UX/手册/优化建议）
  |         |        └─ 3-12 个 persona 独立体验/评审，反馈聚合
  |         └─ 识别产品类别，检索历史经验，生成 persona 档案
  └─ 主 agent 广度扫描 + subagent 集群深入每个功能模块
```

## 输出结构

```
reviews/
  _experience_bank/                   # 经验银行（跨产品共享）
    _index.md
    categories/                       #   按品类积累
    personas/                         #   有效组合 + 反模式
    strategies/                       #   探索和报告策略
  xiaohongshu/
    20260329/
      screenshots/                    # 截图（含 persona 专属子目录）
      exploration-log.md              # 探索日志
      persona-profiles.md             # Persona 完整档案
      persona-feedback-synthesis.md   # 反馈综合分析
      report.md                       # 产品体验报告
      optimization.md                 # 优化建议
```

## 设计参考

| 项目 | 借鉴了什么 |
|------|-----------|
| [MiroFish](https://github.com/666ghj/MiroFish) | 行为参数化 + 情绪触发器 + 闭环记忆演化 |
| [DeepPersona](https://arxiv.org/abs/2511.07338) | 百级属性分层采样，44% 更高独特性 |
| [AgentReviewHub](https://github.com/reynoldw/Agent-Lens) | 多 persona 浏览器自动化 UX 评测 |
| [A/B Agent](https://github.com/Applied-Machine-Learning-Lab/ABAgent) | 疲劳系统控制探索深度 |
| [skill-se-kit](https://github.com/skill-se-kit/skill-se-kit) | 经验银行 + 置信度演化（轻量实现） |

## License

MIT

---

<a name="english"></a>

[中文](#) | English

# Product Review

AI agent skill: operates simulators to test apps, generates 3-12 synthetic users who experience the product independently, outputs structured reports.

## One Perspective Is Never Enough

You've written product experience reports before. One person clicks through every feature, writes "clean UI, smooth interactions," and sends it to the team.

The problem: a 25-year-old PM and a 55-year-old retiree open the same app and see two different products. The PM skips onboarding and goes straight to the core feature. The retiree gets stuck on the registration page for 3 minutes. Your report only captures one of those stories.

This skill does something different. **It operates the app on a simulator, then generates multiple virtual users with distinct backgrounds who each use the product on their own terms.** A 55-year-old retired teacher says "what does this button do?" A 28-year-old quant trader says "the order flow takes two more taps than IBKR." What you get isn't one report. It's a mini user study.

## Quick Start

```bash
# 1. Install
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review

# 2. Add mobile-mcp (simulator control)
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest

# 3. Boot iOS Simulator or Android Emulator, open Claude Code, run:
/product-review suggestion TikTok
```

Prerequisites: Claude Code + simulator (iOS needs Xcode, Android needs Android Studio).

## Four Report Types

```bash
/product-review ue-report Xiaohongshu     # UX 5-layer deep analysis
/product-review manual core WeChat        # Core feature user guide
/product-review manual mece Notion        # Exhaustive MECE documentation
/product-review suggestion TikTok         # Persona feedback + optimization
```

| Command | What You Get | Who It's For |
|---------|-------------|-------------|
| `ue-report` | Strategy-Scope-Structure-Skeleton-Surface analysis with user journey emotion curves | PMs doing competitive research |
| `manual core` | 3-5 core feature walkthroughs + Top 10 FAQ | Teams that need quick product knowledge |
| `manual mece` | Complete feature tree, every button and setting documented, coverage report | Support knowledge bases, compliance docs |
| `suggestion` | Multi-persona feedback aggregation + P0/P1/P2 recommendations with ROI | People making product decisions |

## Persona Simulation: Three Modes

```bash
/product-review suggestion --mode=A TikTok          # Full autonomous: every persona operates the app
/product-review suggestion --mode=B Xiaohongshu      # Hybrid (default): 3 core personas operate + rest review
/product-review suggestion --mode=C WeChat           # Review-only: fastest, uses existing exploration data
/product-review suggestion --personas=8 TikTok       # Set persona count (default: 3-12 based on complexity)
```

| Mode | What Happens | Speed |
|------|-------------|-------|
| A | Every persona operates the app on the simulator, following their own motivation and behavior patterns | Slowest |
| B | 3 critical personas operate (newbie + target user + picky expert), rest review screenshots | Medium |
| C | All personas evaluate the Step 5 exploration data and screenshots | Fastest |

## What a Persona Looks Like

Not "new user" in three words. A person with a name and a story:

> **Persona #3: Wang Shufen**, 55, retired elementary school teacher in Chengdu. Her daughter works in Canada and recommended this app for buying US ETFs as retirement savings. She only knows how to use WeChat and Alipay -- can't even search the App Store without her daughter's help over video call. No concept of "limit orders" vs "market orders." Never spends more than 5 minutes per session. Leaves if anything takes longer than 3 seconds to load.

Each persona carries 20+ structured attributes (age, profession, tech proficiency, domain expertise, motivation, patience level, emotion triggers...), a behavior instruction set (how they react to popups, how far they scroll through lists), and a 200-word backstory.

Diversity is enforced through orthogonal attribute sampling. Any two personas differ on at least 3 core dimensions. Extreme users (lowest tech skill, highest domain expertise) are always represented.

## Experience Bank: Gets Better With Use

The first review and the tenth review shouldn't be the same quality.

After each review, the system extracts 6 types of reusable lessons: which persona attribute combinations produced the best insights, which feature areas are easy to miss for this app category, which exploration strategies wasted time. Stored in `reviews/_experience_bank/` with confidence scores.

Next time you review a similar product, the system retrieves relevant experiences and injects them into persona generation. Confidence must be >= 0.5 to qualify. Scores rise with validation, decay with time and counter-evidence.

The result: after 3 finance app reviews, the system knows "regulatory rule comprehension" must be a persona dimension. After 2 social app reviews, it knows "privacy settings are usually buried three levels deep." The skill evolves through use.

## Pipeline Overview

```
Explore ──→ Profile ──→ Simulate ──→ Report ──→ Learn
   |           |           |           |          |
   |           |           |           |          └─ Extract lessons into experience bank
   |           |           |           └─ Generate reports (UX / manual / optimization)
   |           |           └─ 3-12 personas test independently, feedback aggregated
   |           └─ Detect category, retrieve past experiences, generate persona profiles
   └─ Main agent breadth scan + subagent cluster deep dives per module
```

## Output Structure

```
reviews/
  _experience_bank/                   # Experience bank (shared across products)
    _index.md
    categories/                       #   Per-category lessons
    personas/                         #   Effective combos + anti-patterns
    strategies/                       #   Exploration and report strategies
  xiaohongshu/
    20260329/
      screenshots/                    # Screenshots (with per-persona subdirs)
      exploration-log.md              # Exploration log
      persona-profiles.md             # Full persona profiles
      persona-feedback-synthesis.md   # Feedback synthesis
      report.md                       # UX experience report
      optimization.md                 # Optimization suggestions
```

## Design References

| Project | What We Took |
|---------|-------------|
| [MiroFish](https://github.com/666ghj/MiroFish) | Behavior parameterization + emotion triggers + memory loop |
| [DeepPersona](https://arxiv.org/abs/2511.07338) | 100+ attribute taxonomy, 44% higher uniqueness |
| [AgentReviewHub](https://github.com/reynoldw/Agent-Lens) | Multi-persona browser-automated UX testing |
| [A/B Agent](https://github.com/Applied-Machine-Learning-Lab/ABAgent) | Fatigue system for exploration depth control |
| [skill-se-kit](https://github.com/skill-se-kit/skill-se-kit) | Experience bank + confidence evolution (lightweight impl) |

## License

MIT
