[English](#english) | 中文

# Product Review

AI agent 操控模拟器自主体验 App，生成多角色用户独立测试 + 领域专家评审，输出结构化报告。

实测：150+ 功能模块全覆盖 · 540 张截图 · 8 个拟人用户 · 4 个领域专家 · 7 份报告共 6,000+ 行 · 成本 $45

## 一个人的视角永远不够

你写过产品体验报告。一个人点遍所有功能，写下"界面清晰、操作流畅"，发给团队。

问题是：一个 25 岁的产品经理和一个 63 岁的退休教师打开同一个 App，看到的是两个完全不同的产品。前者跳过引导直奔核心功能，后者在设置页面花了 5 分钟找字体大小。你的报告里只有一个人的视角。

人工团队做同样的事需要 3-5 周、$8,000-20,000。覆盖率通常 60-80%——总有人忘了滚动到页面底部、展开那个折叠菜单、测试那个藏在第三层的设置项。

这个 skill 的做法：**AI agent 操控模拟器走遍每一个按钮和入口，然后生成多个背景完全不同的虚拟用户各自体验，再召集领域专家做深度评审。** 你拿到的不是一份报告，是一场完整的用户调研 + 专家咨询。

## 快速开始

```bash
# 1. 安装
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review

# 2. 添加 mobile-mcp（控制模拟器）
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest

# 3. 启动 iOS Simulator 或 Android Emulator，打开 Claude Code：
/product-review suggestion 小红书
```

前置条件：Claude Code + 模拟器（iOS 需 Xcode，Android 需 Android Studio）。

**推荐安装** [macos-desktop-control](https://github.com/d-wwei/macos-desktop-control)：Retina 高清屏截图压缩后传入 LLM，token 节省 95%+。

## 五种报告

```bash
/product-review ue-report 小红书              # UX 五层模型深度分析
/product-review manual core WeChat           # 核心功能使用指南
/product-review manual mece Notion           # MECE 穷尽式完整手册
/product-review suggestion 抖音              # 拟人用户反馈 + 优化建议
/product-review expert-review [App]          # 领域专家深度评审
```

| 命令 | 产出 | 适用场景 |
|------|------|---------|
| `ue-report` | 战略-范围-结构-框架-表现五层分析 + 用户旅程情绪曲线 + 维度加权评分 | 竞品研究、产品决策 |
| `manual core` | 核心竞争力 + 操作指南 + Top 10 FAQ + 快捷速查表 | 快速了解产品、营销材料 |
| `manual mece` | 功能树 + 每个按钮/设置项的穷尽文档 + 配图版 + 覆盖率报告 | 客服知识库、投资者教育、合规文档 |
| `suggestion` | 多 Persona 反馈聚合 + P0/P1/P2 优化建议 + ROI 评估 + 季度路线图 | 产品改进决策 |
| `expert-review` | 多领域专家并行评审 + 工作流缺口 + 功能机会 + 创新洞察 | 专业深度分析 |

可组合：`/product-review manual mece --expert-review [App]` 同时生成说明书和专家评审。

## Persona 模拟

```bash
/product-review suggestion --mode=A 抖音             # 全自主：每个 Persona 独立操控 App
/product-review suggestion --mode=B 小红书           # 混合（默认）：核心 Persona 操控 + 其余评审
/product-review suggestion --mode=C WeChat           # 全评审：最快，基于已有探索数据
/product-review suggestion --personas=8 抖音          # 指定数量（默认按复杂度 3-12 个）
```

| 模式 | 做了什么 | 速度 |
|------|---------|------|
| A 全自主 | 每个 Persona 在模拟器上实际操作，按自己的性格和动机走不同路径 | 最慢 |
| B 混合 | 3 个核心 Persona 实操（新手/目标用户/挑剔专家），其余基于截图评审 | 中等 |
| C 全评审 | 所有 Persona 评估探索阶段的截图和操作记录 | 最快 |

不是"新手用户"三个字。每个 Persona 有 20+ 结构化属性、行为指令集和 200 字背景叙事：

> **Persona #3: 王淑芬**，55 岁，退休小学教师，住在成都。女儿推荐她试试这个 App。她只会用微信和支付宝，App Store 都要女儿远程帮她搜。每次打开 App 不超过 5 分钟，加载超过 3 秒就退出。

多样性靠属性正交采样保证——任意两个 Persona 至少 3 个核心维度不同，极端用户（最低技术水平、最高领域专家）强制覆盖。

实测中 8 个 Persona 的满意度呈现从入门到专业的递减趋势（入门均值 6.0 vs 专家均值 4.5）。这种"能力谷"揭示了产品功能够吸引专业用户，但深度不够留住他们。单人测试永远发现不了这个结构性问题。

## 专家评审

`expert-review` 召集多个领域专家 Agent 并行执行深度评审，产出工作流缺口分析、功能机会评估和创新建议。

实测中 4 个领域专家 11 分钟内完成并行评审，发现了一般探索容易遗漏的专业级问题——某个关键数据面板实际存在但 UI 可发现性极低（用户误以为功能缺失，实际只需前端调整），将修复成本预估从 4-6 周降低到 1-2 周。

内置 20+ 领域专家模板（金融、电商、社交、教育等），也可根据产品特性自动生成定制专家。

## 实测数据

以下数据来自对一款复杂 App 的完整评测（单设备串行，Claude Opus 模型）：

| 指标 | 数值 |
|------|------|
| 功能模块覆盖 | 150+，100% |
| 截图采集 | 540 张 |
| 探索 Agent 组 | 23 组 |
| 拟人用户 | 8 个差异化 Persona |
| 领域专家 | 4 个并行评审 |
| 最终报告 | 7 份，6,000+ 行 |
| 运行时间 | ~13 小时（无人值守） |
| API 成本 | ~$45 |
| 事实准确率 | 98%+（经迭代修正） |

进一步优化空间：
- **分层模型策略**（探索用 Sonnet，报告生成用 Opus）：预计成本降至 ~$27（-40%），探索质量不受影响
- **多设备并行**：2 台 → ~7.5h，3 台 → ~5.5h；两者叠加可实现 **~$27 / ~5.5h** 完成全量评测

| | AI Agent | 人工团队 |
|--|---------|---------|
| 功能覆盖率 | 100% | 60-80% |
| 用户视角 | 8 个差异化 Persona | 1-2 个测试者 |
| 专家评审 | 4 领域并行 | 需外聘 |
| 耗时 | 13 小时 | 3-5 周 |
| 成本 | $45 | $8,000-20,000 |
| 可复现 | 完全标准化 | 因人而异 |

## Experience Bank：用得越多越准

每次 review 结束后，系统自动提取 6 类可复用经验写入 `reviews/_experience_bank/`，带置信度评分。下次 review 同类产品时自动检索注入。

3 次同类 App review 后，系统知道哪些 Persona 维度必须覆盖、哪些功能区域容易被忽略、哪些探索策略浪费时间。增量 review（同一产品新版本）仅需 ~$20、~5 小时。

## 多平台支持

| 平台 | 依赖 | 状态 |
|------|------|------|
| 移动 App（iOS/Android） | [mobile-mcp](https://github.com/anthropics/mobile-mcp) | 必装 |
| 桌面 App（macOS） | [macos-desktop-control](https://github.com/d-wwei/macos-desktop-control) | 推荐（含截图压缩） |
| Web App | browser-control-skill | 可选 |

桌面和 Web 能力缺失时自动降级，不阻断流程。

## 流程全景

```
初始化 → 自主探索 → 用户模拟 → 专家评审 → 报告生成 → 经验沉淀
  |          |           |           |           |           |
  |          |           |           |           |           └─ 提取经验写入 bank
  |          |           |           |           └─ 多份报告分段生成 + 迭代优化
  |          |           |           └─ 多领域专家并行深度评审
  |          |           └─ 3-12 个 Persona 独立体验/评审，反馈聚合
  |          └─ 主 agent 广度扫描 + subagent 集群按复杂度分组深入
  └─ 设备检测 + App 启动 + 触控验证
```

## 技术架构

模块化按需加载。SKILL.md ~200 行路由，详细指令分散在 10 个模块中，执行到对应步骤才加载。

核心机制：
- **完成驱动**：Agent 退出条件是 6 项完成标准全部达标，不是调用次数限制
- **细粒度分组**：按页面结构拆分（如 11 个子模块 → 9 个 Agent），"宁可拆太细，不可合太粗"
- **Fail-Fast-Then-Split**：Agent 失败时立即拆分任务，不重试同样的负载
- **截图优化**：通过 macos-desktop-control 压缩高清截图，token 节省 95%+

## 输出结构

```
reviews/
  _experience_bank/                   # 经验银行（跨产品共享）
  {app_name}/
    {date}/
      screenshots/                    # 截图（含 Persona 专属子目录）
      exploration-reports/            # 探索报告（按 Agent 组）
      screenshot-feature-map.md       # 截图-功能映射速查表
      persona-profiles.md             # Persona 完整档案
      persona-feedback-synthesis.md   # 反馈综合分析
      report.md                       # 产品体验报告
      manual-core.md                  # 核心使用指南
      manual-full.md                  # MECE 完整手册
      expert-review.md                # 专家评审报告
      optimization.md                 # 优化建议
```

## 设计参考

| 项目 | 借鉴 |
|------|------|
| [MiroFish](https://github.com/666ghj/MiroFish) | 行为参数化 + 情绪触发器 + 闭环记忆演化 |
| [DeepPersona](https://arxiv.org/abs/2511.07338) | 百级属性分层采样，44% 更高独特性 |
| [AgentReviewHub](https://github.com/reynoldw/Agent-Lens) | 多 Persona 浏览器自动化 UX 评测 |
| [A/B Agent](https://github.com/Applied-Machine-Learning-Lab/ABAgent) | 疲劳系统控制探索深度 |
| [skill-se-kit](https://github.com/skill-se-kit/skill-se-kit) | 经验银行 + 置信度演化 |

## License

MIT

---

<a name="english"></a>

[中文](#) | English

# Product Review

AI agent skill that operates simulators to test apps autonomously, generates multi-persona user testing + domain expert reviews, outputs structured reports.

Field-tested: 150+ feature modules at 100% coverage · 540 screenshots · 8 synthetic users · 4 domain experts · 7 reports totaling 6,000+ lines · $45 total cost

## One Perspective Is Never Enough

You've written product experience reports. One person clicks through every feature, writes "clean UI, smooth interactions," sends it to the team.

A 25-year-old PM and a 63-year-old retiree open the same app and see two different products. The PM skips onboarding and goes straight to the core feature. The retiree spends 5 minutes looking for font size settings. Your report captures one of those stories.

A human team doing the same work takes 3-5 weeks and $8,000-20,000. Coverage typically hits 60-80% — someone always forgets to scroll to the bottom, expand that collapsed menu, test the setting buried three levels deep.

This skill takes a different approach. **AI agent operates the simulator to visit every button and entry point, then generates multiple virtual users with distinct backgrounds who each experience the product, followed by domain expert deep reviews.** What you get isn't one report. It's a full user study + expert consultation.

## Quick Start

```bash
# 1. Install
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review

# 2. Add mobile-mcp (simulator control)
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest

# 3. Boot iOS Simulator or Android Emulator, open Claude Code:
/product-review suggestion TikTok
```

Prerequisites: Claude Code + simulator (iOS needs Xcode, Android needs Android Studio).

**Recommended:** Install [macos-desktop-control](https://github.com/d-wwei/macos-desktop-control) for Retina screenshot compression — saves 95%+ tokens when passing screenshots to the LLM.

## Five Report Types

```bash
/product-review ue-report [App]              # UX 5-layer deep analysis
/product-review manual core [App]            # Core feature user guide
/product-review manual mece [App]            # Exhaustive MECE documentation
/product-review suggestion [App]             # Persona feedback + optimization
/product-review expert-review [App]          # Domain expert deep review
```

| Command | Output | Use Case |
|---------|--------|----------|
| `ue-report` | Strategy-Scope-Structure-Skeleton-Surface analysis + user journey emotion curves + weighted scoring | Competitive research, product decisions |
| `manual core` | Core strengths + walkthrough + Top 10 FAQ + cheat sheet | Quick product knowledge, marketing material |
| `manual mece` | Full feature tree, every button/setting documented, illustrated edition, coverage report | Support knowledge base, compliance docs |
| `suggestion` | Multi-persona feedback + P0/P1/P2 recommendations + ROI estimates + quarterly roadmap | Product improvement decisions |
| `expert-review` | Multi-domain expert parallel review + workflow gaps + feature opportunities + innovation insights | Professional deep analysis |

Combine them: `/product-review manual mece --expert-review [App]` generates both a manual and expert review.

## Persona Simulation

```bash
/product-review suggestion --mode=A [App]          # Full autonomous: every persona operates the app
/product-review suggestion --mode=B [App]          # Hybrid (default): core personas operate + rest review
/product-review suggestion --mode=C [App]          # Review-only: fastest, uses existing exploration data
/product-review suggestion --personas=8 [App]      # Set persona count (default: 3-12 based on complexity)
```

| Mode | What Happens | Speed |
|------|-------------|-------|
| A | Every persona operates the app on the simulator, following their own motivation and behavior patterns | Slowest |
| B | 3 critical personas operate (newbie + target user + picky expert), rest review screenshots | Medium |
| C | All personas evaluate the exploration data and screenshots | Fastest |

Not "new user" in three words. Each persona carries 20+ structured attributes, a behavior instruction set, and a 200-word backstory:

> **Persona #3: Wang Shufen**, 55, retired elementary school teacher. Her daughter recommended this app. She only knows WeChat and Alipay — can't search the App Store without help over video call. Never spends more than 5 minutes per session. Leaves if anything takes longer than 3 seconds to load.

Diversity is enforced through orthogonal attribute sampling. Any two personas differ on at least 3 core dimensions. Extreme users (lowest tech skill, highest domain expertise) are always represented.

In field testing, 8 personas' satisfaction scores showed a clear drop-off from beginners to experts (beginner average 6.0 vs expert average 4.5). This "competence valley" revealed that the product had enough features to attract power users but not enough depth to retain them. Single-tester reviews never catch structural issues like this.

## Expert Review

`expert-review` dispatches multiple domain expert agents in parallel for deep analysis. Each expert evaluates the product from their specialized perspective, producing workflow gap analysis, feature opportunity assessments, and innovation recommendations.

In field testing, 4 domain experts completed parallel reviews in 11 minutes. They caught a critical issue that general exploration missed — a key data panel that existed but had near-zero discoverability (users assumed the feature was missing, when it only needed a UI adjustment). This reframed the fix from a 4-6 week backend project to a 1-2 week frontend tweak.

20+ domain expert templates built in (finance, e-commerce, social, education, etc.). Custom experts are auto-generated based on product characteristics.

## Field-Tested Numbers

Data from a full review of a complex app (single device, serial execution, Claude Opus model):

| Metric | Value |
|--------|-------|
| Feature modules covered | 150+, 100% |
| Screenshots captured | 540 |
| Exploration agent groups | 23 |
| Synthetic users | 8 differentiated personas |
| Domain experts | 4 parallel reviews |
| Final reports | 7 reports, 6,000+ lines |
| Runtime | ~13 hours (unattended) |
| API cost | ~$45 |
| Factual accuracy | 98%+ (after iterative correction) |

Room for further optimization:
- **Layered model strategy** (Sonnet for exploration, Opus for report generation): estimated cost drops to ~$27 (-40%), exploration quality unaffected
- **Multi-device parallelism**: 2 devices → ~7.5h, 3 devices → ~5.5h. Combined: **~$27 / ~5.5h** for a full review

| | AI Agent | Human Team |
|--|---------|------------|
| Feature coverage | 100% | 60-80% |
| User perspectives | 8 differentiated personas | 1-2 testers |
| Expert review | 4 domains in parallel | Requires external consultants |
| Time | 13 hours | 3-5 weeks |
| Cost | $45 | $8,000-20,000 |
| Reproducibility | Fully standardized | Varies by person |

## Experience Bank: Gets Better With Use

After each review, the system extracts 6 types of reusable lessons into `reviews/_experience_bank/` with confidence scores. Next time you review a similar product, relevant experiences are auto-retrieved and injected.

After 3 reviews of similar apps, the system knows which persona dimensions to cover, which feature areas get missed, which exploration strategies waste time. Incremental reviews (same product, new version) cost ~$20 and ~5 hours.

## Multi-Platform Support

| Platform | Dependency | Status |
|----------|-----------|--------|
| Mobile apps (iOS/Android) | [mobile-mcp](https://github.com/anthropics/mobile-mcp) | Required |
| Desktop apps (macOS) | [macos-desktop-control](https://github.com/d-wwei/macos-desktop-control) | Recommended (includes screenshot compression) |
| Web apps | browser-control-skill | Optional |

Desktop and web capabilities degrade gracefully when absent.

## Pipeline Overview

```
Init → Explore → Personas → Expert Review → Reports → Learn
  |        |          |            |             |         |
  |        |          |            |             |         └─ Extract lessons into bank
  |        |          |            |             └─ Multi-report generation + iterative refinement
  |        |          |            └─ Multi-domain expert parallel deep review
  |        |          └─ 3-12 personas test independently, feedback aggregated
  |        └─ Main agent breadth scan + subagent cluster deep dives by complexity
  └─ Device detection + app launch + touch validation
```

## Architecture

Modular on-demand loading. SKILL.md is ~200 lines of routing. Detailed instructions live in 10 module files, loaded only when the corresponding step executes.

Core mechanisms:
- **Completion-driven**: Agent exits when all 6 completion criteria are met, not when a call budget runs out
- **Fine-grained splitting**: Agents split by page structure (e.g., 11 sub-modules → 9 agents). "Better to split too fine than too coarse"
- **Fail-Fast-Then-Split**: When an agent fails, split the task immediately — never retry the same payload
- **Screenshot optimization**: Retina screenshots compressed via macos-desktop-control before passing to LLM, saving 95%+ tokens

## Output Structure

```
reviews/
  _experience_bank/                   # Experience bank (shared across products)
  {app_name}/
    {date}/
      screenshots/                    # Screenshots (with per-persona subdirs)
      exploration-reports/            # Exploration reports (per agent group)
      screenshot-feature-map.md       # Screenshot-feature lookup table
      persona-profiles.md             # Full persona profiles
      persona-feedback-synthesis.md   # Feedback synthesis
      report.md                       # UX experience report
      manual-core.md                  # Core user guide
      manual-full.md                  # MECE full manual
      expert-review.md                # Expert review report
      optimization.md                 # Optimization suggestions
```

## Design References

| Project | What We Took |
|---------|-------------|
| [MiroFish](https://github.com/666ghj/MiroFish) | Behavior parameterization + emotion triggers + memory loop |
| [DeepPersona](https://arxiv.org/abs/2511.07338) | 100+ attribute taxonomy, 44% higher uniqueness |
| [AgentReviewHub](https://github.com/reynoldw/Agent-Lens) | Multi-persona browser-automated UX testing |
| [A/B Agent](https://github.com/Applied-Machine-Learning-Lab/ABAgent) | Fatigue system for exploration depth control |
| [skill-se-kit](https://github.com/skill-se-kit/skill-se-kit) | Experience bank + confidence evolution |

## License

MIT
