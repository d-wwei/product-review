[English](#english) | 中文

# Product Review

一个 AI agent skill，自动操控 iOS/Android 模拟器体验移动端 App，**生成 3-12 个拟人用户独立体验产品**，输出结构化产品报告。

## 解决什么问题

产品体验报告写起来慢、靠人肉点点点、每次换个人写结构都不一样。更关键的是——**一个人的视角永远是有限的**。

这个 skill 让 AI agent 自己操作 App，逐页截图、记录路径，然后生成多个具有不同年龄/职业/认知水平/动机/行为习惯的"拟人用户"，让他们各自体验产品并给出主观反馈。最终汇总多视角洞察，输出标准化报告。

## 四个子命令

```bash
/product-review ue-report 小红书          # 产品体验报告（UX 五层模型）
/product-review manual core WeChat        # 使用说明书（核心功能版）
/product-review manual mece Notion        # 使用说明书（MECE 穷尽版）
/product-review suggestion 抖音           # 功能/体验优化建议（含 persona 模拟）
/product-review 小红书                     # 交互式选择报告类型
```

### Persona 模拟参数

```bash
/product-review suggestion --mode=A 抖音             # 全自主：所有 persona 实际操控 App
/product-review suggestion --mode=B 小红书           # 混合（默认）：核心 persona 操控 + 其余评审
/product-review suggestion --mode=C WeChat           # 全评审：最快，基于探索数据
/product-review suggestion --personas=8 抖音          # 指定 8 个 persona
/product-review suggestion --mode=A --personas=5 抖音 # 组合参数
```

| 子命令 | 输出 | 适合场景 |
|--------|------|----------|
| `ue-report` | 基于用户体验要素五层模型的深度分析报告 | 竞品研究、产品评审 |
| `manual core` | 只覆盖核心/独特功能的精简使用指南 | 快速了解产品、内部培训 |
| `manual mece` | 遍历每一个功能和设置项，MECE 原则不遗漏 | 完整知识库、客服手册 |
| `suggestion` | 拟人用户多视角反馈 + P0/P1/P2 优化建议（含 ROI） | 产品改进决策 |

## 工作原理

```
Step 0-4: 环境检查 → 设备选择 → App 确认 → 模式选择 → 创建工作目录

Step 5: 自主探索
  主 agent ──→ 广度扫描（所有 Tab/入口）
           ──→ subagent 集群（每个功能模块深入体验）
           ──→ 特殊维度检查（首次体验、搜索、边界情况）

Step 5.5: 产品用户画像分析（NEW）
  产品类别识别 → 关键维度确定 → 用户分群矩阵 → 属性正交采样

Step 6: 拟人用户体验模拟（NEW）
  6.0 Persona 档案生成（20+ 属性 + 200 字背景叙事）
  6.1 行为指令集（首次启动/导航/内容消费/交互/情绪触发器）
  6.2 三模式体验执行
      ├─ 模式 A: 每个 persona 独立操控 App（最真实）
      ├─ 模式 B: 3 个核心 persona 操控 + 其余评审（默认）
      └─ 模式 C: 所有 persona 基于探索数据评审（最快）
  6.3 反馈聚合（共识/分群差异/情绪热力图/流失风险/付费分析）

Step 7: 报告生成
  7a 产品体验报告 / 7b 使用说明书 / 7c Persona 加权优化建议
```

## Persona 系统

核心理念：**像真实用户一样多样化、行为、反馈**。

### Persona 档案结构

每个 persona 包含完整的结构化属性：

| 维度 | 内容 |
|------|------|
| 基础属性 | 姓名、年龄、性别、职业（具体描述）、教育、所在地、收入、家庭 |
| 技术认知 | 手机熟练度(1-5)、领域经验(1-5)、同类产品经历、新功能接受度 |
| 动机目标 | 核心动机（具体场景）、期望解决的问题、学习成本预算、付费意愿 |
| 行为模式 | 注意力模式、耐心程度、错误容忍度、探索风格、使用场景、单次时长 |
| 性格特征 | 表达风格、关注重点、评价倾向、社交倾向 |
| 背景叙事 | 200 字个人故事，包含生活状态和产品使用背景 |
| 行为参数 | sentiment_bias, patience_score, tech_confidence 等 |

### 行为指令集

每个 persona 有独立的行为规则：
- **首次启动**：看引导/跳过引导/直接操作
- **导航行为**：逐个Tab探索/直奔目标/搜索优先
- **内容消费**：只看标题/扫描关键词/完整阅读
- **情绪触发器**：什么让 TA 开心/烦躁/惊喜/放弃

### 多样性保障

- 任意两个 persona 至少 3 个核心维度不同
- 必须覆盖技术最低和领域最高的极端用户
- 年龄至少覆盖 3 个区间
- 评价倾向必须包含"严格"和"包容"

### 设计参考

| 项目 | 借鉴点 |
|------|--------|
| [MiroFish](https://github.com/666ghj/MiroFish) | 行为参数化 + 情绪触发器 + 闭环记忆 |
| [DeepPersona](https://arxiv.org/abs/2511.07338) | 百级属性分层生成 + 正交采样 |
| [AgentReviewHub](https://github.com/reynoldw/Agent-Lens) | 浏览器自动化 + 多 persona UX 评测 |
| [A/B Agent](https://github.com/Applied-Machine-Learning-Lab/ABAgent) | 疲劳系统 + 记忆模块 |
| [OASIS](https://github.com/camel-ai/oasis) | 百万级 Agent 社交模拟框架 |

## 前置要求

1. **Claude Code**（CLI、桌面端或 IDE 插件均可）
2. **mobile-mcp**：

```bash
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest
```

3. **模拟器环境**：
   - iOS：Xcode + iOS Simulator（已启动）
   - Android：Android Studio + Emulator（已启动）

## 安装

```bash
# 克隆到 Claude Code skills 目录
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review
```

或手动复制 `SKILL.md` 到 `~/.claude/skills/product-review/SKILL.md`。

## 输出目录结构

```
reviews/
  xiaohongshu/
    20260328/
      screenshots/                    # 全部截图
        persona-1/                    # Persona 自主体验截图（模式 A/B）
        persona-2/
      exploration-log.md              # 探索日志
      persona-profiles.md             # Persona 档案（完整属性 + 背景叙事）
      persona-feedback-synthesis.md   # Persona 反馈综合分析
      official-help-data.md           # 官方帮助中心数据
      report.md                       # 产品体验报告
      manual-core.md                  # 使用说明书 Core 版
      manual-full.md                  # 使用说明书 MECE 版
      optimization.md                 # Persona 加权优化建议
```

## 报告模板说明

### 产品体验报告 (`ue-report`)

基于 Jesse James Garrett 的用户体验要素五层模型：

| 层级 | 分析维度 |
|------|----------|
| 战略层 | 产品目标、用户需求、商业模式推断 |
| 范围层 | 功能清单、内容分析、功能聚焦度 |
| 结构层 | 信息架构图、交互流程、用户流程分析 |
| 框架层 | 界面布局、信息设计、导航设计 |
| 表现层 | 视觉风格、色彩、排版、动效、平台规范 |

### 使用说明书 (`manual`)

- **Core**：3-5 个核心功能 + 独特亮点 + Top 10 FAQ
- **MECE**：功能全景树 + 每个功能穷尽所有操作项/设置项 + 覆盖率报告

### 优化建议 (`suggestion`)

多个拟人用户独立体验/评审产品后：
- 反馈聚合：共识发现、分群差异、情绪热力图、流失风险、付费意愿
- P0/P1/P2 优先级排序，每条建议附带 persona 原声引用和影响面分析
- 按用户群分层的专属建议（新手/核心/专家各自 Top 3）

## 技术细节

- 通过 mobile-mcp 的 24 个工具控制模拟器
- 每次操作前用 `mobile_list_elements_on_screen` 获取坐标
- 每个关键步骤操作前截图保存
- Persona patience_score 控制探索深度（0.0-0.3: 2-5min, 0.8-1.0: 30+min）
- 支持同一产品多次体验的版本对比

## License

MIT

---

<a name="english"></a>

[中文](#) | English

# Product Review

An AI agent skill that autonomously operates iOS/Android simulators to experience mobile apps, **generates 3-12 synthetic user personas who independently test the product**, and outputs structured reports.

## The Problem

Writing product experience reports is slow, manual, and inconsistent. More importantly, **one person's perspective is always limited**.

This skill lets AI agents operate apps themselves, screenshot every page, record paths, then generate multiple "synthetic users" with different ages, professions, expertise levels, motivations, and behavior patterns. Each persona experiences the product independently and provides subjective feedback. Multi-perspective insights are aggregated into standardized reports.

## Four Subcommands

```bash
/product-review ue-report Xiaohongshu       # UX report (5-layer model)
/product-review manual core WeChat           # User manual (core features)
/product-review manual mece Notion           # User manual (exhaustive MECE)
/product-review suggestion TikTok            # Optimization suggestions (with persona sim)
/product-review Xiaohongshu                  # Interactive mode selection
```

### Persona Simulation Parameters

```bash
/product-review suggestion --mode=A TikTok            # Full autonomous: all personas operate app
/product-review suggestion --mode=B Xiaohongshu       # Hybrid (default): core personas operate + rest review
/product-review suggestion --mode=C WeChat            # Review-only: fastest, based on exploration data
/product-review suggestion --personas=8 TikTok         # Specify 8 personas
/product-review suggestion --mode=A --personas=5 TikTok # Combined params
```

| Subcommand | Output | Best For |
|------------|--------|----------|
| `ue-report` | Deep analysis based on UX 5-layer model | Competitive research, product reviews |
| `manual core` | Concise guide covering core/unique features | Quick onboarding, training |
| `manual mece` | Exhaustive docs, every feature and setting, MECE | Knowledge base, support handbook |
| `suggestion` | Multi-persona feedback + P0/P1/P2 recommendations with ROI | Product improvement decisions |

## How It Works

```
Steps 0-4: Environment check → Device → App → Mode → Work directory

Step 5: Autonomous Exploration
  Main agent ──→ Breadth scan (all tabs/entries)
             ──→ Subagent cluster (deep dive per module)
             ──→ Special checks (first launch, search, edge cases)

Step 5.5: Product User Profiling (NEW)
  Category detection → Key dimensions → User segmentation → Orthogonal sampling

Step 6: Persona-Driven User Simulation (NEW)
  6.0 Persona profile generation (20+ attributes + 200-word backstory)
  6.1 Behavior instruction sets (triggers, reactions, attention patterns)
  6.2 Three execution modes
      ├── Mode A: Every persona operates the app independently (most authentic)
      ├── Mode B: 3 core personas operate + rest review (default)
      └── Mode C: All personas review exploration data (fastest)
  6.3 Feedback aggregation (consensus, segments, emotion heatmap, churn risk, willingness to pay)

Step 7: Report Generation
  7a UX report / 7b User manual / 7c Persona-weighted optimization
```

## Persona System

Core principle: **Diverse like real users. Behave like real users. React like real users.**

### Persona Profile Structure

Each persona has structured attributes covering:

| Dimension | Content |
|-----------|---------|
| Demographics | Name, age, gender, profession (specific), education, location, income, family |
| Tech & Domain | Phone proficiency (1-5), domain expertise (1-5), competing apps used, openness to new features |
| Motivation | Core motivation (specific scenario), problems to solve, learning budget, willingness to pay |
| Behavior | Attention pattern, patience level, error tolerance, exploration style, usage context, session length |
| Personality | Expression style, focus area, rating tendency, social tendency |
| Backstory | 200-word narrative with life context and product relationship |
| Parameters | sentiment_bias, patience_score, tech_confidence, etc. |

### Diversity Guarantees

- Any two personas differ in at least 3 core dimensions
- Covers extreme low-tech and domain-expert users
- At least 3 age brackets represented
- Includes both strict and lenient evaluators

### Design References

| Project | What We Borrowed |
|---------|-----------------|
| [MiroFish](https://github.com/666ghj/MiroFish) | Behavior parameterization + emotion triggers + memory loop |
| [DeepPersona](https://arxiv.org/abs/2511.07338) | 100+ attribute taxonomy + orthogonal sampling |
| [AgentReviewHub](https://github.com/reynoldw/Agent-Lens) | Browser automation + multi-persona UX testing |
| [A/B Agent](https://github.com/Applied-Machine-Learning-Lab/ABAgent) | Fatigue system + memory module |
| [OASIS](https://github.com/camel-ai/oasis) | Million-scale agent social simulation framework |

## Prerequisites

1. **Claude Code** (CLI, desktop app, or IDE extension)
2. **mobile-mcp**:

```bash
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest
```

3. **Simulator**:
   - iOS: Xcode + iOS Simulator (booted)
   - Android: Android Studio + Emulator (running)

## Installation

```bash
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review
```

Or manually copy `SKILL.md` to `~/.claude/skills/product-review/SKILL.md`.

## Output Structure

```
reviews/
  xiaohongshu/
    20260328/
      screenshots/                    # All screenshots
        persona-1/                    # Persona autonomous screenshots (mode A/B)
        persona-2/
      exploration-log.md              # Exploration log
      persona-profiles.md             # Persona profiles (full attributes + backstories)
      persona-feedback-synthesis.md   # Persona feedback synthesis
      official-help-data.md           # Official help center data
      report.md                       # UX experience report
      manual-core.md                  # User manual (core)
      manual-full.md                  # User manual (MECE)
      optimization.md                 # Persona-weighted optimization suggestions
```

## Technical Details

- Controls simulators via mobile-mcp's 24 tools
- Uses `mobile_list_elements_on_screen` for coordinates before every tap
- Screenshots before each key operation
- Persona patience_score controls exploration depth (0.0-0.3: 2-5min, 0.8-1.0: 30+min)
- Supports version comparison across multiple reviews

## License

MIT
