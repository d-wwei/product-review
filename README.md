[English](#english) | 中文

# Product Review

一个 AI agent skill，自动操控 iOS/Android 模拟器体验移动端 App，生成结构化产品报告。

## 解决什么问题

产品体验报告写起来慢、靠人肉点点点、每次换个人写结构都不一样。这个 skill 让 AI agent 自己操作 App，逐页截图、记录路径、模拟多种用户视角，最后输出标准化报告。一个 App 的完整体验报告，从启动到交付，全程自动。

## 四个子命令

```bash
/product-review ue-report 小红书          # 产品体验报告（UX 五层模型）
/product-review manual core WeChat        # 使用说明书（核心功能版）
/product-review manual mece Notion        # 使用说明书（MECE 穷尽版）
/product-review suggestion 抖音           # 功能/体验优化建议
/product-review 小红书                     # 交互式选择报告类型
```

| 子命令 | 输出 | 适合场景 |
|--------|------|----------|
| `ue-report` | 基于用户体验要素五层模型（战略→范围→结构→框架→表现）的深度分析报告 | 竞品研究、产品评审 |
| `manual core` | 只覆盖核心/独特功能的精简使用指南 | 快速了解产品、内部培训 |
| `manual mece` | 遍历每一个功能和设置项，MECE 原则不遗漏 | 完整知识库、客服手册 |
| `suggestion` | 多用户角色模拟反馈 + 优化建议优先级排序（含 ROI 评估） | 产品改进决策 |

## 工作原理

```
主 agent                              subagent 集群
  │                                      │
  ├─ 广度扫描：遍历所有 Tab/入口  ──────→ 每个功能模块派发一个 subagent 深入体验
  ├─ 梳理产品信息架构                      ├─ 截图 + 记录操作路径
  ├─ 检索官方帮助中心（补充 FAQ）           ├─ 评价交互/视觉/性能
  │                                      └─ 返回结构化探索报告
  ├─ 汇总所有 subagent 发现
  ├─ 派发用户角色模拟 subagent  ──────→ 新手用户 / 活跃用户 / 专业用户 ...
  └─ 生成最终报告                          └─ 各角色视角的体验反馈
```

**探索策略**：先广度（遍历所有主入口），再深度（逐个功能模块深入）。主 agent 只负责协调和汇总，保持上下文干净。

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

把 skill 目录放到 Claude Code 的 skills 路径下：

```bash
# 克隆到 Claude Code skills 目录
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review
```

或手动复制 `SKILL.md` 到 `~/.claude/skills/product-review/SKILL.md`。

## 输出目录结构

每次体验按产品名 + 日期独立存储：

```
reviews/
  xiaohongshu/
    20260328/
      screenshots/           # 全部截图（按模块-步骤-描述命名）
      exploration-log.md     # 探索日志
      official-help-data.md  # 官方帮助中心检索数据
      report.md              # 产品体验报告
      manual-core.md         # 使用说明书 Core 版
      manual-full.md         # 使用说明书 MECE 版
      optimization.md        # 优化建议报告
```

## 报告模板说明

### 产品体验报告 (`ue-report`)

基于 Jesse James Garrett 的用户体验要素五层模型，从抽象到具体逐层分析：

| 层级 | 分析维度 |
|------|----------|
| 战略层 | 产品目标、用户需求、商业模式推断 |
| 范围层 | 功能清单、内容分析、功能聚焦度 |
| 结构层 | 信息架构图、交互流程、用户流程分析 |
| 框架层 | 界面布局、信息设计、导航设计 |
| 表现层 | 视觉风格、色彩、排版、动效、平台规范 |

额外包含：五层贯穿分析（层间一致性检查）、用户体验地图（含情绪曲线）。

### 使用说明书 (`manual`)

两种模式：
- **Core**：3-5 个核心功能 + 独特亮点 + Top 10 FAQ
- **MECE**：功能全景树 + 每个功能穷尽所有操作项/设置项 + 覆盖率报告

FAQ 来源三个渠道：官方帮助中心检索、应用商店用户评价、产品体验发现。

### 优化建议 (`suggestion`)

派发多个 subagent 模拟不同用户角色，汇总反馈后按 P0/P1/P2 优先级排序，每条建议包含：现状、问题、方案、预期收益、实现成本、ROI 评估。

## 技术细节

- 通过 mobile-mcp 的 24 个工具控制模拟器（截图、元素列表、点击、滑动、输入等）
- 每次操作前先用 `mobile_list_elements_on_screen` 获取坐标，不猜测坐标
- 每个关键步骤操作前截图保存当前状态
- 报告语言跟随用户设置（默认中文）
- 支持同一产品多次体验的版本对比（按日期独立存储）

## License

MIT

---

<a name="english"></a>

[中文](#) | English

# Product Review

An AI agent skill that autonomously operates iOS/Android simulators to experience mobile apps and generate structured product reports.

## The Problem

Writing product experience reports is slow, manual, and inconsistent. Different people produce different structures every time. This skill lets the AI agent operate the app itself, screenshot every page, record navigation paths, simulate multiple user perspectives, and output standardized reports. Full app review from launch to delivery, fully automated.

## Four Subcommands

```bash
/product-review ue-report Xiaohongshu       # UX report (5-layer model)
/product-review manual core WeChat           # User manual (core features)
/product-review manual mece Notion           # User manual (exhaustive MECE)
/product-review suggestion TikTok            # Optimization suggestions
/product-review Xiaohongshu                  # Interactive mode selection
```

| Subcommand | Output | Best For |
|------------|--------|----------|
| `ue-report` | Deep analysis based on UX Elements 5-layer model (Strategy→Scope→Structure→Skeleton→Surface) | Competitive research, product reviews |
| `manual core` | Concise guide covering only core/unique features | Quick onboarding, internal training |
| `manual mece` | Exhaustive documentation of every feature and setting, MECE principle | Knowledge base, support handbook |
| `suggestion` | Multi-persona feedback simulation + prioritized optimization recommendations with ROI | Product improvement decisions |

## How It Works

```
Main agent                                Subagent cluster
  │                                          │
  ├─ Breadth scan: visit all tabs/entries ──→ One subagent per feature module
  ├─ Map product information architecture     ├─ Screenshot + record paths
  ├─ Search official help center (for FAQ)    ├─ Evaluate interaction/visual/performance
  │                                          └─ Return structured exploration report
  ├─ Aggregate all subagent findings
  ├─ Dispatch persona simulation agents ───→ New user / Power user / Pro user ...
  └─ Generate final report                    └─ Experience feedback per persona
```

**Exploration strategy**: breadth first (all main entry points), then depth (each module in detail). Main agent only coordinates and aggregates, keeping its context clean.

## Prerequisites

1. **Claude Code** (CLI, desktop app, or IDE extension)
2. **mobile-mcp**:

```bash
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest
```

3. **Simulator environment**:
   - iOS: Xcode + iOS Simulator (booted)
   - Android: Android Studio + Emulator (running)

## Installation

Place the skill directory under Claude Code's skills path:

```bash
git clone https://github.com/d-wwei/product-review.git ~/.claude/skills/product-review
```

Or manually copy `SKILL.md` to `~/.claude/skills/product-review/SKILL.md`.

## Output Structure

Each review is stored independently by product name + date:

```
reviews/
  xiaohongshu/
    20260328/
      screenshots/           # All screenshots (module-step-description naming)
      exploration-log.md     # Exploration log
      official-help-data.md  # Official help center data
      report.md              # Product experience report
      manual-core.md         # User manual (core)
      manual-full.md         # User manual (MECE)
      optimization.md        # Optimization suggestions
```

## Report Templates

### UX Report (`ue-report`)

Based on Jesse James Garrett's Elements of User Experience, analyzing layer by layer:

| Layer | Analysis Dimensions |
|-------|--------------------|
| Strategy | Product goals, user needs, business model inference |
| Scope | Feature inventory, content analysis, feature focus |
| Structure | Information architecture diagram, interaction flows, user journey analysis |
| Skeleton | Interface layout, information design, navigation design |
| Surface | Visual style, color, typography, motion, platform compliance |

Includes: cross-layer consistency check and user journey maps with emotion curves.

### User Manual (`manual`)

Two modes:
- **Core**: 3-5 key features + unique highlights + Top 10 FAQ
- **MECE**: Full feature tree + exhaustive operations/settings per feature + coverage report

FAQ sourced from three channels: official help center, app store reviews, hands-on exploration.

### Optimization Suggestions (`suggestion`)

Multiple subagents simulate different user personas. Feedback is aggregated and prioritized as P0/P1/P2. Each recommendation includes: current state, problem, proposed solution, expected benefit, implementation cost, and ROI assessment.

## Technical Details

- Controls simulators via mobile-mcp's 24 tools (screenshot, element listing, tap, swipe, type, etc.)
- Uses `mobile_list_elements_on_screen` for coordinates before every tap (no guessing)
- Screenshots saved before each key operation
- Report language follows user settings (default: Chinese)
- Supports version comparison across multiple reviews of the same product (stored by date)

## License

MIT
