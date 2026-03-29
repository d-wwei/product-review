---
name: product-review
description: |
  Use when the user asks to experience, review, or evaluate a mobile app.
  Trigger phrases: "体验一下 [App]", "写一份 [App] 的产品体验报告",
  "测评 [App]", "product review", "review this app".
  Subcommands: ue-report, manual core, manual mece, suggestion.
  Requires mobile-mcp (@mobilenext/mobile-mcp) for iOS/Android simulator control.
  Works with any AI agent that supports MCP (Claude Code, Cursor, Windsurf, Codex, etc.).
type: interactive
argument-hint: "[ue-report | manual core | manual mece | suggestion] [App名称]"
---

# Product Review

通用 AI agent skill，自主体验移动端产品，撰写结构化产品体验报告。

通过 mobile-mcp 控制 iOS/Android 模拟器，系统性浏览 App 的每个功能，
截图记录，最终输出专业的产品体验报告、使用说明书或优化建议。

适用于任何支持 MCP 的 AI agent 平台（Claude Code, Cursor, Windsurf, Codex, Cline 等）。

### agent 能力要求

本 skill 需要 agent 具备以下能力（大多数主流 agent 平台均支持）：

| 能力 | 用途 | 必需 |
|------|------|------|
| MCP 工具调用 | 调用 mobile-mcp 控制模拟器 | 是 |
| 文件读写 | 保存截图、生成报告 | 是 |
| Shell 命令执行 | 创建目录、检查环境 | 是 |
| 子 agent 派发 | 并行探索功能模块、模拟用户角色 | 推荐（无则主 agent 串行执行） |
| Web 搜索 / 网页抓取 | 检索官方帮助中心补充 FAQ | 推荐（无则跳过） |
| 用户交互询问 | 选择模式、确认目标 App | 推荐（无则使用子命令参数） |

## 子命令

| 命令 | 说明 | 输出文件 |
|------|------|----------|
| `/product-review ue-report [App]` | 产品体验报告（UX 五层模型） | `report.md` |
| `/product-review manual core [App]` | 使用说明书 Core 模式（核心功能） | `manual-core.md` |
| `/product-review manual mece [App]` | 使用说明书详尽模式（MECE 穷尽） | `manual-full.md` |
| `/product-review suggestion [App]` | 功能/体验优化建议 | `optimization.md` |
| `/product-review [App]` | 无子命令时，交互式选择报告类型 | 按选择生成 |

**示例：**
```
/product-review ue-report 小红书
/product-review manual core WeChat
/product-review manual mece Notion
/product-review suggestion 抖音
/product-review 小红书              → 弹出选择菜单
```

## 子命令路由

解析 `$ARGUMENTS`（skill 接收到的参数字符串）：

```
参数解析规则：
1. 提取第一个词作为子命令候选
2. 如果是 "ue-report" → 模式 = UE_REPORT，剩余参数 = App 名称
3. 如果是 "manual"  → 读取第二个词：
   - "core" → 模式 = MANUAL_CORE，剩余参数 = App 名称
   - "mece" → 模式 = MANUAL_MECE，剩余参数 = App 名称
   - 其他   → 模式 = MANUAL_CORE（默认），第二个词起为 App 名称
4. 如果是 "suggestion" → 模式 = SUGGESTION，剩余参数 = App 名称
5. 其他情况 → 模式 = INTERACTIVE，整个参数 = App 名称
```

**模式与探索深度的映射：**

| 模式 | 探索深度 | 报告类型 |
|------|----------|----------|
| UE_REPORT | 完整体验（广度+深度） | 产品体验报告 |
| MANUAL_CORE | 核心功能体验 | 使用说明书 Core |
| MANUAL_MECE | 完整体验（穷尽所有功能） | 使用说明书 MECE |
| SUGGESTION | 完整体验 + 用户角色模拟 | 优化建议 |
| INTERACTIVE | 用户选择 | 用户选择 |

当模式 = INTERACTIVE 时，进入 Step 3 的交互式选择。
其他模式跳过 Step 3，直接按对应配置执行。

---

## Step 0: 环境检查

确认 mobile-mcp 可用。尝试调用 `mobile_list_available_devices`：
- 如果调用成功（返回设备列表）→ mobile-mcp 已配置，继续。
- 如果调用失败或工具不存在 → mobile-mcp 未配置。

**如果 mobile-mcp 未检测到：**

告诉用户：

> mobile-mcp 未检测到。请根据你使用的 agent 平台安装：
>
> **Claude Code：**
> ```bash
> claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest
> ```
>
> **Cursor / Windsurf / 其他支持 MCP 的 IDE：**
> 在 MCP 配置文件中添加：
> ```json
> {
>   "mcpServers": {
>     "mobile-mcp": {
>       "command": "npx",
>       "args": ["-y", "@mobilenext/mobile-mcp@latest"]
>     }
>   }
> }
> ```
>
> **Codex / 其他 CLI agent：**
> 参考对应平台的 MCP server 配置文档，添加 `@mobilenext/mobile-mcp`。
>
> **前置要求**：Node.js v22+，iOS 需要 Xcode CLI tools，Android 需要 Platform Tools。
> 安装后重启 agent 再试。

**停止执行，等用户完成安装。**

---

## Step 1: 获取设备信息

调用 mobile-mcp 工具获取可用设备：

```
mobile_list_available_devices
```

展示设备列表，让用户选择目标设备。记录：
- 设备 ID（后续所有操作都需要 `device` 参数）
- 设备名称和型号
- 平台（iOS / Android）
- 系统版本

获取屏幕尺寸：

```
mobile_get_screen_size  device: <device_id>
```

---

## Step 2: 确认目标 App

用 AskUserQuestion 询问：

1. **目标 App 名称**（如果用户还没说明）
2. **App 的 bundle ID / package name**（可以先列出已安装 App 让用户选）

```
mobile_list_apps  device: <device_id>
```

从返回列表中匹配目标 App。如果未安装，提示用户先安装。

启动 App：

```
mobile_launch_app  device: <device_id>  packageName: <package_name>
```

截图确认启动成功：

```
mobile_take_screenshot  device: <device_id>
```

---

## Step 3: 选择模式（仅 INTERACTIVE 模式）

如果子命令路由已确定模式，跳过此步。

否则，用 AskUserQuestion 让用户选择：

### 探索模式

| 模式 | 说明 |
|------|------|
| **完整体验** | 遍历所有功能，穷尽所有特性，不重不漏 |
| **指定体验** | 只体验用户指定的某个功能模块 |

### 报告类型

| 子命令 | 类型 | 说明 |
|--------|------|------|
| `ue-report` | 产品体验报告 | UX 五层模型分析，含用户体验地图 |
| `manual core` | 使用说明书 Core | 只覆盖核心/重点/独特功能 |
| `manual mece` | 使用说明书 MECE | 穷尽每一个功能，不遗漏 |
| `suggestion` | 优化建议 | 多用户视角，含收益分析和优先级 |

可多选。选择后按对应模式执行。

---

## Step 4: 创建工作目录

```bash
APP_NAME="<app_name_lowercase>"  # 英文小写，连字符分隔
DATE=$(date +%Y%m%d)
VERSION="<app_version_if_available>"
BASE_DIR="/Users/admin/Documents/AI/product-experience-review/reviews/${APP_NAME}/${DATE}"
SCREENSHOTS_DIR="${BASE_DIR}/screenshots"
mkdir -p "$SCREENSHOTS_DIR"
echo "工作目录: $BASE_DIR"
```

目录结构：

```
reviews/
  <app-name>/
    <YYYYMMDD>/
      screenshots/          # 所有截图
        01-launch.png
        02-home-tab1.png
        ...
      report.md             # 产品体验报告
      manual.md             # 使用说明书（如选择）
      optimization.md       # 优化建议（如选择）
      exploration-log.md    # 探索日志（主 agent 汇总）
```

---

## Step 5: 自主探索阶段

### 探索策略：先广度，再深度

**主 agent 职责：**
1. 梳理产品整体结构（Tab 数量、主入口、导航架构）
2. 每遇到一个子功能模块，派生一个 subagent 去深入体验
3. 汇总所有 subagent 的发现
4. 保持自身上下文干净

### 5.1 广度扫描（主 agent 执行）

先截图并列出当前屏幕元素：

```
mobile_take_screenshot       device: <device_id>
mobile_list_elements_on_screen  device: <device_id>
```

识别主导航结构：
- 底部 Tab Bar（iOS）/ Bottom Navigation（Android）
- 侧边栏 / 汉堡菜单
- 顶部 Tab
- 其他入口

**遍历每个主入口**，对每个入口：
1. 点击进入
2. 截图 + 列出元素
3. 记录该入口下的子功能列表
4. 返回主页

```
mobile_click_on_screen_at_coordinates  device: <device_id>  x: <x>  y: <y>
mobile_take_screenshot                 device: <device_id>
mobile_list_elements_on_screen         device: <device_id>
mobile_save_screenshot                 device: <device_id>  saveTo: <path>
```

广度扫描完成后，输出产品结构大纲到 `exploration-log.md`。

### 5.2 深度探索（subagent 执行）

对每个子功能模块，派生一个 subagent：

```python
# subagent dispatch 模式（伪代码）
Agent(
    prompt="""
    你是一个产品体验测试员。你的任务是深入体验以下功能模块：

    ## 目标模块
    - 模块名称: {module_name}
    - 入口路径: {entry_path}（从首页开始的点击路径）
    - 设备 ID: {device_id}
    - 截图保存目录: {screenshots_dir}

    ## 你要做的事
    1. 按照入口路径导航到目标模块
    2. 体验该模块的所有子功能
    3. 每个操作前先截图记录当前状态（mobile_save_screenshot）
    4. 用 mobile_list_elements_on_screen 获取元素信息，优先于纯截图分析
    5. 记录以下内容：
       - 功能描述
       - 操作路径（每一步点击了什么）
       - 页面切换是否流畅
       - 是否有 loading 状态、toast 提示、动画反馈
       - 空状态处理
       - 错误处理
       - 视觉风格是否与整体一致
       - 任何异常或亮点

    ## 操作方法
    - 点击: mobile_click_on_screen_at_coordinates
    - 滑动: mobile_swipe_on_screen (direction: up/down/left/right)
    - 输入: mobile_type_keys
    - 返回: mobile_press_button (button: BACK, Android) 或找返回按钮点击
    - 截图: mobile_save_screenshot
    - 元素列表: mobile_list_elements_on_screen

    ## 截图命名规则
    {screenshot_prefix}-{step_number}-{description}.png
    例: settings-01-main-page.png, settings-02-account-detail.png

    ## 输出格式
    返回一份 markdown 格式的探索报告，包含：
    - 功能概述
    - 详细操作路径（每步操作 + 截图文件名）
    - 体验发现（亮点和问题）
    - 交互设计评价
    - 视觉设计评价
    - 性能感知
    """,
    description="探索 {module_name} 模块"
)
```

**重要：尽量并行派发 subagent。** 如果多个功能模块之间没有依赖关系（大多数情况），
同时派发多个 subagent 来加速探索。

### 5.3 特殊维度检查

广度和深度探索之外，单独检查：

**首次启动体验：**
- 有没有引导页/教程
- 权限请求是否合理（时机、说明）
- 登录/注册流程

**搜索功能：**
- 搜索入口是否明显
- 搜索建议/热词
- 搜索结果展示
- 空搜索结果处理

**边界情况：**
- 断网提示（如果可以模拟）
- 空列表/空状态页面
- 长文本/特殊字符输入
- 快速连续点击

---

## Step 6: 用户视角模拟（subagent）

派生多个 subagent，每个模拟一类典型用户：

```python
# 根据产品类型确定用户角色
# 例：社交 App 可能有 "新用户"、"活跃用户"、"内容创作者"
# 例：电商 App 可能有 "浏览型用户"、"比价用户"、"老客户"

Agent(
    prompt="""
    你是 {persona_name}（{persona_description}）。

    你正在使用 {app_name}。以下是产品功能概览：
    {product_structure_summary}

    请以你的角色视角，评价以下功能的体验：
    {feature_list_with_screenshots}

    关注点：
    - 这个功能对你来说重要吗？为什么？
    - 使用路径是否符合你的直觉？
    - 有什么让你困惑或不满的地方？
    - 有什么让你觉得惊喜的地方？
    - 你会给这个功能打几分（1-10）？

    输出格式：
    ## {persona_name} 的体验反馈

    ### 整体印象
    [一段话总结]

    ### 各功能评价
    | 功能 | 评分 | 评价 | 改进建议 |
    |------|------|------|----------|
    | ... | ... | ... | ... |

    ### 最想要的改进（Top 3）
    1. ...
    2. ...
    3. ...
    """,
    description="模拟 {persona_name} 视角评价"
)
```

---

## Step 7: 报告生成

### 7a. 产品体验报告（基于 UX 五层模型）

写入 `{BASE_DIR}/report.md`：

```markdown
# {App 名称} 产品体验报告

## 基本信息
- 体验平台：{iOS/Android}
- 体验设备：{设备名称}
- 屏幕尺寸：{width} x {height}
- App 版本：{version，如果能获取}
- 体验日期：{YYYY-MM-DD}
- 体验模式：{完整体验/指定体验}

---

## 一、第一印象
- **启动速度**：{描述}
- **引导流程设计**：{描述}
- **视觉风格概述**：{描述}

---

## 二、核心功能体验

### 2.1 {功能名称}
- **功能描述**：{做什么的}
- **操作路径**：首页 → {Tab} → {按钮} → {页面}
- **体验感受**：
  - 流畅度：{评价}
  - 易用性：{评价}
  - 符合直觉程度：{评价}
- **截图参考**：![{描述}](screenshots/{filename}.png)

### 2.2 {功能名称}
...（对每个核心功能重复）

---

## 三、交互设计评价
- **导航架构**：{是否清晰，层级是否合理}
- **手势交互**：{滑动、长按等是否自然}
- **反馈机制**：{loading、toast、动画是否及时恰当}
- **错误处理和空状态**：{是否有友好的错误提示和空状态设计}

---

## 四、UI/视觉设计评价
- **整体风格一致性**：{评价}
- **排版与留白**：{评价}
- **色彩运用**：{评价}
- **图标与插画质量**：{评价}
- **平台规范符合度**：{iOS HIG / Material Design 合规性}

---

## 五、性能感知
- **页面切换流畅度**：{评价}
- **列表滚动性能**：{评价}
- **图片加载速度**：{评价}

---

## 六、亮点与不足

### 亮点
- {值得学习的设计或功能}

### 不足
| 问题描述 | 严重程度 | 改进建议 |
|----------|----------|----------|
| {描述} | 高/中/低 | {建议} |

---

## 七、竞品对比（如适用）
- 与同类产品的差异化分析

---

## 八、总结评分

| 维度 | 评分(1-10) | 说明 |
|------|-----------|------|
| 功能完整度 | {分} | {说明} |
| 易用性 | {分} | {说明} |
| 视觉设计 | {分} | {说明} |
| 性能体验 | {分} | {说明} |
| 创新性 | {分} | {说明} |
| **综合评分** | **{分}** | {总结} |

---

## 九、用户体验地图

（对核心用户旅程，画出 text-based 体验地图）

### {核心路径名称}

| 阶段 | 用户行为 | 触点 | 情绪 | 痛点 | 机会点 |
|------|----------|------|------|------|--------|
| 发现 | {行为} | {触点} | {😊😐😟} | {痛点} | {机会} |
| 进入 | ... | ... | ... | ... | ... |
| 使用 | ... | ... | ... | ... | ... |
| 完成 | ... | ... | ... | ... | ... |
| 留存 | ... | ... | ... | ... | ... |

---

## 附录：探索路径截图
（按顺序展示所有截图及说明）

| 序号 | 截图 | 操作说明 |
|------|------|----------|
| 1 | ![](screenshots/01-launch.png) | App 启动画面 |
| 2 | ![](screenshots/02-home.png) | 首页主界面 |
| ... | ... | ... |
```

### 7b. 产品使用说明书

**设计原则**（参考 Stripe Docs / Notion Help / Apple User Guide）：
- **任务驱动**：按"用户要完成什么"组织，不按功能列表堆砌
- **渐进式披露**：先快速上手，再逐步深入
- **结论优先**：每个章节开头说明能解决什么问题
- **视觉优先**：每个步骤配带标注的手机截图
- **官方信息补充**：FAQ 和排错部分必须检索官方帮助中心，不能只靠产品体验推测

#### 7b-前置：官方帮助中心信息检索

在生成使用说明书之前，必须检索产品的官方帮助信息来补充内容。
派发 subagent 执行：

```python
Agent(
    prompt="""
    你是一个产品文档研究员。搜索并整理 {app_name} 的官方帮助文档。

    ## 任务
    1. 用 WebSearch 搜索：
       - "{app_name} 帮助中心" / "{app_name} help center FAQ"
       - "{app_name} 使用教程 入门" / "{app_name} 常见问题"
       - "{app_name} troubleshooting"

    2. 找到官方帮助中心 URL 后，用 WebFetch 抓取：
       - 帮助中心首页（获取分类和热门文章列表）
       - 前 10 篇热门帮助文章
       - FAQ 页面 / 新手引导页面

    3. 搜索 App Store / Google Play：
       - 官方功能描述 / 最新版本更新说明
       - 用户评价中反复出现的问题（低分评价）

    ## 输出
    ### 官方帮助中心
    - URL: {地址}
    - 分类结构: {所有分类}
    - 热门文章: {标题 + 摘要}

    ### 官方 FAQ
    | 问题 | 官方解答摘要 | 来源 URL |
    |------|-------------|----------|

    ### 用户高频问题（应用商店评价）
    | 问题 | 出现频率 | 是否有官方解答 |
    |------|----------|---------------|
    """,
    description="检索 {app_name} 官方帮助文档"
)
```

检索结果保存到 `{BASE_DIR}/official-help-data.md`，作为使用说明书的补充素材。

#### 7b-模式选择

用 AskUserQuestion 让用户选择说明书模式：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **Core 模式** | 只覆盖核心/重点/独特功能，精简高效 | 快速了解产品、内部培训、产品简介 |
| **详尽模式 (MECE)** | 遍历每一个功能和特性，不遗漏任何一点 | 完整知识库、客服参考手册、合规文档 |

---

#### 7b-A：Core 模式使用说明书

**覆盖范围**：只写核心功能 + 独特卖点 + 最常用场景。
**筛选标准**：
- 产品的核心价值流程（没有这个功能产品就不成立）
- 独特/差异化功能（竞品没有的）
- 高频使用功能（用户每天都会用的）
- 新手最可能卡住的地方

写入 `{BASE_DIR}/manual-core.md`：

```markdown
# {App 名称} 使用指南（核心版）

> **版本**：{版本} | **平台**：{iOS/Android} | **更新日期**：{YYYY-MM-DD}
> **官方帮助中心**：{URL}

---

## 产品简介
{一句话核心价值} + {适合谁用}

---

## 快速上手（2 分钟）

> 按照以下 {N} 步，快速体验产品核心功能。

### Step 1: {动作}
{说明}
![{描述}](screenshots/quickstart-01.png)

### Step 2: {动作}
...

### Step 3: {动作}
...

**你已经完成了 {App} 的核心操作。**

---

## 主界面导览

![主界面标注图](screenshots/home-annotated.png)

| 区域 | 功能 |
|------|------|
| {区域 A} | {功能说明} |
| {区域 B} | {功能说明} |

---

## 核心功能

### {核心功能 1}：{用户想完成什么}

**为什么重要**：{这是产品的核心价值所在}

| 步骤 | 操作 | 截图 |
|------|------|------|
| 1 | {操作} | ![](screenshots/core1-01.png) |
| 2 | {操作} | ![](screenshots/core1-02.png) |

**小贴士**：{效率技巧}

### {核心功能 2}：{用户想完成什么}
...（只覆盖 3-5 个核心功能）

---

## 独特功能亮点

| 功能 | 说明 | 竞品对比 |
|------|------|----------|
| {功能} | {做什么、怎么用} | {竞品没有/做得更好} |

---

## 常见问题 Top 10

| # | 问题 | 解答 | 来源 |
|---|------|------|------|
| 1 | {问题} | {解答} | {官方/用户反馈} |
| 2 | {问题} | {解答} | {官方/体验发现} |
...

---

## 获取帮助
- **官方帮助中心**：{URL}
- **App 内反馈**：{路径}
```

---

#### 7b-B：详尽模式使用说明书（MECE）

**覆盖范围**：遍历产品的每一个功能、每一个设置项、每一个交互细节。
**组织原则**：MECE（Mutually Exclusive, Collectively Exhaustive）
- **相互独立**：每个章节/功能模块边界清晰，不重叠
- **完全穷尽**：不遗漏任何功能、设置项、操作路径

**MECE 分类方法**：
1. 先按产品信息架构做一级分类（通常对应 Tab / 主模块）
2. 每个一级分类下，穷尽列出所有二级功能
3. 每个二级功能下，穷尽列出所有操作和设置项
4. 最后做交叉检查：是否有功能没被归入任何分类

**穷尽性保障**：
- 主 agent 在 Step 5 广度扫描时生成完整的功能树
- 每个 subagent 深度探索时记录该模块下的所有子功能
- 汇总后做 gap analysis：对照功能树，标记哪些功能已写入说明书，哪些遗漏
- 官方帮助中心的功能分类作为交叉验证

写入 `{BASE_DIR}/manual-full.md`：

```markdown
# {App 名称} 完整使用说明书

> **版本**：{版本} | **平台**：{iOS/Android} | **更新日期**：{YYYY-MM-DD}
> **官方帮助中心**：{URL}
> **覆盖模式**：详尽模式（MECE），覆盖率 {X}%

---

## 目录
（自动生成，完整的多级目录）

---

## 一、产品概述

### 1.1 产品简介
- **产品名称**：{名称}
- **核心价值**：{一句话}
- **目标用户**：{人群}
- **支持平台**：iOS {版本}+ / Android {版本}+

### 1.2 功能全景图

```
{App 名称} 功能树（MECE）
├── A. {一级模块 1}
│   ├── A1. {二级功能}
│   │   ├── A1a. {三级功能/操作}
│   │   └── A1b. {三级功能/操作}
│   ├── A2. {二级功能}
│   └── A3. {二级功能}
├── B. {一级模块 2}
│   ├── B1. ...
│   └── B2. ...
├── C. {一级模块 3}
│   └── ...
├── D. 账户与设置
│   ├── D1. 个人资料
│   ├── D2. 隐私设置
│   ├── D3. 通知管理
│   ├── D4. 订阅与付费
│   └── D5. 数据与存储
└── E. 系统功能
    ├── E1. 搜索
    ├── E2. 通知中心
    ├── E3. 分享
    └── E4. Widget / 快捷指令
```

### 1.3 MECE 覆盖检查表

| 一级模块 | 二级功能数 | 已覆盖 | 覆盖率 |
|----------|-----------|--------|--------|
| {模块 A} | {N} | {N} | 100% |
| {模块 B} | {N} | {N} | 100% |
| ... | ... | ... | ... |
| **合计** | **{总数}** | **{已覆盖}** | **{X}%** |

---

## 二、下载、安装与注册

### 2.1 下载安装
（iOS / Android 分别说明）

### 2.2 注册方式
| 方式 | 步骤 | 截图 |
|------|------|------|
| {方式 1} | {步骤} | ![](screenshots/...) |
| {方式 2} | {步骤} | ![](screenshots/...) |

### 2.3 登录与登出
### 2.4 忘记密码
### 2.5 多设备登录
### 2.6 首次使用引导

---

## 三、主界面与导航

### 3.1 主界面布局
![主界面标注图](screenshots/home-annotated.png)

### 3.2 底部导航栏
| Tab | 功能 | 截图 |
|-----|------|------|
| {每个 Tab} | {说明} | ![](screenshots/...) |

### 3.3 顶部操作栏
### 3.4 侧边栏/抽屉菜单（如有）
### 3.5 手势操作一览

| 手势 | 位置 | 效果 |
|------|------|------|
| {所有手势穷尽列出} | | |

### 3.6 搜索功能
- 搜索入口 / 搜索建议 / 搜索筛选 / 搜索历史 / 空结果

### 3.7 通知中心
- 通知类型 / 通知操作 / 已读未读

---

## 四、{一级模块 A}：完整功能说明

### 4.1 {二级功能 A1}

#### 功能说明
{这个功能做什么，解决什么问题}

#### 使用步骤
| 步骤 | 操作 | 截图 |
|------|------|------|
| 1 | {操作说明} | ![](screenshots/...) |
| 2 | {操作说明} | ![](screenshots/...) |
| ... | ... | ... |

#### 所有操作项

| 操作 | 入口 | 说明 |
|------|------|------|
| {操作 1} | {怎么触发} | {效果} |
| {操作 2} | {怎么触发} | {效果} |
| {穷尽列出所有操作} | | |

#### 设置项（如有）

| 设置项 | 选项 | 默认值 | 说明 |
|--------|------|--------|------|
| {设置 1} | {可选值} | {默认} | {影响什么} |
| {穷尽列出所有设置} | | | |

#### 边界情况
- 空状态：{截图 + 说明}
- 上限/限制：{数量限制、大小限制等}
- 错误处理：{常见错误及提示}

#### FAQ
| 问题 | 解答 | 来源 |
|------|------|------|
| {问题} | {解答} | {官方帮助/体验发现} |

### 4.2 {二级功能 A2}
...（对每个二级功能重复以上完整结构）

---

## 五、{一级模块 B}：完整功能说明
...（对每个一级模块重复第四章结构）

---

## N-3. 账户与设置（穷尽）

### 个人资料
| 设置项 | 操作 | 说明 |
|--------|------|------|
| {穷尽每一个设置项} | | |

### 隐私设置
| 设置项 | 选项 | 默认值 | 说明 |
|--------|------|--------|------|
| {穷尽} | | | |

### 通知管理
| 通知类型 | 默认状态 | 可关闭 | 说明 |
|----------|----------|--------|------|
| {穷尽} | | | |

### 订阅与付费（如有）
| 套餐 | 价格 | 权益 | 操作 |
|------|------|------|------|

### 存储与缓存
### 数据导出
### 注销与删除账户

---

## N-2. 常见问题 FAQ（全量）

> 来源：官方帮助中心 + 应用商店用户反馈 + 产品体验发现

### 登录与账户
| 问题 | 解答 | 来源 |
|------|------|------|

### {模块 A} 相关
| 问题 | 解答 | 来源 |
|------|------|------|

### {模块 B} 相关
...（按 MECE 分类，每个模块的 FAQ 独立一节）

### 付费与订阅
| 问题 | 解答 | 来源 |
|------|------|------|

---

## N-1. 故障排除

### 通用问题
| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| App 闪退 | {原因} | {方案} |
| 加载失败 | {原因} | {方案} |
| 同步异常 | {原因} | {方案} |

### {模块 A} 故障
| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|

### {模块 B} 故障
...

---

## N. 附录

### 版本更新记录
| 版本 | 日期 | 更新内容 |
|------|------|----------|

### 快捷操作速查表
| 操作 | 手势/入口 |
|------|----------|
| {穷尽所有快捷操作} | |

### 获取帮助
- 官方帮助中心 / App 内反馈 / 客服 / 社区

### 覆盖率报告
（说明书完成后，自动生成覆盖率统计：
  对照功能树，列出每个功能是否已写入说明书，计算总覆盖率。
  如有遗漏，标注原因：如"需要登录"、"需要付费"、"地区限制"等。）
```

### 7c. 功能/体验优化建议

写入 `{BASE_DIR}/optimization.md`，结构：

```markdown
# {App 名称} 优化建议报告

## 数据来源
- 产品体验探索发现
- {N} 个模拟用户角色反馈汇总

## 优化建议总览

| 优先级 | 建议 | 预期收益 | 估计成本 | ROI 评估 |
|--------|------|----------|----------|----------|
| P0 | {建议} | {收益} | {成本} | {评估} |
| P1 | {建议} | {收益} | {成本} | {评估} |
| P2 | {建议} | {收益} | {成本} | {评估} |

## 详细建议

### P0: {建议标题}
- **现状**：{当前体验描述}
- **问题**：{具体问题}
- **建议方案**：{改进方案}
- **预期收益**：{用户体验/转化率/留存等}
- **实现成本**：{开发量评估}
- **用户反馈汇总**：
  - {角色 1}：{反馈}
  - {角色 2}：{反馈}

### P1: ...
### P2: ...

## 用户反馈原始数据
（各模拟用户角色的完整反馈）
```

---

## 操作速查

### mobile-mcp 工具速查

| 操作 | 工具 | 关键参数 |
|------|------|----------|
| 列出设备 | `mobile_list_available_devices` | 无 |
| 屏幕尺寸 | `mobile_get_screen_size` | device |
| 列出 App | `mobile_list_apps` | device |
| 启动 App | `mobile_launch_app` | device, packageName |
| 终止 App | `mobile_terminate_app` | device, packageName |
| 截图（查看）| `mobile_take_screenshot` | device |
| 截图（保存）| `mobile_save_screenshot` | device, saveTo(.png/.jpg) |
| 列出元素 | `mobile_list_elements_on_screen` | device |
| 点击 | `mobile_click_on_screen_at_coordinates` | device, x, y |
| 双击 | `mobile_double_tap_on_screen` | device, x, y |
| 长按 | `mobile_long_press_on_screen_at_coordinates` | device, x, y, duration(ms) |
| 滑动 | `mobile_swipe_on_screen` | device, direction(up/down/left/right) |
| 输入文字 | `mobile_type_keys` | device, text, submit(bool) |
| 按钮 | `mobile_press_button` | device, button(HOME/BACK/VOLUME_UP/DOWN/ENTER) |
| 打开 URL | `mobile_open_url` | device, url |
| 屏幕方向 | `mobile_get_orientation` / `mobile_set_orientation` | device |
| 录屏开始 | `mobile_start_screen_recording` | device |
| 录屏结束 | `mobile_stop_screen_recording` | device |

### 操作要点

1. **先 list 再 click**：每次操作前用 `mobile_list_elements_on_screen` 获取坐标，不要猜测坐标
2. **先截图再操作**：每个关键步骤操作前，先 `mobile_save_screenshot` 记录当前状态
3. **截图不要缓存**：`mobile_take_screenshot` 和 `mobile_list_elements_on_screen` 的结果不要缓存，屏幕状态随时变化
4. **iOS 返回**：没有 BACK 按钮，需要找屏幕上的返回按钮点击，或从左边缘右滑
5. **Android 返回**：用 `mobile_press_button button: BACK`
6. **滑动浏览**：列表页面用 `mobile_swipe_on_screen direction: up` 向下滚动查看更多内容
7. **截图命名**：`{module}-{step}-{description}.png`，保持有序

---

## 注意事项

- 报告语言跟随用户习惯（默认中文）
- 每次体验按日期和版本独立存储，方便追踪产品迭代
- subagent 探索时如果遇到需要登录的功能，报告为"需要登录"并跳过，不要尝试注册
- 如果 App 崩溃或无响应，记录问题并尝试重新启动
- 探索时间较长的产品，定期向用户汇报进度
