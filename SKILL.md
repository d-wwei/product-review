---
name: product-review
description: |
  Use when the user asks to experience, review, or evaluate a mobile app.
  Trigger phrases: "体验一下 [App]", "写一份 [App] 的产品体验报告",
  "测评 [App]", "product review", "review this app".
  Subcommands: ue-report, manual core, manual mece, suggestion, playcover-setup, android-setup, ios-sim-setup, ios-device-setup.
  Features persona-driven user simulation: generates 3-12 diverse synthetic users
  (varying demographics, expertise, motivations, behavior patterns) who independently
  experience the product and provide authentic, subjective feedback.
  Three simulation modes: A (full autonomous - all personas operate the app),
  B (hybrid - core personas operate + others review), C (review-only - fastest).
  Requires mobile-mcp (@mobilenext/mobile-mcp) for iOS/Android simulator control.
type: interactive
argument-hint: "[ue-report | manual core | manual mece | suggestion] [App名称]"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - AskUserQuestion
  - WebSearch
  - WebFetch
  - TaskCreate
  - TaskUpdate
  - TaskList
---

# Product Review

AI agent 自主体验移动端产品，撰写结构化产品体验报告。

通过 mobile-mcp 控制 iOS/Android 模拟器，系统性浏览 App 的每个功能，
截图记录，最终输出专业的产品体验报告、使用说明书或优化建议。

## 子命令

| 命令 | 说明 | 输出文件 |
|------|------|----------|
| `/product-review ue-report [App]` | 产品体验报告（UX 五层模型） | `report.md` |
| `/product-review manual core [App]` | 使用说明书 Core 模式（核心功能） | `manual-core.md` |
| `/product-review manual mece [App]` | 使用说明书详尽模式（MECE 穷尽） | `manual-full.md` |
| `/product-review suggestion [App]` | 功能/体验优化建议 | `optimization.md` |
| `/product-review playcover-setup [App]` | 配置 PlayCover 让 iOS app 以 iPhone 模式运行 | 终端输出 |
| `/product-review android-setup [App]` | 设置 Android 模拟器环境 | 终端输出 |
| `/product-review ios-sim-setup [App]` | 设置 iOS 模拟器环境 | 终端输出 |
| `/product-review ios-device-setup` | 连接 iOS 真机进行测试 | 终端输出 |
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
1. 从参数中提取 --mode=X 和 --personas=N 标志（如有），记录后从参数中移除
2. 提取第一个词作为子命令候选
3. 如果是 "ue-report" → 模式 = UE_REPORT，剩余参数 = App 名称
4. 如果是 "manual"  → 读取第二个词：
   - "core" → 模式 = MANUAL_CORE，剩余参数 = App 名称
   - "mece" → 模式 = MANUAL_MECE，剩余参数 = App 名称
   - 其他   → 模式 = MANUAL_CORE（默认），第二个词起为 App 名称
5. 如果是 "suggestion" → 模式 = SUGGESTION，剩余参数 = App 名称
   5.1 如果是 "playcover-setup" → 模式 = PLAYCOVER_SETUP，剩余参数 = App 名称/bundleID
   5.2 如果是 "android-setup" → 模式 = ANDROID_SETUP，剩余参数 = App 名称/APK 路径
   5.3 如果是 "ios-sim-setup" → 模式 = IOS_SIM_SETUP，剩余参数 = App 名称
   5.4 如果是 "ios-device-setup" → 模式 = IOS_DEVICE_SETUP，剩余参数 = 无或设备名
6. 其他情况 → 模式 = INTERACTIVE，整个参数 = App 名称
7. 如果提取到 --mode 标志，记录 persona_mode = A/B/C（默认 B）
8. 如果提取到 --personas 标志，记录 persona_count = N（默认由 Step 5.5 决定）
```

**模式与探索深度的映射：**

| 模式 | 探索深度 | 报告类型 | Persona 模拟 |
|------|----------|----------|-------------|
| UE_REPORT | 完整体验（广度+深度） | 产品体验报告 | 可选（默认开启，模式 B） |
| MANUAL_CORE | 核心功能体验 | 使用说明书 Core | 无 |
| MANUAL_MECE | 完整体验（穷尽所有功能） | 使用说明书 MECE | 无 |
| SUGGESTION | 完整体验 + 拟人用户模拟 | 优化建议 | 必须（默认模式 B） |
| INTERACTIVE | 用户选择 | 用户选择 | 用户选择 |

**Persona 模拟模式参数**（可选，附加在子命令后）：

```
/product-review suggestion --mode=A 抖音     → 全自主体验
/product-review suggestion --mode=B 小红书   → 混合体验（默认）
/product-review suggestion --mode=C WeChat   → 全评审体验
/product-review suggestion --personas=8 抖音  → 指定 8 个 persona
/product-review suggestion --mode=A --personas=5 抖音  → 全自主 + 5 个 persona
```

当模式 = INTERACTIVE 时，进入 Step 3 的交互式选择。
其他模式跳过 Step 3，直接按对应配置执行。
Persona 模式和数量参数未指定时使用默认值（模式 B，数量由产品复杂度决定）。

---

## Step 0: 环境检查

运行以下检查，确认 mobile-mcp 可用：

```bash
# 检查 mobile-mcp 是否已配置
if grep -q "mobile" ~/.claude/settings.json 2>/dev/null || \
   grep -q "mobile" ~/.claude/settings.local.json 2>/dev/null; then
  echo "MOBILE_MCP=configured"
else
  echo "MOBILE_MCP=not_found"
fi
```

**如果 `MOBILE_MCP=not_found`：**

告诉用户：

> mobile-mcp 未检测到。请先安装：
>
> ```bash
> claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest
> ```
>
> 前置要求：Node.js v22+，iOS 需要 Xcode CLI tools，Android 需要 Platform Tools。
> 安装后重启 Claude Code 再试。

**停止执行，等用户完成安装。**

如果已配置，继续。

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

### Persona 模拟配置（当报告类型包含 suggestion 或 ue-report 时）

如果用户选择了包含 persona 模拟的报告类型，额外询问以下配置：

**体验模式（Persona Simulation Mode）**

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **A: 全自主体验** | 所有 persona 独立操控 App，各自走不同路径 | 最高质量评测，时间充裕 |
| **B: 混合体验（默认）** | 3 个核心 persona 实际操控 App + 其余 persona 基于探索数据评审 | 平衡质量与效率 |
| **C: 全评审体验** | 所有 persona 基于 Step 5 探索数据和截图进行评审 | 快速迭代，时间紧迫 |

**Persona 数量**

根据产品复杂度和目标用户群广度，建议 persona 数量（初始估计，Step 5.5 会精化）：

| 产品复杂度 | 功能模块数 | 建议 persona 数 | 说明 |
|-----------|-----------|----------------|------|
| 简单工具类 | 1-2 个核心功能 | 3-4 | 目标用户群相对集中 |
| 中等复杂度 | 3-5 个功能模块 | 5-6 | 有多类用户群，核心路径清晰 |
| 复杂应用 | 6+ 个功能模块 | 7-8 | 用户群较广，场景多样 |
| 平台级产品 | 超级 App / 平台 | 8-12 | 用户群极广，场景极多 |

Step 3 的初始选择是估计值。Step 5 探索完成后，Step 5.5 会基于实际产品结构精化 persona 数量建议，由用户最终确认。用户也可通过 `--personas=N` 直接指定。

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
  _experience_bank/                     # 经验银行（跨产品共享，自动积累）
    _index.md                           #   经验索引
    categories/                         #   按产品类别的经验
      finance-investing.md
      social-content.md
      general.md
      ...
    personas/                           #   Persona 相关经验
      effective-combinations.md         #     高效属性组合
      anti-patterns.md                  #     无效模式
    strategies/                         #   策略经验
      exploration-patterns.md           #     探索策略
      report-quality.md                 #     报告质量
  <app-name>/
    <YYYYMMDD>/
      screenshots/                      # 所有截图
        01-launch.png
        02-home-tab1.png
        persona-1/                      # Persona 自主体验截图（模式 A/B）
        persona-2/
        ...
      exploration-reports/              # Subagent 探索报告（每个 subagent 一个文件）
        subagent-E1-watchlists-all.md
        subagent-E2-msft-detail.md
        ...
      exploration-queue.md              # 探索队列进度追踪（审计文档）
      explored-pages-registry.md        # 已探索页面去重清单（审计文档）
      mece-coverage-audit.md            # MECE 覆盖检查表（审计文档，不放入 manual）
      exploration-log.md                # 探索日志（主 agent 广度扫描汇总）
      persona-profiles.md               # Persona 档案（Step 5.5 + 6.0 生成）
      persona-feedback-synthesis.md     # Persona 反馈综合分析（Step 6.3）
      report.md                         # 产品体验报告（如选择 ue-report）
      manual-core.md                    # 使用说明书 Core 模式（如选择）
      manual-full.md                    # 使用说明书 MECE 模式（如选择）
      optimization.md                   # 优化建议（如选择 suggestion）
      official-help-data.md             # 官方帮助数据（manual 模式生成）
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
2. **执行滚动扫描协议**（至少 3 次 swipe up + list_elements 对比，产出滚动摘要）
3. 记录该入口下的子功能列表（包括滚动后才可见的入口）
4. 返回主页

**主 agent 也必须遵守滚动扫描协议。** 广度扫描时很容易只看每个 Tab 的第一屏就切走——这会导致遗漏大量子入口。

```
mobile_click_on_screen_at_coordinates  device: <device_id>  x: <x>  y: <y>
mobile_take_screenshot                 device: <device_id>
mobile_list_elements_on_screen         device: <device_id>
mobile_save_screenshot                 device: <device_id>  saveTo: <path>
```

广度扫描完成后，输出产品结构大纲到 `exploration-log.md`。

### 5.1.5 构建探索队列（Exploration Queue）

广度扫描完成后，主 agent 在深度探索之前，必须生成一个结构化的**探索队列**。
此队列列出所有需要深度探索的页面/入口，是后续分派 subagent 和验证覆盖完整性的核心数据结构。

**写入文件**：`{BASE_DIR}/exploration-queue.md`

```markdown
# 探索进度追踪

## 统计
- 总入口数: {N}
- 已完成: 0 (0%)
- 待探索: {N}
- 受限/跳过: 0

## 探索队列（Exploration Queue）

| ID | 页面/入口 | 入口路径 | 预估复杂度 | 状态 | 分配给 | 滚动次数 | 到底 |
|----|----------|---------|-----------|------|--------|---------|------|
| E1 | {页面名} | {从首页开始的点击路径} | 高/中/低 | pending | - | - | - |
| E2 | {页面名} | {路径} | 高/中/低 | pending | - | - | - |
| ... | | | | | | | |
```

**复杂度分级规则**：
- **高**：股票详情页（多个子 Tab）、Settings（大量子页面）、Assets/Portfolio 等已知多屏内容
- **中**：一般功能页面（Markets 子 tab、Discover 子模块等）
- **低**：简单列表页、空状态页、单屏信息页

**队列构建原则**：
1. **按导航效率排序**：同一个 Tab 下的页面连续排列（减少 Tab 切换开销）
2. **同一详情页的多个子 Tab 相邻排列**（如 MSFT 的 Chart/Options/Fundamentals 连续）
3. **每个可点击入口都是一个队列项**——不合并、不跳过
4. **Tab 类页面拆分**：如果一个页面包含多个 Tab/标签，每个 Tab 作为独立队列项

**同时初始化已探索页面去重清单**：`{BASE_DIR}/explored-pages-registry.md`

```markdown
# 已探索页面清单（Explored Pages Registry）

## 页面唯一标识规则
- 股票详情页：stock-detail:{SYMBOL}
- 功能页面：{module}:{subpage}（如 markets:overview、accounts:assets）
- 设置子页面：settings:{item_name}
- 新闻/帖子详情：article:{title_hash} 或 post:{post_id}

## 清单

| 页面唯一标识 | 探索者 | 入口路径 | 状态 |
|-------------|--------|---------|------|
```

### 5.2 深度探索（One-Page-One-Agent 架构）

**核心规则：每个探索队列中的条目由一个独立 subagent 完成。**

不论复杂度高低，每个页面/入口分派一个独立 subagent。理由：
- 彻底消除"后半段页面被跳过"——每个 subagent 只需关注一个页面，context window 充裕
- 即使简单页面，独立 subagent 也能确保滚动扫描协议被严格执行
- 单设备串行约束下，subagent 逐个执行，但每个执行时间短（3-8 分钟）

#### 5.2.1 Subagent 分派流程

```python
# 主 agent 按探索队列顺序，逐个分派 subagent
# ⚠️ 单设备同一时间只能有一个 subagent 操作，必须串行

current_device_position = "首页"  # 追踪设备当前位置

for entry in exploration_queue:
    if entry.status != "pending":
        continue

    # 更新队列状态
    entry.status = "in_progress"
    entry.assigned_to = f"subagent-{entry.id}"

    Agent(
        prompt=f"""
你是页面探索员。目标：深度探索一个页面。

## 目标页面：{entry.page_name}
## 入口路径：{entry.entry_path}
## 设备 ID：{device_id}
## 截图目录：{screenshots_dir}
## 报告输出文件：{BASE_DIR}/exploration-reports/subagent-{entry.id}-{entry.page_slug}.md
## 设备当前位置：{current_device_position}

## 已探索页面（不需要重复进入）
{explored_pages_list}

## 任务（严格按顺序执行）
1. 从当前设备位置导航到目标页面（入口路径仅供参考，从当前位置选最短路径）
2. 执行滚动扫描协议（见下方，至少3次滚动 + 元素对比 + 摘要）
3. 点击页面上的每个入口/按钮/Tab，进入子页面后也执行滚动扫描
4. 实际操作可交互功能（搜索框输入、筛选切换、表单填写等）
5. 将报告写入指定的报告输出文件
6. 将设备导航回底部 Tab 首页之一（优先选与下一个目标最近的 Tab）

## ⚠️ 滚动扫描协议（每个页面必须执行，不可跳过）

```
STEP A: 初始快照
  → mobile_list_elements_on_screen → 记为 snapshot_0，元素数 = N0
  → mobile_save_screenshot（{{page}}-scroll-0-top.png）

STEP B: 滚动循环（增强版，含无限滚动检测）
  → scroll_count = 0, same_count = 0
  → LOOP:
      → mobile_swipe_on_screen direction: up
      → mobile_list_elements_on_screen → 记为 snapshot_{{scroll_count+1}}
      → scroll_count += 1
      → 有新元素 → same_count=0；完全相同 → same_count+=1

      // 常规退出
      → scroll_count >= 3 且 same_count >= 2 → 退出

      // ⚠️ 无限滚动检测
      → 如果连续 3 次滚动都出现新的同类型列表项（如帖子、新闻、股票行）：
        1. 标注 [无限滚动列表]
        2. 记录列表项结构特征（每条包含哪些字段）
        3. 记录已看到的条目数量
        4. **立即停止滚动**
        5. 备注："此页面为无限加载列表，已采样 {{N}} 条内容，
           列表结构为 {{描述}}。继续滚动不会产生新的功能发现。"

      // 安全上限
      → scroll_count >= 15 → 强制退出
      // 接力阈值（非无限滚动时）
      → scroll_count >= 10 且仍有新内容 → 停止并报告 [接力点]

STEP C: 底部快照
  → mobile_save_screenshot（{{page}}-scroll-{{scroll_count}}-bottom.png）

STEP D: 滚动摘要
  → "页面: {{name}} | 滚动: {{scroll_count}}次 | 首屏: {{N0}}个 |
     末屏: {{Nf}}个 | 新增: {{total_new}} | 到底: {{是/否}} | 底部: {{标志}}"
```

无限滚动典型特征（供参考判断）：
| 特征 | 说明 |
|------|------|
| 列表项结构重复 | 每条都是 头像+用户名+内容+时间+互动按钮 |
| 持续加载新内容 | 每次滚动后 list_elements 都有新的同结构元素 |
| 无固定底部 | 没有版权信息、"已加载全部"等底部标志 |
| loading 指示器 | 底部有旋转加载动画或"加载中"文字 |

## 页面内多 Tab 处理规则
如果目标页面包含多个 Tab/标签（如股票详情页的 Chart/Options/Fundamentals/Comments/News/Analysis），
**每个 Tab 都是独立的第 1 层页面，都必须逐一点击并执行滚动扫描协议。**
不可只探索第一个 Tab 就返回。

## 去重规则
当你点击一个入口进入新页面时：
1. 判断该页面的唯一标识（如 stock-detail:MSFT、markets:overview）
2. 检查是否在"已探索页面"列表中：
   - 已探索 → 截图记录"此页面已探索"，立即返回
   - 未探索 → 执行完整探索
3. 对于股票列表中的个股：
   - 只需点击 1-2 只代表性个股验证详情页结构
   - 不同市场（US vs CA）至少各探索一只

## 交互操作清单（按页面类型，适用的必须执行）

### 列表页面：
□ 排序（点击列头切换升序/降序）
□ 下拉刷新
□ 长按（查看是否有上下文菜单）
□ 左/右滑（查看是否有快捷操作）

### 表单页面：
□ 实际输入内容到输入框
□ 切换不同选项（如订单类型）
□ 走到最后一步确认页面（不提交）
□ 测试输入验证（如输入非法值）

### 图表页面：
□ 长按查看十字光标
□ 切换至少3个时间周期
□ 添加/切换至少2个技术指标
□ 全屏模式

### 搜索功能：
□ 输入至少2个不同关键词
□ 查看搜索建议
□ 点击搜索结果进入详情
□ 清空搜索

### 下拉菜单/折叠面板：
□ 展开所有下拉选择器并截图记录完整选项列表
□ 展开所有 ">" 箭头、"查看更多"、折叠面板

## 数据准确性规则（"所见即所录"）
1. 【禁止推断】报告中所有数据必须来自 App 中实际看到的内容，不可推断
2. 【下拉菜单必须展开】遇到下拉选择器，必须点击展开并截图记录所有可选项
3. 【折叠面板必须展开】遇到 ">"、"查看更多"、"展开" 按钮，必须点击展开
4. 【数量必须实数】写"支持 N 种订单类型"时，N 必须逐一看到，每种列出名称
5. 【截图作为证据】关键数据点必须有对应截图作为依据，引用时标注截图文件名

## 性能感知记录（每个页面额外记录）
| 维度 | 记录内容 | 评价标准 |
|------|---------|---------|
| 页面加载 | 点击到内容完全显示的体感时间 | 快(<1s)/正常(1-3s)/慢(>3s) |
| 滚动流畅度 | 滚动时是否有掉帧/卡顿 | 流畅/偶尔卡顿/明显卡顿 |
| 动画反馈 | 点击按钮后是否有视觉反馈 | 有/无/延迟 |
| 图片加载 | 图片/图表是否及时渲染 | 即时/延迟可见/占位图长时间 |
| 错误状态 | 是否遇到报错/闪退/白屏 | 有(详述)/无 |
| 空状态 | 无数据时的展示是否友好 | 有引导/仅文字/空白 |

## 受限功能处理（遇到认证/付费阻断时）
1. 截图阻断界面
2. 记录阻断前可见的所有元素
3. 尝试其他入口（同一功能可能有多个入口，有的可能不需要认证）
4. 在"未探索入口清单"中标注具体阻断原因
5. 不要直接放弃——先尝试 BACK 再进，或从其他页面进入

## 敏感输入处理（交易密码/登录等）
遇到需要输入密码、个人信息、银行账户等敏感信息的页面时：
1. 截图当前页面，记录所有可见元素
2. 通过 AskUserQuestion 询问用户是否手动输入以继续
3. 绝不自行输入或猜测密码
4. 用户选择跳过时，记录到"未探索入口清单"

## 输出要求

### 报告必须写入文件
将完整报告写入：{{BASE_DIR}}/exploration-reports/subagent-{{entry.id}}-{{page_slug}}.md

### 返回给主 agent 的只是摘要（不超过 20 行）
"页面: {{name}} | 状态: 完成/接力/受限 | 滚动: {{N}}次 | 到底: 是/否 |
 新发现入口: {{数量}} | 截图: {{数量}}张 | 报告: {{文件路径}}
 新探索的页面标识: {{列表}}
 性能问题: {{如有}}"

### 报告末尾必须包含：

#### 【滚动覆盖汇总表】
| 页面 | 滚动次数 | 首屏元素 | 末屏元素 | 新增元素 | 到底 | 性能感知 | 备注 |
|------|---------|---------|---------|---------|------|---------|------|
| {{页面名}} | {{N}} | {{N0}} | {{Nf}} | {{new}} | 是/否 | {{评价}} | |

#### 【未探索入口清单】（必须，即使为空也要写"无"）
| 入口名称 | 发现位置 | 未探索原因 | 建议处理 |
|---------|---------|-----------|---------|
| {{名称}} | {{位置}} | {{原因}} | 需新subagent/标记受限/... |
| 无 | - | - | - |

没有滚动覆盖汇总表或未探索入口清单 = 探索不合格。
""",
        description=f"探索 {{entry.page_name}}"
    )

    # subagent 完成后更新队列和去重清单
    current_device_position = subagent_result.device_position
    entry.status = "completed" / "relay_needed" / "restricted"
    update_explored_pages_registry(subagent_result.explored_pages)
    add_new_entries_to_queue(subagent_result.unexplored_entries)
```

#### 5.2.2 长页面接力机制（Relay Protocol）

当 subagent 报告 `[接力点]`（scroll_count 达到 10 且页面仍有新内容时），主 agent 派生接力 subagent：

```
接力规则：

1. 当 subagent 的滚动次数达到 10 且页面仍有新内容（非无限滚动）：
   - 当前 subagent 保存当前位置截图
   - 报告标注：[接力点] scroll_count=10, 最后可见元素={列表}
   - 结束当前 subagent（不返回首页）

2. 主 agent 派生接力 subagent，prompt 包含：
   "你是接力探索员。前一位在 {page_name} 滚动了 10 次到达 {位置描述}。
    请从当前屏幕位置继续向下滚动，直到到底。
    然后探索页面上尚未被前一位点击的入口。"

3. 接力最多 3 轮（即最深 30 次滚动 + 安全上限 15 = 45 次）

4. 接力 subagent 的报告追加到同一个报告文件中
```

#### 5.2.3 Subagent 导航效率规则

```
导航标准化规则：

1. 每个 subagent 结束时，必须将设备导航回"底部 Tab 首页"之一。
   优先回到与下一个 subagent 目标最近的 Tab。

2. 主 agent 分派时告知当前设备位置。

3. Subagent 从当前位置出发导航，不假设在首页。

4. 主 agent 按导航效率排序探索队列：
   - 同一 Tab 下的页面连续探索
   - 同一详情页的多个子 Tab 由相邻 subagent 探索
```

### 5.3 全局功能专项探索 + 特殊维度检查

广度和深度探索完成后，单独派发 subagent 专门测试**全局功能**和**特殊维度**。

#### 5.3.1 全局功能专项（独立 subagent）

以下功能在多个页面都可见，但容易被各页面的 subagent 跳过。单独派发一个 subagent 专门测试：

```
全局功能专项 subagent prompt：

你是全局功能探索员。你的任务是测试 App 中跨页面的全局功能。

## 必须测试的全局功能

### 1. 搜索功能
- 找到搜索入口（通常在顶部栏）
- 点击进入搜索页面
- 输入至少 3 个不同关键词（如一只股票、一个新闻话题、一个用户名）
- 记录：搜索建议、搜索结果分类（Quotes/News/Community 等）、结果排序
- 点击搜索结果进入详情
- 测试空搜索结果
- 清空搜索并返回

### 2. 消息/通知中心
- 找到消息/通知入口（通常在顶部栏，可能有红点/badge）
- 点击进入消息列表
- 查看不同消息分类（如有）
- 打开一条消息查看详情
- 执行滚动扫描协议

### 3. AI 上下文入口（如有）
- 在不同页面（如股票详情页）找到 AI 入口
- 点击进入 AI 对话
- 验证是否携带当前页面上下文（如当前股票信息）
- 实际提问并记录回答质量

### 4. 分享功能
- 在详情页找到分享按钮
- 点击查看分享选项和内容预览

### 5. 横屏模式（如有）
- 在图表页面点击横屏按钮
- 截图横屏界面
- 返回竖屏

输出报告写入文件，格式同普通 subagent。
```

#### 5.3.2 首次启动体验检查

**首次启动体验：**
- 有没有引导页/教程
- 权限请求是否合理（时机、说明）
- 登录/注册流程

#### 5.3.3 边界情况检查

**边界情况：**
- 断网提示（如果可以模拟）
- 空列表/空状态页面
- 长文本/特殊字符输入
- 快速连续点击

### 5.4 验证与补扫（Verify & Redispatch Loop）

**所有 subagent 完成后，主 agent 执行验证-补扫循环。这是确保探索完整性的最后防线。**

```
验证与补扫流程：

1. 收集所有 subagent 的摘要报告

2. 合并所有滚动覆盖汇总表，逐行检查：
   - 滚动次数 < 3 → 标记为 [不合格]
   - 滚动次数 = 0 → 标记为 [未探索]
   - 到底 = 否 且非无限列表 且非接力完成 → 标记为 [未到底]

3. 收集所有"未探索入口清单"，将新入口加入探索队列（标记为 pending）

4. 对所有 [不合格] 和 [未探索] 的页面，重新分派 subagent 补扫

5. 补扫 subagent prompt 模板：
   """
   你是补扫员。以下页面在之前的探索中未完成滚动扫描。
   请对每个页面严格执行滚动扫描协议。

   ## 待补扫页面
   {从探索队列中提取的不合格/未探索项}

   ## 注意
   - 每个页面必须至少滚动 3 次
   - 发现新入口要记录到"未探索入口清单"
   - 最后输出滚动覆盖汇总表和未探索入口清单
   - 将报告写入文件
   """

6. 循环直到探索队列中所有 pending 项都变为 completed 或 skipped(with reason)

7. 最终更新 exploration-queue.md：
   - 统计：总入口数、已完成数、完成率、受限/跳过数
   - 每行状态更新为 ✅完成 / ⚠️接力完成 / ❌受限 / ⏭️跳过(原因)
```

**补扫循环的退出条件**：
- 所有 pending 项都已处理（完成或标记为 skipped）
- 补扫轮次达到 3 轮（防止无限循环）
- 每轮补扫后新发现的未探索入口数为 0

### 5.4.5 探索断点恢复

```
如果会话恢复后重新运行 product-review：

1. 检查 {BASE_DIR}/exploration-queue.md 是否存在
2. 如果存在，用 AskUserQuestion 询问：
   "检测到之前的探索进度（{N}/{M} 完成）。是否从断点继续？"
3. 继续时，只对 pending/failed 的队列项派发 subagent
4. 每个 subagent 完成后立即更新队列文件和报告文件
```

---

## Step 5.5: 产品用户画像分析

在 Step 5 探索完成后、Step 6 拟人模拟前执行。主 agent 基于探索数据分析产品的目标用户群体，为 persona 生成提供结构化输入。

**输入**：Step 5 的 `exploration-log.md` + 截图 + 产品结构大纲 + **Experience Bank 历史经验（如有）**
**输出**：`{BASE_DIR}/persona-profiles.md`

### 5.5.0 经验检索（Experience Retrieval）

**首次 review 时跳过此步（experience bank 尚不存在）。**

如果 `reviews/_experience_bank/` 已存在，执行 Step 8 的经验检索逻辑：

```
1. 检查 reviews/_experience_bank/_index.md 是否存在
2. 如果存在：
   a. 从 Step 5 探索结果推断产品类别
   b. 加载 categories/{category}.md + categories/general.md
   c. 加载 personas/effective-combinations.md + personas/anti-patterns.md
   d. 过滤置信度 >= 0.5 的条目
   e. 将经验注入后续 5.5.1 - 5.5.4 的分析 context 中
3. 如果不存在：跳过，正常执行 5.5.1
```

注入后的效果示例：
- 品类经验说"金融 App 必须有风控理解度维度" → 5.5.1 会将其加入关键维度
- Persona 经验说"50+ 岁低技术用户 + 最低领域经验是高价值组合" → 5.5.3 采样时优先覆盖
- Anti-pattern 说"同时设定高技术+低耐心的 persona 反馈价值低" → 5.5.3 采样时避免

### 5.5.1 产品类别与 Persona 维度识别

主 agent 分析产品属于哪个类别，确定该类别下最关键的 persona 差异化维度：

| 产品类别 | 关键差异化维度 | 示例维度值 | 领域专家原型（见附录 A） |
|----------|-------------|-----------|------------------------|
| 金融/投资 | 投资经验、风险偏好、资产规模、交易频率 | 小白理财/定投族/日内交易者/长线价值投资者 | 期权交易者、基本面投资者、日内交易者、宏观/ETF投资者、定投/被动投资者、新手小白 |
| 社交/内容 | 创作能力、社交活跃度、内容偏好、隐私敏感度 | 潜水党/评论家/内容创作者/社区运营者 | 短视频创作者、图文博主、社区运营者、隐私极客、算法研究者、纯消费用户 |
| 工具/效率 | 技术水平、使用频率、工作流复杂度、付费意愿 | 轻度用户/重度依赖者/团队管理者/个人用户 | 自动化达人、键盘流极客、团队管理者、跨平台用户、API/插件开发者、轻度新用户 |
| 电商/消费 | 消费习惯、价格敏感度、品牌忠诚度、退货频率 | 冲动购物/比价达人/品牌忠粉/羊毛党 | 供应链/选品买手、优惠券/返利专家、跨境购物者、品类垂直重度用户、售后维权达人、首次网购用户 |
| 教育/学习 | 学习目标、自律程度、时间预算、基础水平 | 应试备考/兴趣学习/职业转型/终身学习者 | 备考刷题者、教师/教练、课程设计者、学龄儿童家长、自学编程者、语言学习者 |
| 健康/运动 | 运动习惯、健康目标、数据敏感度、社交需求 | 减脂新手/健身老手/慢病管理/运动社交 | 马拉松跑者、力量训练者、慢病管理患者、营养师/教练、可穿戴设备重度用户、产后恢复用户 |
| 出行/旅行 | 出行频率、预算敏感度、体验偏好、规划习惯 | 穷游背包客/商务差旅/家庭度假/说走就走 | 商务高频差旅、自驾深度游、亲子出行规划者、签证/出入境专家、酒店/航空常客会员玩家、无障碍出行需求者 |

**领域专家原型使用规则**：
1. 生成 persona 时，**至少 50% 的 persona 必须从"领域专家原型"中选取**（而非全部使用通用维度组合）
2. 每个领域专家原型自带"领域知识注入段"和"领域专项测试清单"（见附录 A），生成 persona 时必须注入
3. 如果产品涉及专业功能（如期权、财报分析、算法推荐等），对应领域专家原型的 persona 是**必选**的，不能被采样规则排除

如果产品跨多个类别（如"社交+电商"），合并相关维度。

### 5.5.2 用户分群矩阵生成

基于识别的关键维度，生成用户分群矩阵。确保覆盖以下通用维度 + 产品专属维度：

**通用维度（所有产品都需要覆盖）**：

| 维度 | 分档 | 说明 |
|------|------|------|
| 年龄段 | 18-25 / 26-35 / 36-50 / 50+ | 不同代际的数字素养和偏好差异 |
| 技术水平 | 低/中/高 | 从仅会微信到技术极客 |
| 领域经验 | 小白/入门/进阶/专家 | 在产品所属领域的认知深度 |
| 使用动机 | 刚需/好奇/被推荐/竞品迁移 | 进入产品的原因 |
| 使用场景 | 通勤碎片/办公专注/睡前放松/户外移动 | 影响注意力和耐心 |

**产品专属维度**（基于 5.5.1 识别的关键维度，每个产品不同）

### 5.5.3 属性正交采样

生成 persona 时，确保多样性的具体规则：

```
正交采样规则：
1. 最小差异约束：任意两个 persona 至少在 3 个核心维度上取值不同
2. 极端值覆盖：至少 1 个 persona 在技术水平维度取最低值（数字素养最差的用户）
3. 极端值覆盖：至少 1 个 persona 在领域经验维度取最高值（行业专家）
4. 动机多样性：不同 persona 的核心动机不能超过 50% 相同
5. 年龄分布：至少覆盖 3 个不同年龄段
6. 性格对冲：至少有 1 对 persona 在耐心/挑剔维度上完全相反

验证方法：生成完所有 persona 后，输出属性矩阵表格，
主 agent 检查是否满足以上所有约束。如不满足，调整后重新生成。
```

### 5.5.4 Persona 数量精化与确认

基于 Step 5 的实际探索结果（功能模块数、用户入口数、内容多样性），
精化 Step 3 的初始 persona 数量估计：

```
精化逻辑（与 Step 3 建议表保持一致）：
- 1-2 个核心功能 → 3-4 个 persona
- 3-5 个功能模块 → 5-6 个 persona
- 6+ 个功能模块、多种用户群 → 7-8 个 persona
- 平台级产品（超级 App / 社交平台）→ 8-12 个 persona

额外调整因素：
- 产品有明显的 B 端 + C 端双面 → +2 persona（覆盖两侧用户）
- 产品有国际化/多语言 → +1 persona（覆盖非母语用户）
- 产品涉及专业领域（医疗/金融/法律）→ +1 persona（覆盖领域专家视角）

如果用户在子命令中通过 --personas=N 指定了数量，以用户指定为准。
否则用 AskUserQuestion 展示精化建议，由用户确认。
最终数量记录到 persona-profiles.md 开头。
```

---

## Step 6: 拟人用户体验模拟

> **设计哲学**：Persona 不是"评论员"，而是"使用者"。每个 persona 是一个具有独立人格、
> 认知水平、动机和行为习惯的模拟用户。他们的反馈应该带有个人偏见、情绪波动和认知盲区——
> 就像真实用户一样。
>
> **参考架构**：MiroFish（行为参数化 + 闭环记忆）、DeepPersona（百级属性分层生成）、
> AgentReviewHub（浏览器自动化 + 多 persona UX 评测）、A/B Agent（疲劳系统 + 记忆）

### 6.0 Persona 档案生成

主 agent 基于 Step 5.5 的用户分群矩阵，为每个 persona 生成完整档案。

**档案结构**：每个 persona 包含以下结构化属性 + 200 字背景叙事。

```markdown
## Persona #{N}: {姓名}（{一句话标签}）

### 基础属性
| 属性 | 值 |
|------|-----|
| 姓名 | {真实感的中文/英文名} |
| 年龄 | {age} |
| 性别 | {gender} |
| 职业 | {profession}（{具体描述，如"三甲医院骨科住院医"而非"医生"}) |
| 教育背景 | {education} |
| 所在地 | {city, country} |
| 收入水平 | {月收入范围 或 年薪范围} |
| 家庭状态 | {单身/已婚/有孩子等} |

### 技术与领域认知
| 维度 | 等级 | 具体表现 |
|------|------|---------|
| 智能手机熟练度 | {1-5} | {具体描述，如"只会用微信和支付宝，App Store 都不会自己搜"} |
| 产品领域经验 | {1-5} | {具体描述，如"炒股 3 年，主要做 A 股，用过同花顺和东方财富"} |
| 同类产品经历 | — | {列出用过的 2-3 个同类 App 及使用深度} |
| 新功能接受度 | {保守/跟随/尝鲜} | {具体描述} |

### 动机与目标
- **核心动机**：{具体场景，如"最近想开始定投美股 ETF，朋友推荐了这个 App"，不要写泛泛的"想投资"}
- **期望解决的问题**：
  1. {具体问题 1}
  2. {具体问题 2}
  3. {具体问题 3}（可选）
- **学习成本预算**：{低/中/高} — {具体描述，如"最多愿意花 10 分钟搞清楚怎么用"}
- **付费意愿**：{免费党/按需付费/订阅型} — {描述，如"免费功能够用就不会付费"}

### 行为模式
| 行为维度 | 取值 | 说明 |
|---------|------|------|
| 注意力模式 | {快速扫描/仔细阅读/目标导向} | {如"只看大标题和按钮，文字说明基本不读"} |
| 耐心程度 | {低/中/高} | {如"加载超过 3 秒就会退出重试"} |
| 错误容忍度 | {低/中/高} | {如"遇到一次 bug 就会去 App Store 打差评"} |
| 探索风格 | {引导依赖/自主探索/搜索优先} | {如"会主动点击每个菜单看看有什么"} |
| 使用场景 | {具体场景} | {如"早上通勤地铁上，信号不稳定"} |
| 单次使用时长 | {时长范围} | {如"每次打开不超过 5 分钟"} |

### 性格特征
| 特征 | 取值 | 说明 |
|------|------|------|
| 表达风格 | {直接犀利/温和委婉/技术分析/情绪感受} | {如"喜欢吐槽，评价两极分化"} |
| 关注重点 | {功能/视觉/效率/性价比} | {优先关注什么} |
| 评价倾向 | {严格挑剔/公正中立/包容宽松} | {打分是偏高还是偏低} |
| 社交倾向 | {独立使用/喜欢讨论/会写评测} | {会不会分享使用感受} |

### 个人背景故事（200 字）

{一段叙事性的背景描述，包含这个人的生活状态、为什么会接触这个产品、
之前用同类产品的经历（好的和坏的）、对这类产品的期望和偏见。
这段叙事让 persona 从属性表格变成一个"活生生的人"。}

### 领域专业知识（如适用，基于附录 A 的领域专家原型）

> **何时需要此字段**：当该 persona 对应某个"领域专家原型"时必须填写。
> 普通用户（如新手小白）不需要此字段。

- **领域专家原型**: {对应附录 A 中的原型名称，如"期权交易者"/"基本面投资者"，无则填"通用用户"}
- **领域知识注入**（200-300 字，注入到 persona 的 system prompt 中）:
  {该 persona 掌握的专业知识概述。不是教科书，而是这个人"脑子里有什么"。
  例：期权交易者应知道 Greeks（Delta/Gamma/Theta/Vega）、IV Rank/Percentile、
  常见策略（Iron Condor/Vertical Spread/Straddle）、期权定价模型的直觉理解。
  这些知识让 persona 能给出专业级反馈，而不是只说"期权功能还不错"。}

- **领域专项测试清单**（10-15 项具体操作，从附录 A 选取或定制）:
  1. {具体操作描述，如"打开 AAPL 期权链 → 选 30DTE expiry → 检查是否显示 Delta/Gamma/Theta/Vega"}
  2. {具体操作描述}
  3. ...
  > 测试清单中的每一项必须是**可执行的具体操作**，不是"检查期权功能是否完善"这种空泛要求

- **专业反馈标准**（该 persona 给出的反馈必须达到的深度）:
  - 浅层反馈示例（❌ 不合格）: {如"期权功能比较完善"}
  - 深层反馈示例（✅ 合格）: {如"期权链支持按 Expiry Date 筛选但缺少 Delta-based strike 筛选；Greeks 只显示 Delta 和 Theta，缺少 Gamma/Vega/Rho，这对做 volatility play 的交易者是硬伤"}

### 评价倾向参数（内部使用，不展示给用户）
- sentiment_bias: {-1.0 到 1.0}  — 整体评价乐观/悲观倾向
- detail_orientation: {0 到 1.0}  — 关注细节 vs 大局的程度
- comparison_tendency: {0 到 1.0} — 倾向于与竞品比较的程度
- patience_score: {0 到 1.0}     — 耐心分数，影响探索深度
- tech_confidence: {0 到 1.0}    — 技术自信度，影响遇到问题时的反应
```

**多样性验证**（生成所有 persona 后执行）：

```
生成完所有 persona 后，输出以下验证矩阵：

| Persona | 年龄段 | 技术水平 | 领域经验 | 动机类型 | 耐心 | 评价倾向 |
|---------|--------|---------|---------|---------|------|---------|
| #{1} 张伟 | 26-35 | 中 | 小白 | 刚需 | 低 | 严格 |
| #{2} 李芳 | 50+ | 低 | 入门 | 被推荐 | 高 | 包容 |
| ...

验证规则：
✅ 任意两行至少 3 列不同
✅ 技术水平列至少包含"低"和"高"各 1 个
✅ 年龄段列至少覆盖 3 个不同区间
✅ 动机类型列不超过 50% 相同
✅ 评价倾向列至少包含"严格"和"包容"各 1 个

如不满足，调整相关 persona 的属性后重新输出。
```

### 6.1 Persona 行为指令生成

基于每个 persona 的档案，生成**行为指令集**。这不是给 persona 的提示词补充，
而是一份决策规则表，指导 persona 在具体场景下如何反应。

```markdown
## {姓名} 的行为指令集

### 首次启动行为
- 看到引导页: {认真看完每一页 / 快速翻过 / 直接跳过 / 看到第二页就跳过}
- 看到权限请求: {全部允许 / 逐个考虑 / 全部拒绝 / 不确定就拒绝}
- 看到登录页: {直接注册 / 用第三方登录 / 先看看能不能不登录用}
- 看到首页: {按引导操作 / 先四处看看 / 直奔目标功能}

### 导航行为
- 发现底部 Tab: {从左到右逐个点 / 直接找目标 Tab / 先待在首页不切换}
- 看到搜索框: {忽略 / 试着搜点东西 / 这是第一个使用的功能}
- 遇到深层嵌套: {每层都点进去 / 到第二层就返回 / 根据标题决定是否深入}
- 遇到未知图标: {猜测含义后点击 / 忽略 / 长按看有没有提示}

### 内容消费行为
- 长列表: {浏览前 3 条就够了 / 滑到第 10 条 / 一直滑到底}
- 详情页: {只看标题和摘要 / 完整阅读 / 直接看价格/关键数据}
- 图表/数据: {看不懂跳过 / 仔细研究 / 只关心涨跌颜色}
- 弹窗/广告: {立即关闭 / 阅读内容 / 如果有优惠会看一下}

### 交互与输入行为
- 表单填写: {尽量少填 / 认真填写 / 看到必填项太多就放弃}
- 遇到报错: {再试一次 / 截图然后退出 / 尝试不同输入}
- 功能卡顿: {等待 / 退出重进 / 直接卸载}
- 需要付费: {立即关闭 / 看看价格 / 找免费替代方案}

### 情绪触发器
| 触发条件 | 情绪反应 | 行为后果 |
|---------|---------|---------|
| {正面触发: 如"操作一步就完成"} | 满意/惊喜 | {继续探索/分享给朋友} |
| {负面触发: 如"找不到返回按钮"} | 困惑/烦躁 | {反复尝试/放弃该功能} |
| {强烈负面: 如"闪退丢失输入内容"} | 愤怒 | {卸载/差评/再也不用} |
| {惊喜触发: 如"智能推荐很准"} | 超出预期 | {深度使用/推荐给朋友} |

### 领域专项测试清单（如该 persona 有领域专家原型）

> **此部分仅对"领域专家"类型的 persona 生成。** 普通用户 persona 跳过。
> 清单内容从 persona 档案的"领域专项测试清单"字段复制而来。

```
领域专项测试执行规则：

1. persona 在自然使用过程中，必须覆盖清单中至少 70% 的测试项
2. 每项测试的反馈必须达到"专业反馈标准"中定义的深度
3. 如果测试项对应的功能不存在或找不到入口，这本身就是一条关键反馈
   （记为"缺失：{功能描述}"，比"功能完善"更有价值）
4. 测试过程中发现的非清单内问题同样记录（专家视角的意外发现往往最有价值）

示例（金融 App 期权交易者）：
□ 打开 AAPL 期权链 → 选 30DTE expiry → 记录是否显示 Delta/Gamma/Theta/Vega
□ 尝试构建 Bull Call Spread → 记录是否有策略构建器
□ 查看 IV Rank / IV Percentile → 是否有 IV 分析工具
□ 检查期权链加载速度（合约数 > 200 时）
□ 切换不同 expiry date → 是否支持一键切换
□ 查看期权的 Bid-Ask Spread → 流动性信息是否充分
□ ...（完整清单见 persona 档案）
```

### 反馈深度强制要求

> **所有 persona（无论是否为领域专家）的反馈都必须满足以下深度标准。**

```
反馈深度验证规则：

每条反馈必须通过"具体性检验"——读者能从反馈中还原出：
1. 你在哪个页面/功能上
2. 你具体做了什么操作
3. 你看到了什么结果
4. 与你的预期有什么差距
5. 与你用过的竞品相比如何

不合格反馈特征（出现以下任一即为不合格）：
❌ 纯情绪无事实："界面很好看"、"操作很流畅"、"功能很完善"
❌ 无对比无标准："加载速度还行"（多快算还行？）
❌ 泛泛而谈："社区内容丰富"（什么类型的内容？数量级？质量？）
❌ 缺少上下文："搜索功能不错"（搜了什么词？结果准确吗？）

合格反馈特征：
✅ 有具体操作："我搜索'AAPL'，0.5 秒出结果，联想词准确"
✅ 有量化对比："图表切换到 1 年周期约 1.2 秒，比 Webull 的 <0.5 秒慢一倍"
✅ 有专业判断："RSI 指标支持调整周期（默认 14），但没有背离标记功能"
✅ 有缺失指出："找不到 IV Rank，只有当前 IV 值，无法判断 IV 是否处于历史高位"
```
```

### 6.2 三模式体验执行

根据 Step 3 选择的模式（或子命令参数指定的模式），执行不同级别的 persona 体验。

#### 模式 A: 全自主体验

**所有 persona 独立操控 App**。每个 persona 作为独立 subagent，
通过 mobile-mcp 实际操作 App，按自己的行为指令集自然使用。

```python
# 对每个 persona 派发一个 subagent
# 尽量并行执行（但同一设备同一时间只能有一个 subagent 操作）
# 如果有多台模拟器设备，可以真正并行

for persona in personas:
    Agent(
        prompt="""
        # 你的身份档案
        {persona.full_profile}

        # 你的行为规则
        {persona.behavior_instructions}

        # 你的领域专业知识（如有）
        {persona.domain_knowledge_injection if persona.expert_archetype else "（通用用户，无领域专业知识注入）"}

        # 你的领域专项测试清单（如有）
        {persona.domain_test_checklist if persona.expert_archetype else "（无专项测试清单，按自然使用即可）"}

        # 环境信息
        - App 名称: {app_name}
        - 设备 ID: {device_id}
        - 截图保存目录: {screenshots_dir}/persona-{persona.id}/

        # 你的任务
        你是 {persona.name}，{persona.age} 岁的 {persona.profession}。
        你{persona.motivation_narrative}，所以下载了 {app_name}。

        **请以你自己的方式自然地使用这个 App。**

        重要原则：
        1. **你不是测试员，你是真正的用户。** 不需要覆盖所有功能。
        2. **按你的动机行动。** 你下载这个 App 是为了 {persona.core_motivation}，
           围绕这个目标去使用。
        3. **按你的性格反应。** 你的耐心程度是 {persona.patience}，
           遇到问题时你会 {persona.error_reaction}。
        4. **自然探索。** 你的探索风格是 {persona.exploration_style}。
           不要系统遍历，按直觉走。
        5. **在你觉得"够了"的时候停下来。** 基于你的 patience_score={persona.patience_score}，
           大约使用 {patience_to_duration(persona.patience_score)} 后自然结束。
        6. **持续输出内心独白。** 用第一人称、口语化的方式记录你的真实想法：
           - 困惑: "这个按钮是干嘛的？"
           - 满意: "哦不错，比{竞品}简单多了"
           - 烦躁: "又要登录？刚才不是登录过了吗？"
           - 惊喜: "居然有这个功能，之前用{竞品}的时候一直想要"
        7. **【领域专家专属】如果你有领域专项测试清单：**
           - 在自然使用过程中，主动寻找并测试清单中的项目
           - 找不到的功能同样要记录（"缺失"本身就是关键发现）
           - 你的反馈必须体现你的专业深度——用专业术语，给出量化对比
           - 禁止空洞评价。不要说"期权功能不错"，要说"期权链支持按 expiry 筛选
             但缺少 delta-based strike 过滤，Greeks 只显示 Delta/Theta 缺 Gamma/Vega"
        8. **反馈深度要求（所有 persona 适用）：**
           - 每条反馈必须包含：你在哪、做了什么、看到什么、与预期差距、与竞品对比
           - 禁止纯情绪无事实的评价（如"界面好看"/"功能完善"）
           - 至少 3 处反馈要有量化数据（加载时间、步骤数、信息密度等）

        ## 操作方法
        - 截图: mobile_save_screenshot  device: {device_id}  saveTo: {path}
        - 点击: mobile_click_on_screen_at_coordinates  device: {device_id}  x: {x}  y: {y}
        - 滑动: mobile_swipe_on_screen  device: {device_id}  direction: {dir}
        - 输入: mobile_type_keys  device: {device_id}  text: {text}
        - 元素: mobile_list_elements_on_screen  device: {device_id}
        - 返回: mobile_press_button  device: {device_id}  button: BACK (Android)
                或找返回按钮点击 (iOS)

        ## 输出格式

        ### {persona.name} 的使用日记

        #### 我是谁
        {一句话自我介绍，用口语}

        #### 第一印象（打开 App 的前 30 秒）
        {内心独白，带情绪}

        #### 使用过程
        | # | 我做了什么 | 我的想法 | 情绪 | 截图 |
        |---|----------|---------|------|------|
        | 1 | 打开 App | {独白} | {😊😐😟😡} | {screenshot.png} |
        | 2 | {操作} | {独白} | {emoji} | {screenshot.png} |
        | ... | | | | |

        #### 我完成目标了吗？
        - 我想做的事: {core_motivation}
        - 完成了吗: {完成/部分完成/没完成}
        - 为什么: {具体原因}
        - 花了多久: {时间}

        #### 让我印象深刻的
        - 最喜欢: {1-2 个}
        - 最讨厌: {1-2 个}
        - 最困惑: {1-2 个}

        #### 领域专项测试结果（仅领域专家 persona 填写）
        | # | 测试项 | 结果 | 专业评价 |
        |---|--------|------|---------|
        | 1 | {测试项描述} | ✅存在/❌缺失/⚠️部分支持 | {专业深度的评价，不少于 30 字} |
        | 2 | ... | | |
        | 覆盖率 | {已测试项}/{清单总项} = {百分比} | | |

        #### 专业深度反馈（仅领域专家 persona 填写）
        > 以你的专业视角，给出 3-5 条同行级别的深度反馈。
        > 想象你在一个专业论坛（如 r/options 或 EliteTrader）发帖评价这个 App。
        1. {深度反馈，含专业术语、量化对比、具体缺失}
        2. {深度反馈}
        3. {深度反馈}

        #### 我的结论
        - 满意度: {1-10 分}，因为{原因}
        - 会继续用吗: {会/不确定/不会}，因为{原因}
        - 会推荐给朋友吗: {会/不确定/不会}
        - 一句话: "{用自己的语气说一句总结}"
        """,
        description="Persona #{persona.id} {persona.name} 自主体验 {app_name}"
    )
```

**时间与深度控制**：基于 persona 的 patience_score 决定体验深度。

| patience_score | 预期使用时长 | 探索深度 |
|---------------|------------|---------|
| 0.0 - 0.3 | 2-5 分钟 | 只看首页和 1 个核心功能 |
| 0.3 - 0.6 | 5-15 分钟 | 核心功能 + 1-2 个次要功能 |
| 0.6 - 0.8 | 15-30 分钟 | 大部分功能都会尝试 |
| 0.8 - 1.0 | 30+ 分钟 | 深度探索，包括设置和边缘功能 |

#### 模式 B: 混合体验（默认）

**3 个核心 persona 实际操控 App（模式 A），其余 persona 基于数据评审（模式 C）。**

核心 persona 选择规则（按优先级排序，选取前 3-4 个）：
1. **新手 persona**（技术水平最低 + 领域经验最低的那个）— 测试产品对小白的友好度
2. **目标用户 persona**（最匹配产品核心用户画像的那个）— 测试核心价值交付
3. **挑剔用户 persona**（评价倾向最严格 + 竞品经验最丰富的那个）— 测试竞争力
4. **【新增】领域专家 persona**（如有，选领域专项测试清单最长的那个）— 测试专业功能深度

> **领域专家优先规则**：如果产品涉及专业功能（如金融App的期权/财报、
> 设计工具的图层/蒙版），至少 1 个核心 persona 必须是领域专家。
> 如果只能选 3 个，领域专家替换"挑剔用户"（因为领域专家天然就是最挑剔的用户）。

这 3-4 个 persona 按模式 A 执行自主体验。
其余 persona 按模式 C 执行评审式体验。

两种体验的输出汇合到同一个 `persona-feedback-synthesis.md`。

#### 模式 C: 全评审体验

**所有 persona 基于 Step 5 的探索数据和截图进行评审。**
不操控 App，而是"看"其他人（Step 5 的 subagent）的探索结果。

```python
for persona in personas:
    Agent(
        prompt="""
        # 你的身份档案
        {persona.full_profile}

        # 你的行为规则
        {persona.behavior_instructions}

        # 你的领域专业知识（如有）
        {persona.domain_knowledge_injection if persona.expert_archetype else "（通用用户，无领域专业知识注入）"}

        # 你的领域专项测试清单（如有）
        {persona.domain_test_checklist if persona.expert_archetype else "（无专项测试清单）"}

        # 产品信息（来自其他人的探索记录）
        ## 产品结构
        {exploration_log_content}

        ## 功能截图
        {按模块组织的截图列表，每张截图附带操作路径说明}

        ## 产品详情
        {official_help_data 如果有}

        # 你的任务
        你是 {persona.name}。有人给你展示了 {app_name} 这个 App 的完整截图和功能介绍。
        请以你的身份和视角，给出你的真实看法。

        **重要**：你不是在写产品评测，你是在跟朋友聊天时说起这个 App。
        用你自己的语气（{persona.expression_style}），带上你的偏见和个人经历。

        **反馈深度要求（所有 persona 必须遵守）**：
        - 每条反馈必须包含：你在看哪个截图/功能、具体观察到什么、与预期的差距、与竞品对比
        - 禁止空洞评价（如"界面好看"/"功能完善"/"操作流畅"）
        - 至少 3 处反馈包含量化数据或具体对比

        ## 评审维度（按你的关注重点排序）

        ### 1. 第一印象
        看到首页截图，你的第一反应是什么？
        （基于你的 {attention_pattern}，你最先注意到什么？忽略了什么？）

        ### 2. 功能匹配度
        | 功能 | 对你重要吗 | 原因 | 基于你的经验对比 |
        |------|-----------|------|-----------------|
        | {feature} | {很重要/一般/不在乎} | {原因} | {vs 竞品} |

        ### 3. 易用性预判
        基于你的技术水平（{tech_level}/5），看这些截图和操作路径：
        - 哪些你觉得一看就会？
        - 哪些你觉得需要人教？
        - 哪些你觉得太复杂会直接放弃？

        ### 4. 情绪旅程
        假设你按首页→{主要路径}的顺序使用：
        | 阶段 | 预期情绪 | 原因 |
        |------|---------|------|
        | 打开 App | {emoji + 描述} | {原因} |
        | {阶段2} | {emoji + 描述} | {原因} |
        | ... | | |

        ### 5. 竞品对比
        相比你用过的 {persona.previous_apps}：
        - 比它们好的地方：
        - 不如它们的地方：
        - 差不多的地方：

        ### 6. 付费/续费决策
        - 你会为这个产品付费吗？{会/不会/要看}
        - 你的付费心理价位：{价格范围}
        - 触发付费的条件：{什么功能/体验让你愿意掏钱}

        ### 7. 领域专项评审（仅领域专家 persona 填写）
        基于你的专业知识和测试清单，对截图中能看到的专业功能逐项评审：

        | # | 测试项 | 从截图中观察到 | 专业评价 |
        |---|--------|--------------|---------|
        | 1 | {测试项} | ✅可见/❌未见/⚠️不完整 | {专业深度评价} |
        | 2 | ... | | |

        **专业深度反馈**（3-5 条，想象你在专业论坛发帖）：
        1. {深度反馈}
        2. {深度反馈}
        3. {深度反馈}

        ### 8. 最终判决
        - 满意度: {1-10}
        - 会继续用: {会/不确定/不会}
        - 一句话: "{口语化总结}"
        """,
        description="Persona #{persona.id} {persona.name} 评审 {app_name}"
    )
```

### 6.3 Persona 反馈聚合

所有 persona 体验/评审完成后，主 agent 收集所有反馈，执行多维度聚合分析。

**输出**：`{BASE_DIR}/persona-feedback-synthesis.md`

```markdown
# {App 名称} Persona 反馈综合分析

## 参与 Persona 概览

| # | 姓名 | 标签 | 年龄 | 技术 | 领域 | 体验模式 | 满意度 |
|---|------|------|------|------|------|---------|--------|
| 1 | {name} | {tag} | {age} | {tech} | {domain} | A/C | {score}/10 |
| ... | | | | | | | |
| — | **平均** | | | | | | **{avg}/10** |

## 一、共识发现（多数 persona 提到）

### 共同亮点
| 发现 | 提到的 persona | 代表性原声 |
|------|--------------|-----------|
| {亮点} | #{1}#{3}#{5} (3/N) | "{persona.name}: {原话}" |

### 共同问题
| 问题 | 提到的 persona | 严重度 | 代表性原声 |
|------|--------------|--------|-----------|
| {问题} | #{2}#{4}#{6} (3/N) | 高/中/低 | "{persona.name}: {原话}" |

## 二、分群差异

### 新手 vs 专家
| 维度 | 新手 persona 的看法 | 专家 persona 的看法 |
|------|-------------------|-------------------|
| {维度} | {观点} | {观点} |

### 年轻用户 vs 年长用户
| 维度 | 年轻 persona | 年长 persona |
|------|-------------|-------------|
| {维度} | {观点} | {观点} |

### 刚需用户 vs 好奇用户
| 维度 | 刚需 persona | 好奇 persona |
|------|-------------|-------------|
| {维度} | {观点} | {观点} |

## 三、情绪热力图

| 功能/页面 | 正面情绪 | 中性 | 负面情绪 | 净情绪分 |
|----------|---------|------|---------|---------|
| {功能} | {N} 人 | {N} 人 | {N} 人 | {+/-score} |
| ... | | | | |

## 四、流失风险点

按 persona 的"放弃"反应和"不会继续用"判断，识别潜在流失节点：

| 风险点 | 受影响 persona | 用户类型 | 流失原因 | 估计影响面 |
|--------|--------------|---------|---------|-----------|
| {节点} | #{N} | {类型} | {原因} | {比例} |

## 五、付费意愿分析

| Persona | 会付费 | 心理价位 | 触发条件 |
|---------|--------|---------|---------|
| {name} | {是/否/看情况} | {价格} | {条件} |

**付费转化预估**：{N}/{total} 的 persona 表示愿意付费，
主要集中在 {用户类型} 群体。关键付费触发条件是 {条件}。

## 六、原声集锦（按情绪分类）

### 最正面的反馈
> "{persona.name}（{persona.tag}）: {原话}" — 满意度 {score}/10

### 最负面的反馈
> "{persona.name}（{persona.tag}）: {原话}" — 满意度 {score}/10

### 最有洞察的反馈
> "{persona.name}（{persona.tag}）: {原话}"
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

    ## 数据源约束（严格执行）
    **只允许使用官方渠道的数据。严禁引用任何第三方评测网站、博客、论坛的数据。**
    所有费用、功能描述、交易规则等事实性信息必须直接从官方网站或 App 内获取。

    ## 任务
    1. 用 WebSearch 搜索产品**官方网站和帮助中心**（注意只找官方域名）：
       - "{app_name} 帮助中心" / "{app_name} help center FAQ"
       - "site:{official_domain} 费用" / "site:{official_domain} pricing"
       - "{app_name} 使用教程 入门" / "{app_name} 常见问题"

    2. 找到官方帮助中心 URL 后，用 WebFetch 抓取：
       - 帮助中心首页（获取分类和热门文章列表）
       - **官方费用/定价页面**（逐项记录准确数字）
       - 前 10 篇热门帮助文章
       - FAQ 页面 / 新手引导页面

    3. 搜索 App Store / Google Play **官方描述**（开发商填写的部分）：
       - 官方功能描述 / 最新版本更新说明

    ## 输出
    ### 官方帮助中心
    - URL: {地址}
    - 分类结构: {所有分类}
    - 热门文章: {标题 + 摘要}

    ### 官方费用/定价（直接从官方页面抓取）
    | 项目 | 费用 | 来源 URL |
    |------|------|----------|
    （每项费用必须附带官方来源 URL）

    ### 官方 FAQ
    | 问题 | 官方解答摘要 | 来源 URL |
    |------|-------------|----------|

    ### 用户高频问题（仅 App Store 评价，非第三方）
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

### 1.3 覆盖说明

> 本说明书覆盖率详见审计文档 `mece-coverage-audit.md`。
> 功能树中每个节点的覆盖状态均已在该文件中逐项检查。

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
> 详见 `mece-coverage-audit.md`（审计文档，不包含在用户手册中）
```

**MECE 覆盖检查表单独存放**：说明书完成后，自动生成 `{BASE_DIR}/mece-coverage-audit.md`：

```markdown
# MECE 覆盖检查表

> 生成日期: {YYYY-MM-DD}
> 对应说明书: manual-full.md

## 覆盖统计

| 一级模块 | 二级功能数 | 已覆盖 | 覆盖率 | 遗漏原因 |
|----------|-----------|--------|--------|---------|
| {模块 A} | {N} | {N} | 100% | - |
| {模块 B} | {N} | {M} | {X}% | {原因} |
| ... | | | | |
| **合计** | **{总数}** | **{已覆盖}** | **{X}%** | |

## 逐项检查

| 功能树节点 | 说明书章节 | 截图证据 | 状态 |
|-----------|-----------|---------|------|
| A1. {功能} | 四、4.1 | {截图文件名} | ✅ |
| A2. {功能} | 四、4.2 | {截图文件名} | ✅ |
| B1. {功能} | - | - | ❌ 需登录 |
| ... | | | |
```

### 7c. 功能/体验优化建议（Persona 加权版）

写入 `{BASE_DIR}/optimization.md`。

数据来源：Step 5 产品探索 + Step 6 persona 反馈综合分析（`persona-feedback-synthesis.md`）。

**优先级确定逻辑**：
- P0（必须修复）: 60%+ persona 提到 + 任何"流失风险点"关联的问题
- P1（应该改进）: 30-60% persona 提到，或核心用户 persona 强烈反馈
- P2（可以优化）: <30% persona 提到，但有洞察价值

```markdown
# {App 名称} 优化建议报告

## 数据来源
- 产品功能探索（Step 5）：{N} 个模块深度探索
- Persona 模拟（Step 6）：{N} 个拟人用户体验/评审
- 体验模式：{A/B/C}
- 反馈综合分析：`persona-feedback-synthesis.md`

## Persona 参与概览

| Persona | 标签 | 满意度 | 会继续用 | 会付费 |
|---------|------|--------|---------|--------|
| {name} | {tag} | {score}/10 | {是/否} | {是/否} |
| ... | | | | |
| **汇总** | | **{avg}/10** | **{ratio}** | **{ratio}** |

## 优化建议总览

| 优先级 | 建议 | 影响 persona | 影响面 | 预期收益 | 成本 | ROI |
|--------|------|-------------|--------|----------|------|-----|
| P0 | {建议} | {N}/{total} | {用户占比估计} | {收益} | {成本} | {高/中/低} |
| P1 | {建议} | {N}/{total} | {用户占比估计} | {收益} | {成本} | {高/中/低} |
| P2 | {建议} | {N}/{total} | {用户占比估计} | {收益} | {成本} | {高/中/低} |

## 详细建议

### P0-1: {建议标题}

**问题描述**
- **现状**：{当前体验描述}
- **问题本质**：{根因分析}

**Persona 证据**
| Persona | 反应 | 原声引用 |
|---------|------|---------|
| {name}（{tag}）| {行为/情绪} | "{原话}" |
| {name}（{tag}）| {行为/情绪} | "{原话}" |

**影响分析**
- 影响的 persona 数量：{N}/{total}（{percentage}%）
- 影响的用户类型：{列出}
- 潜在流失风险：{高/中/低} — {N} 个 persona 表示会因此"不继续使用"
- 付费影响：{有/无} — {N} 个 persona 表示这影响了付费决策

**建议方案**
- 方案描述：{改进方案}
- 预期效果：{具体改善}
- 估计成本：{开发量}

**不同用户群的优先级差异**
| 用户群 | 对这个问题的感受 | 优先级 |
|--------|---------------|--------|
| 新手用户 | {感受} | {高/中/低} |
| 核心用户 | {感受} | {高/中/低} |
| 专家用户 | {感受} | {高/中/低} |

---

### P0-2: {建议标题}
...（每条建议重复以上结构）

### P1-1: ...
### P2-1: ...

## 用户群专属建议

按用户类型汇总，每类用户最想要的 Top 3 改进：

### 新手用户
1. {建议} — 来自 persona #{N} 等
2. {建议}
3. {建议}

### 核心用户
1. ...

### 专家用户
1. ...

## 附录：Persona 完整反馈
（链接到各 persona 的完整反馈文件或内联展示摘要）
```

---

### 7d. 报告生成质量保障规则

**每份报告由独立 subagent 生成，主 agent 负责验证。**

```
报告生成流程：

1. 每份报告由独立 subagent 生成（一报告一 agent）
   - manual-full.md → 独立 subagent
   - report.md → 独立 subagent
   - optimization.md → 独立 subagent

2. 主 agent 在分派报告生成 subagent 时，提供结构化输入：
   - 不传原始 subagent 输出，而是指向 exploration-reports/ 目录
   - "请读取 {BASE_DIR}/exploration-reports/ 下的所有文件，
     综合这些探索数据生成 {报告类型}"

3. 报告生成后，主 agent 验证：
   a) 文件是否存在且非空
   b) 行数是否合理：
      - manual-full.md > 500 行
      - report.md > 300 行
      - optimization.md > 200 行
   c) 是否包含所有必需章节（按模板检查）
   d) 关键数据是否有截图依据

4. 验证失败的报告重新生成（最多重试 2 次）

5. 报告数据一致性检查：
   - 报告中引用的数据必须来自探索数据（如"支持8种订单类型"
     必须与 exploration-reports/ 中实际记录的一致）
   - 不可编造未探索到的功能或数据
```

**截图-发现关联验证**：

```
报告完成后，主 agent 执行引用验证：

1. 提取报告中所有截图引用（如 screenshots/xxx.png）
2. 检查每个引用的截图文件是否存在
3. 不存在的标记为 [证据缺失]
4. 关键数据点（费率、功能数量、订单类型列表等）
   必须标注截图文件名作为证据来源：

   正确："支持 8 种订单类型（见截图 trade-03-order-types.png）：
          Limit, Market, Stop_Limit, Stop, Limit-If-Touched,
          Trailing Stop, Trailing Limit, Stop_Market"

   错误："支持 8 种订单类型"（无截图依据）

5. 对 [证据缺失] 的数据点，要么补截图，要么标注"未经截图验证"
```

---

## Step 7.5: 经验沉淀（Experience Bank）

> **设计哲学**：借鉴 skill-se-kit 的 Skill Bank + Experience Bank 双存储模式，
> 但用纯 markdown 实现，零依赖、人工可审查、无回归风险。
> 经验只注入 context 辅助决策，不自动修改 SKILL.md 本身。

每次 review 完成后（Step 7 报告生成之后），主 agent 执行经验提取和沉淀。

### 7.5.1 Experience Bank 目录结构

```
reviews/
  _experience_bank/                       # 经验银行根目录（跨产品共享）
    _index.md                             # 经验索引（按类别+置信度排序）
    categories/
      finance-investing.md                # 金融/投资类 App 经验
      social-content.md                   # 社交/内容类 App 经验
      tools-productivity.md               # 工具/效率类 App 经验
      ecommerce.md                        # 电商/消费类 App 经验
      education.md                        # 教育/学习类 App 经验
      health-fitness.md                   # 健康/运动类 App 经验
      travel.md                           # 出行/旅行类 App 经验
      general.md                          # 跨品类通用经验
    personas/
      effective-combinations.md           # 高效 persona 属性组合记录
      anti-patterns.md                    # 无效/冗余的 persona 模式
    strategies/
      exploration-patterns.md             # 探索策略经验
      report-quality.md                   # 报告质量经验
    per-app/                              # 单 App 版本间对比经验（append-only）
      moomoo/
        _app-index.md                     #   该 App 的经验索引 + 版本历史
        feature-inventory.md              #   功能清单快照（每次 review 追加版本段）
        version-diffs.md                  #   版本间差异记录（append-only 时间线）
        app-specific-insights.md          #   该 App 专属洞察（跨版本积累）
      xiaohongshu/
        _app-index.md
        ...
```

### 7.5.2 经验提取流程

Step 7 报告生成完成后，主 agent 执行以下提取：

```
经验提取维度（6 类）：

1. Persona 有效性经验
   - 哪些 persona 产出了最有洞察力的反馈？（分析 persona-feedback-synthesis.md）
   - 哪些 persona 的反馈高度重复/没有新增价值？
   - 哪些属性组合对这个产品类别特别有效？
   → 写入 personas/effective-combinations.md

2. 产品类别专属经验
   - 这类 App 哪些功能区域容易被忽略但很重要？
   - 这类 App 的用户最关心什么维度？
   - 哪些 persona 维度对这类 App 特别关键？
   → 写入 categories/{category}.md

3. 探索策略经验
   - 哪些探索路径效率最高？
   - 哪些页面/功能在第一屏以下藏着重要信息？
   - 哪些交互模式需要特别测试（如长按、左滑、3D Touch）？
   → 写入 strategies/exploration-patterns.md

4. 报告质量经验
   - 用户对报告哪些部分反馈最有价值？
   - 哪些分析维度用户觉得多余或缺失？
   - 优化建议的 P0/P1/P2 判断是否准确？
   → 写入 strategies/report-quality.md

5. 失败教训（anti-patterns）
   - 哪些 persona 设定导致了不真实/低质量的反馈？
   - 哪些探索策略浪费了时间但没有发现？
   - 哪些报告模板的部分经常是空话？
   → 写入 personas/anti-patterns.md

6. 跨品类通用洞察
   - 不论什么产品都适用的 persona/探索/报告经验
   → 写入 categories/general.md

7. ⭐ 单 App 版本间经验（Per-App Experience）
   - 该 App 的功能清单快照（本次 review 发现了哪些功能）
   - 与上次 review 的版本差异（新增/移除/变更了什么）
   - 该 App 特有的探索难点和技巧（如 moomoo 的交易密码阻断处理）
   - 该 App 的 Persona 反馈趋势（跨版本满意度变化）
   → 写入 per-app/{app_name}/ 下的对应文件
```

### 7.5.3 经验条目格式

每条经验是一个结构化条目，带元数据用于后续检索和置信度衰减：

```markdown
### EXP-{category}-{seq}: {经验标题}

- **置信度**: {0.0-1.0}（初始根据证据强度设定，随验证次数提升）
- **来源**: {app_name} review ({date})
- **验证次数**: {N}（后续 review 中被验证为有效的次数）
- **最后验证**: {date}
- **产品类别**: {category}
- **适用条件**: {什么情况下这条经验适用}

**经验内容**：
{具体的可操作经验描述}

**证据**：
{来自哪个 persona 的什么反馈，或哪个探索发现支持这条经验}

**反例**（如有）：
{什么情况下这条经验不适用}
```

**示例**：

```markdown
### EXP-finance-001: 金融 App 必须包含"风控规则理解度"维度的 persona

- **置信度**: 0.7
- **来源**: moomoo review (2026-03-29)
- **验证次数**: 1
- **最后验证**: 2026-03-29
- **产品类别**: finance-investing
- **适用条件**: 涉及交易功能的金融 App

**经验内容**：
生成金融 App 的 persona 时，"风控规则理解度"是一个关键维度。
不理解风控规则的用户（如不知道 T+2 结算、PDT 规则）会在关键交易
环节遇到完全不同的困惑点，这类反馈对产品改进非常有价值。
建议至少 1 个 persona 的风控理解度设为最低。

**证据**：
Persona #3（王大妈，55 岁退休教师）在尝试买入操作时，
完全不理解"限价单"和"市价单"的区别，这暴露了产品在
订单类型教育上的严重缺失——而其他 persona 都跳过了这个问题。
```

### 7.5.4 经验索引维护

每次沉淀新经验后，更新 `_experience_bank/_index.md`：

```markdown
# Experience Bank Index

> 上次更新: {date} | 总经验数: {N} | 覆盖品类: {N}

## 按品类

| 品类 | 经验数 | 最新来源 | 高置信度(>0.7) |
|------|--------|---------|---------------|
| finance-investing | {N} | {app} ({date}) | {N} |
| social-content | {N} | {app} ({date}) | {N} |
| ... | | | |

## 按类型

| 类型 | 数量 | 文件 |
|------|------|------|
| Persona 有效组合 | {N} | personas/effective-combinations.md |
| Persona 反模式 | {N} | personas/anti-patterns.md |
| 品类专属经验 | {N} | categories/*.md |
| 探索策略 | {N} | strategies/exploration-patterns.md |
| 报告质量 | {N} | strategies/report-quality.md |

## 高置信度经验 Top 10
1. [EXP-{id}] {title} — 置信度 {score}, 验证 {N} 次
2. ...
```

### 7.5.5 置信度演化规则

```
经验条目的置信度不是静态的，会随时间和验证情况变化：

初始置信度设定：
- 基于单个 persona 的反馈 → 0.4
- 基于多个 persona 的共识 → 0.6
- 基于用户（Eli）直接反馈确认 → 0.8
- 基于跨多次 review 验证 → 0.9

置信度提升：
- 后续 review 中被验证有效 → +0.1（上限 1.0）
- 用户手动确认 → 直接设为 0.9

置信度衰减：
- 后续 review 中被发现不适用 → -0.15
- 超过 90 天未被验证 → -0.05（最低不低于 0.2）
- 置信度降到 0.2 以下 → 标记为 [待清理]，下次 review 时提示用户是否删除

注入阈值：
- 置信度 >= 0.5 的经验才会在 Step 5.5 中被注入到 persona 生成 context
- 置信度 >= 0.7 的经验会被标记为 [高置信]，优先展示
```

### 7.5.6 Per-App 版本间经验（Version Tracking）

**设计哲学**：同一 App 的多次 review（版本更新、功能迭代）是最有价值的纵向数据。
通过 append-only 的版本记录，实现跨版本的功能对比、回归检测和改进趋势追踪。

#### 目录与文件

每个 App 在 `per-app/{app_name}/` 下有 4 个文件：

**1. `_app-index.md` — App 经验索引**

```markdown
# {App 名称} Experience Index

> App ID: {bundle_id / package_name}
> 品类: {finance-investing / social-content / ...}
> 首次 review: {date}
> 最近 review: {date}
> 总 review 次数: {N}

## 版本历史

| # | 日期 | App 版本 | 平台 | 功能数 | Persona 均分 | 报告目录 |
|---|------|---------|------|-------|-------------|---------|
| 1 | 2026-03-29 | 14.2.1 | iOS | 35 | 6.8/10 | moomoo/20260329/ |
| 2 | 2026-05-15 | 14.5.0 | iOS | 38 | 7.2/10 | moomoo/20260515/ |
| ... | | | | | | |

## 关键指标趋势
| 指标 | v14.2.1 | v14.5.0 | 变化 |
|------|---------|---------|------|
| 功能总数 | 35 | 38 | +3 |
| Persona 均分 | 6.8 | 7.2 | +0.4 |
| P0 问题数 | 5 | 3 | -2 |
| 探索覆盖率 | 85% | 95% | +10% |
```

**2. `feature-inventory.md` — 功能清单快照（append-only）**

每次 review 追加一个版本段，不修改历史段。用于跨版本功能对比。

```markdown
# {App 名称} Feature Inventory

---
## [v14.2.1] 2026-03-29

### 功能树
├── Watchlists (5 子功能)
│   ├── All / US / CA / HK / Custom tabs
│   ├── 4 种视图模式
│   └── 排序、横屏
├── Markets (8 子功能)
│   ├── Overview / US / CA / HK / Crypto / Futures / AutoBuy / Screeners
│   └── ...
├── Trade (受限-需交易密码)
├── ...
总计: 35 个功能点

---
## [v14.5.0] 2026-05-15

### 功能树
├── Watchlists (6 子功能) [+1: Smart Lists]
│   ├── All / US / CA / HK / Custom / ⭐Smart tabs
│   ...
├── Markets (9 子功能) [+1: Options Scanner]
│   ...
总计: 38 个功能点

### 与上一版本差异
| 变更类型 | 功能 | 详情 |
|---------|------|------|
| ⭐新增 | Smart Lists | Watchlists 新增智能分组 tab |
| ⭐新增 | Options Scanner | Markets 新增期权扫描器 |
| ⭐新增 | AI Portfolio Review | Me→AI 新增组合分析 |
| 🔄变更 | Chart 指标 | 技术指标从 45 种增加到 52 种 |
| 🗑移除 | - | 无 |
```

**3. `version-diffs.md` — 版本间差异时间线（append-only）**

记录每次 review 发现的与上一版本的差异。是功能层面的 changelog。

```markdown
# {App 名称} Version Diffs

---
## [v14.2.1 → v14.5.0] 2026-05-15

### 新增功能
1. **Smart Lists**（Watchlists Tab）
   - 功能：基于 AI 的智能股票分组
   - 入口：Watchlists → Smart tab
   - 截图依据：smart-lists-01.png
   - Persona 反馈：7.5/10 均分，"终于有智能分组了"(#2张伟)，"分组逻辑不透明"(#5陈明)

2. **Options Scanner**（Markets Tab）
   - 功能：按条件筛选期权合约
   - 入口：Markets → Options Scanner
   - 截图依据：options-scanner-01.png ~ 03.png
   - Persona 反馈：6.0/10 均分，"筛选条件太少"(#1期权交易者)

### 改进功能
1. **Chart 技术指标** 从 45 种增加到 52 种
   - 新增：VWAP, Ichimoku Cloud, Keltner Channel, ...
   - Persona 反馈："VWAP 终于有了！"(#3 日内交易者)

### 回归/退步
1. **Watchlists 加载速度**
   - v14.2.1: 正常(<1s)
   - v14.5.0: 慢(2-3s)，可能因为 Smart Lists 增加了计算开销
   - 影响 Persona: 3/6 提到加载变慢

### 未解决的老问题（从上版本遗留）
| 问题 | 上版本状态 | 本版本状态 | P级 |
|------|-----------|-----------|-----|
| Trade 密码阻断 | 未解决 | 未解决 | P1 |
| Comments 加载慢 | 慢(3s+) | 正常(1s) | ✅已修复 |
```

**4. `app-specific-insights.md` — App 专属洞察（跨版本积累）**

该 App 特有的经验，不适用于同类别其他 App。格式同 7.5.3 的经验条目。

```markdown
# {App 名称} Specific Insights

### APP-moomoo-001: moomoo 的交易密码是核心阻断点
- **置信度**: 0.9
- **验证次数**: 2 (v14.2.1, v14.5.0 均确认)
- **最后验证**: 2026-05-15

**洞察**：
moomoo 在 Trade 页面有独立的交易密码（非账户登录密码）。
这导致每次 review 的交易流程探索都被阻断。
建议在 review 开始前就通过 AskUserQuestion 获取交易密码。

### APP-moomoo-002: moomoo 的 MSFT 详情页有 7 个子 Tab
- **置信度**: 0.8
- **验证次数**: 1

**洞察**：
股票详情页子 Tab 数量多（Chart/Options/Fundamentals/ETFs/Comments/News/Analysis），
需要为每只代表性股票的详情页分配 2-3 个 subagent 才能覆盖完全。
```

#### Per-App 经验提取流程

```
Step 7 报告完成后，针对 per-app 经验执行：

1. 检查 per-app/{app_name}/ 是否存在：
   - 不存在（首次 review）→ 创建目录 + 初始化 4 个文件
   - 存在（重复 review）→ 进入版本对比流程

2. 首次 review：
   a. 生成 _app-index.md（版本历史第 1 行）
   b. 生成 feature-inventory.md（第 1 个版本段）
   c. version-diffs.md 写入 "首次 review，无对比基线"
   d. 提取 app-specific-insights.md 初始条目

3. 重复 review（版本对比流程）：
   a. 读取上一版本的 feature-inventory.md 最新段
   b. 与本次探索的功能树逐项对比（diff）
   c. 生成差异报告：
      - 新增功能列表（本次有、上次无）
      - 移除功能列表（上次有、本次无）
      - 变更功能列表（同一功能但内容/布局变化）
      - 回归检测（上次好的指标本次变差）
   d. 追加到 feature-inventory.md（新版本段）
   e. 追加到 version-diffs.md（新差异段）
   f. 更新 _app-index.md 版本历史表 + 指标趋势
   g. 更新 app-specific-insights.md（验证或新增条目）

4. Provenance（溯源）：
   - 每条差异记录标注截图依据文件名
   - 每条 Persona 反馈标注来源 Persona ID
   - 版本段标注该次 review 的完整报告目录路径
```

#### 版本对比注入（下次 review 时使用）

```
当重复 review 同一 App 时，在 Step 5 广度扫描开始前注入：

## 上次 review 快照（来自 Per-App Experience Bank）

### 已知功能清单（v{上一版本}，{日期}）
{从 feature-inventory.md 提取的最新版本段的功能树}

### 已知阻断点
{从 app-specific-insights.md 提取的阻断/受限记录}

### 关注点
- 请重点检查以下区域是否有变化：{上次 P0/P1 问题列表}
- 上次遗留的未探索入口：{列表}

## 本次 review 目标
在常规全量探索基础上，额外关注：
1. 与上版本的功能差异（新增/移除/变更）
2. 上次 P0/P1 问题是否已修复
3. 性能是否有回归
```

### 7.5.7 主 agent 经验提取 prompt

```
Step 7 报告完成后，主 agent 执行以下操作：

1. 读取本次 review 的所有产出文件：
   - persona-profiles.md（persona 设定）
   - persona-feedback-synthesis.md（反馈综合）
   - report.md / optimization.md（报告产出）
   - exploration-queue.md（探索队列最终状态）

2. 分析并提取经验，按 7.5.2 的 7 个维度分别提取（含第 7 维度 per-app）

3. 检查 _experience_bank/ 是否已存在：
   - 如果不存在 → 创建完整目录结构（含 per-app/）+ 写入首批经验
   - 如果存在 → 读取已有经验，避免重复，合并或更新

4. 对已有经验进行验证更新：
   - 本次 review 与已有经验一致 → 验证次数 +1，置信度 +0.1
   - 本次 review 与已有经验矛盾 → 置信度 -0.15，添加反例说明

5. ⭐ Per-App 版本间经验沉淀：
   a. 检查 per-app/{app_name}/ 是否存在
   b. 首次 review → 初始化 4 个文件（见 7.5.6）
   c. 重复 review → 执行版本对比流程：
      - 读取上版本功能清单 → 与本次 diff → 写入 version-diffs.md
      - 追加本次功能清单到 feature-inventory.md
      - 更新 _app-index.md 版本历史 + 指标趋势
      - 验证/新增 app-specific-insights.md 条目

6. 更新 _index.md

7. 向用户简要汇报：
   "本次 review 沉淀了 {N} 条新经验，验证了 {M} 条已有经验，
    {K} 条经验的置信度发生变化。
    Per-App：{首次建立 / 与 v{上版本} 对比完成，{X} 项功能变更}。
    详见 _experience_bank/_index.md。"
```

---

## Step 8: 经验注入（Experience Retrieval）

> 此步骤在 **下一次** review 时生效。首次 review 时 experience bank 为空，跳过此步。

### 8.1 触发时机

在 Step 5.5（产品用户画像分析）**之前**执行。即流程变为：

```
Step 5 自主探索 → Step 8 经验注入 → Step 5.5 用户画像分析 → Step 6 拟人模拟
```

### 8.2 经验检索逻辑

```
1. 确定本次 review 的产品类别（可从 Step 5 探索结果中推断）

2. 检索相关经验：
   a. categories/{category}.md — 品类专属经验（全部加载）
   b. categories/general.md — 通用经验（全部加载）
   c. personas/effective-combinations.md — 有效 persona 组合（全部加载）
   d. personas/anti-patterns.md — 无效模式（全部加载）
   e. strategies/exploration-patterns.md — 探索策略（选择性加载）
   f. ⭐ per-app/{app_name}/ — 该 App 的版本间经验（如存在，全部加载）
      - _app-index.md（版本历史 + 指标趋势）
      - feature-inventory.md 最新版本段（上次功能清单）
      - version-diffs.md 最新段（上次发现的差异）
      - app-specific-insights.md（该 App 专属阻断点和技巧）

3. 过滤：只保留置信度 >= 0.5 的经验条目
   （per-app 经验不受此过滤，因为是该 App 的直接经验，全部加载）

4. 排序：按 置信度 × 相关度 降序
   - per-app 经验相关度 = 1.0（最高优先级）
   - 品类完全匹配的相关度 = 0.8
   - 通用 = 0.5

5. 截断：最多注入 20 条品类/通用经验 + per-app 经验不限
```

### 8.3 注入方式

检索到的经验以 context block 的形式注入到后续步骤：

```markdown
## 历史经验参考（来自 Experience Bank）

> 以下经验来自之前的产品 review，按置信度排序。
> 请在生成 persona 和制定探索策略时参考，但不要盲从——
> 每个产品都有独特性，经验仅供参考。

### ⭐ 该 App 版本间经验（最高优先级）
> 以下来自对同一 App 的历史 review，直接适用于本次。

**上次 review**: v{版本} ({日期})，Persona 均分 {X}/10
**上次功能清单**: {功能树摘要}
**上次未解决的 P0/P1 问题**: {列表}
**已知阻断点**: {如"Trade 需要交易密码"}
**探索技巧**: {如"MSFT 详情页有 7 个子 Tab，需要 2-3 个 subagent"}

**本次重点关注**:
1. 与 v{上版本} 的功能差异（新增/移除/变更）
2. 上次 P0/P1 问题是否已修复
3. 性能是否有回归（特别关注上次标注的慢页面）

### 品类经验（{category}）
- [高置信] EXP-{id}: {经验内容摘要}
- [高置信] EXP-{id}: {经验内容摘要}
- EXP-{id}: {经验内容摘要}
...

### Persona 经验
- [高置信] 有效组合: {描述}
- 反模式: {描述}（避免）
...

### 探索策略经验
- {经验}
...
```

**注入点**：

1. **Step 5.5 的主 agent context**：品类经验 + persona 经验 → 影响 persona 维度选择和属性采样
2. **Step 6.0 的 persona 生成 prompt**：有效组合经验 → 影响具体 persona 属性值
3. **Step 5.2 的 subagent prompt**（可选）：探索策略经验 → 影响探索深度和重点

### 8.4 经验使用记录

注入经验后，记录哪些经验被实际使用：

```
在本次 review 的 exploration-log.md 末尾追加：

## Experience Bank 使用记录
| 经验 ID | 标题 | 置信度 | 是否采纳 | 备注 |
|---------|------|--------|---------|------|
| EXP-{id} | {title} | {score} | 是/否/部分 | {简要说明} |

这份记录在 Step 7.5 经验沉淀时用于更新置信度。
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

## 数据源约束（严格执行）

### 禁止依赖第三方数据

报告中涉及的所有产品事实性信息（费用、功能列表、账户类型、交易规则等），**只允许**使用以下来源：

| 允许的来源 | 说明 |
|-----------|------|
| **App 内页面** | 直接在 App 中截图获取的信息（费用页面、帮助中心、设置等） |
| **官方网站** | 产品开发商自己的官网（如 moomoo.com/ca） |
| **官方帮助中心** | 产品开发商运营的帮助中心页面 |
| **App Store / Google Play 官方描述** | 开发商自己填写的应用描述 |

**严格禁止**使用以下来源的数据写入报告：

| 禁止的来源 | 原因 |
|-----------|------|
| 第三方评测网站（如 stocktrades.ca、NerdWallet 等） | 数据可能过时或不准确 |
| 第三方博客/文章 | 信息可能有误 |
| 用户论坛（如 Reddit）中的非官方数据 | 未经验证 |
| 竞品对比网站 | 数据口径可能不一致 |

### 验证流程

对于费用、价格、交易规则等关键数据：
1. **优先从 App 内获取**：导航到 App 的费用/定价页面，截图作为依据
2. **其次从官方网站获取**：用 WebFetch 抓取官方费用页面，逐项核实
3. **如果无法从以上渠道获取**：在报告中标注"未经官方验证"，不要填写不确定的数据

---

## 深度探索协议（严格执行）

### 最高原则：像真实用户一样使用，而不只是浏览

**你不是在"看"App，你是在"用"App。** 报告中最有价值的内容是你作为用户的真实使用感受，而不是页面元素的罗列。

对于每一个功能，你必须**实际执行操作**，而不仅仅是查看页面：

| 功能类型 | 必须执行的操作 | 不够的做法 |
|---------|--------------|-----------|
| 搜索功能 | 实际输入关键词搜索，观察搜索建议、结果排序、空结果处理 | 只截图搜索框 |
| 交易/下单 | 实际填写交易表单（选股票、选订单类型、输入价格数量），走到确认前一步 | 只看交易页面布局 |
| AI 对话 | 实际输入问题与 AI 对话，评估回答质量和响应速度 | 只截图 AI 入口按钮 |
| 社区/评论 | 实际浏览帖子、尝试发表评论（如果允许） | 只看社区首页 |
| 设置页面 | 实际点击每个设置项，查看可选值和效果 | 只看设置列表 |
| 筛选/排序 | 实际切换不同筛选条件，观察结果变化 | 只看筛选选项存在 |
| 输入框/表单 | 实际输入内容，测试输入验证、键盘弹出、自动补全 | 只看空表单 |
| 图表/数据可视化 | 实际操作图表（缩放、切换周期、添加指标），观察交互响应 | 只截图默认图表 |
| 通知/提醒 | 实际设置一个提醒，观察设置流程 | 只看提醒入口 |
| 分享功能 | 实际点击分享，观察分享选项和内容预览 | 只看分享按钮 |

### 实际使用测试清单（Subagent 必须执行）

每个模块探索时，subagent 必须完成以下实际操作（根据模块适用性）：

```
□ 在搜索框中输入至少 2 个不同关键词，记录搜索体验
□ 在有输入框的页面实际输入内容
□ 在有筛选/排序的页面实际切换至少 3 种不同选项
□ 在有图表的页面实际切换时间周期、添加/切换技术指标
□ 在有列表的页面实际点击至少 3 个不同条目进入详情
□ 对于核心业务流程（如下单），走完整个流程到确认前一步
□ 对于 AI/聊天功能，实际发送问题并记录回答
□ 对于社区功能，实际浏览多个帖子，尝试互动操作
□ 在报告中用第一人称描述"我做了什么，感受到什么"
```

### 报告中的体验描述要求

**错误的写法**（纯描述型）：
> "Trade 页面有 Buy/Sell 按钮、订单类型下拉、价格输入框和数量输入框。"

**正确的写法**（使用体验型）：
> "我尝试买入 2 股 BTE：选择 Buy → Limit 订单 → 输入价格 6.01 → 输入数量 2。整个流程流畅，但发现价格输入框的加减按钮步长是 0.01，对于低价股合适但对高价股（如 NVDA $166）调整太慢。订单类型切换时有可视化教学弹窗，对新手很友好。实际从选股到填完表单大约需要 15 秒。"

### 核心原则：穷尽遍历

每个页面的探索必须做到以下所有步骤，缺一不可：

### 1. 滚动扫描协议（Scroll Scan Protocol）

**每个页面都必须执行完整的滚动扫描。** 这是整个 skill 中最容易失败的环节——
agent 天然倾向于"看了第一屏就觉得够了"。以下协议强制量化追踪。

```
对每个页面执行以下精确序列：

STEP A: 初始快照
  → mobile_list_elements_on_screen → 记为 snapshot_0，元素数 = N0
  → mobile_save_screenshot（{page}-scroll-0-top.png）

STEP B: 滚动循环（增强版，含无限滚动检测 + 接力机制）
  → scroll_count = 0
  → same_count = 0
  → LOOP:
      → mobile_swipe_on_screen  direction: up
      → mobile_list_elements_on_screen → 记为 snapshot_{scroll_count+1}
      → scroll_count += 1
      → 对比 snapshot_{scroll_count} 和 snapshot_{scroll_count-1}：
          有新元素 → same_count = 0
          完全相同 → same_count += 1

      // 常规退出条件（必须同时满足）：
          ① scroll_count >= 3（最少 3 次，硬性下限）
          ② same_count >= 2（连续 2 次无变化）

      // ⚠️ 无限滚动检测（新增）
      → 分析当前元素列表的结构特征：
        a) 是否为重复结构的列表项？（如帖子、新闻、股票行）
        b) 每次滚动后是否持续出现新的同类型元素？
        c) 是否有 "加载更多" / loading spinner / "pull to refresh" 等无限列表特征？
      → 如果连续 3 次滚动都出现新的同类型列表项（无限滚动）：
        1. 标注 [无限滚动列表]
        2. 记录列表项结构特征 + 已看到条目数量
        3. **立即停止滚动**
        4. 备注："无限加载列表，已采样 {N} 条，继续滚动不会产生新功能发现"

      // 安全上限：scroll_count >= 15 强制退出
      // 接力阈值：scroll_count >= 10 且不是无限滚动 且仍有新内容 → 触发接力机制
      → 否则继续 LOOP

STEP C: 底部快照
  → mobile_save_screenshot（{page}-scroll-{scroll_count}-bottom.png）

STEP D: 滚动摘要（必须写入报告）
  → "页面: {name} | 滚动: {scroll_count}次 | 首屏: {N0}个元素 | "
  → "末屏: {Nf}个元素 | 新增: {total_new} | 到底: {是/否} | 底部: {标志}"
```

**强制规则**：
1. **最少滚动 3 次**，即使第 1 次滚动后看起来到底了。很多 App 的内容在第 2-3 屏才开始出现（如设置页面的高级选项、列表的懒加载内容）
2. **"到底"判定 = 连续 2 次 list_elements 完全相同**。"看起来差不多"不算。
3. **滚动摘要是必须的产出物**，不是可选的。没有摘要 = 没执行滚动 = 探索不合格。
4. **异常标注**：如果首屏元素 > 10 但只滚了 3 次就到底 → 标 `[需复查]`
5. **无限列表处理**：如果检测到无限滚动（连续 3 次出现新的同类型列表项），立即停止并标注 `[无限滚动列表]`，记录结构特征和已采样条目数
6. **接力机制**：非无限滚动页面，滚动 10 次仍有新内容时触发接力（见 Step 5.2.2），最深可达 45 次滚动

**为什么是 3 次最低？**
实测中发现的常见漏判场景：
- 设置页面：第一屏只有 5-6 项，滚动 1 次后出现 10+ 项高级设置
- 详情页面：第一屏是概要，第 2-3 屏才是完整数据
- 列表页面：首屏加载 5 条，滚动触发懒加载后出现 20+ 条
- 底部浮动栏：遮挡了最后 2-3 个元素，需要滚动才能看到

### 2. 点击所有入口（Click-Every-Entry Protocol）

**每个可点击的元素都必须点击进入查看**。包括但不限于：

| 必须点击的元素 | 说明 |
|---------------|------|
| 所有按钮 | 包括图标按钮、文字按钮、浮动按钮 |
| 所有列表项/卡片 | 每个可点击的列表条目或卡片 |
| 所有 Tab/标签 | 顶部 Tab、底部 Tab、筛选标签 |
| 所有链接文字 | 如 "查看更多"、"了解详情"、">" 箭头 |
| 所有下拉菜单 | 点击展开查看所有选项 |
| 所有开关/选项 | 查看有哪些可配置项 |
| 设置页面的每一项 | 逐一点击进入查看子页面 |
| 弹窗/底部弹窗内的选项 | 弹窗出现后记录所有选项 |

### 3. 子页面递归探索（每层同等深度）

**核心原则：不论第几层，每个新页面都执行同样的探索标准。**

一个功能不会因为"藏得深"就不重要——相反，越深的页面越可能包含用户真正需要的核心操作（如设置项的子页面、期权策略的详情页等）。

```
探索深度规则（修订版）：

每一层的新页面都执行相同标准：
✅ 滚动扫描协议（至少 3 次滚动）
✅ 点击每个入口/按钮
✅ 记录功能描述
✅ 截图（顶部 + 底部）

唯一的退出条件不是"层数"，而是：
1. 页面已在"已探索页面清单"中 → 跳过（去重）
2. 页面需要认证/付费无法进入 → 记录到"未探索入口清单"
3. 页面内容与已探索页面完全相同（如同一个股票详情页从不同入口进入）→ 跳过

对于 Tab 类页面（如 MSFT 详情页的 Chart/Options/Fundamentals/Comments/News/Analysis）：
→ 每个 Tab 都是独立页面，必须各自执行完整滚动扫描
```

### 4. Subagent 探索指令模板（增强版）

派发 subagent 时，必须包含以下指令：

```
## 探索深度要求（必须严格遵守）

你必须做到以下所有步骤：

### A. 滚动扫描（每个页面强制执行）
- 每个页面至少 swipe up 3 次，即使第 1 次后看起来到底了
- 每次滚动后必须 list_elements_on_screen，对比与上一次的差异
- 退出条件：滚动 >= 3 次 且 连续 2 次 list_elements 完全相同
- 安全上限：15 次滚动强制退出
- 必须产出滚动摘要：页面名 + 滚动次数 + 首屏元素数 + 末屏元素数 + 新增元素数

### B. 点击穷尽
- 页面上的每一个按钮、入口、链接、Tab、卡片都要点击进入
- 进入子页面后同样要滚动到底 + 点击所有入口
- 特别注意：
  - 右上角的设置/菜单图标
  - "查看更多" / ">" 箭头
  - 下拉选择器（点击展开看所有选项）
  - 底部 Tab 的每个子标签
  - 列表的第一个和最后一个元素（确认内容结构）

### C. 记录要求
- 每个功能页面至少截图 2 张（scroll-0-top + scroll-N-bottom）
- 记录每个页面的完整元素清单（首屏 + 滚动后所有新增）
- 特别记录页面滚动后才能看到的内容（很多重要功能在首屏下方）

### D. 探索完成标准（可验证）
一个页面的探索只有在满足以下所有条件时才算完成：
✅ 有滚动摘要（滚动次数 >= 3，有首屏/末屏元素数）
✅ 有顶部截图和底部截图（至少 2 张）
✅ 已点击并进入了页面上的所有入口/链接/按钮
✅ 已对每个子页面重复以上步骤

### E. 报告末尾必须附【滚动覆盖汇总表】+ 【未探索入口清单】

| 页面 | 滚动次数 | 首屏元素 | 末屏元素 | 新增元素 | 到底 | 性能感知 | 备注 |
|------|---------|---------|---------|---------|------|---------|------|
| {页面名} | {N} | {N0} | {Nf} | {new} | 是/否 | {快/正常/慢} | |

### F. 【未探索入口清单】（必须，即使为空也要写"无"）

| 入口名称 | 发现位置 | 未探索原因 | 建议处理 |
|---------|---------|-----------|---------|
| {名称} | {位置} | {原因} | 需新subagent/标记受限 |
| 无 | - | - | - |

没有滚动覆盖汇总表或未探索入口清单 = 探索不合格。主 agent 会拒收并要求重做。

### G. 报告必须写入文件

Subagent 的完整报告必须写入 `{BASE_DIR}/exploration-reports/subagent-{id}-{page}.md`。
返回给主 agent 的只是不超过 20 行的摘要。这保护主 agent 的 context window。
```

---

## 附录 A：领域专家原型库

> **设计哲学**：Persona 的价值不在数量，而在"视角差异化"。
> 一个懂期权的 persona 发现"缺少 IV Rank"，比十个通用 persona 说"功能还不错"有价值一万倍。
> 领域专家原型库为每个产品类别预定义了 6 个深度差异化的专家视角，
> 每个原型自带**领域知识注入段**和**领域专项测试清单**，确保 persona 能给出专业级反馈。

### A.0 反馈深度标准（通用基准）

在给出领域原型之前，先明确什么是"深度反馈"。以下 8 组对比适用于所有产品类别：

| # | 浅层反馈（❌ 不合格） | 深层反馈（✅ 合格） |
|---|---------------------|-------------------|
| 1 | "期权功能比较完善" | "期权链支持按 Expiry Date 筛选但缺少 Delta-based strike 筛选（如只看 0.30 delta 附近的合约）；Greeks 只显示 Delta 和 Theta，缺少 Gamma/Vega/Rho，这对做 volatility play 的交易者是硬伤；没有 IV Rank/IV Percentile 指标，无法快速判断当前 IV 水平是否偏高" |
| 2 | "图表功能不错" | "K 线图支持 1m/5m/15m/1h/1D/1W/1M 共 7 个周期，响应速度约 0.8s，但从日线切到分钟线需要重新加载整个图表（Webull 是无缝切换）。技术指标有 45 种（对标 TradingView 的 100+），缺少 Ichimoku Cloud 和 VWAP。指标参数可自定义但没有保存预设功能" |
| 3 | "搜索功能好用" | "输入 'AAPL' 联想词 0.3s 出结果，准确匹配股票+ETF+期权。但搜 '高分红 ETF' 这类语义查询返回空结果，说明只支持 ticker/名称精确匹配，不支持自然语言搜索。另外搜索历史不支持删除单条，只有清空全部" |
| 4 | "社区内容丰富" | "社区首页瀑布流以股评和持仓晒单为主（约 70%），技术分析深帖少（<10%）。帖子排序只有'最新'和'最热'，缺少'精华'和'我关注的话题'过滤。评论区无楼中楼（嵌套回复），长讨论串阅读体验差。比 Stocktwits 内容密度低但比 Webull 社区质量高" |
| 5 | "开户流程简单" | "开户 5 步（身份→地址→就业→投资经验→协议），总耗时约 4 分钟。第 3 步'年收入'下拉选项只有 5 档（<25K/25-50K/50-100K/100-200K/>200K），粒度偏粗。SSN 输入框没有格式提示（XXX-XX-XXXX），直到提交才报格式错误。好的方面：支持拍照自动识别驾照信息，省去手动输入地址" |
| 6 | "加载速度还行" | "首页冷启动 2.1s（WiFi 环境），热启动 0.6s。Watchlist 30 只股票的实时行情刷新间隔约 1s。MSFT 详情页首次加载 1.8s，切换 Tab（如从 Chart 到 Options）约 0.9s。期权链展开（显示 50+ strikes）约 1.5s，有明显的加载占位符动画" |
| 7 | "UI 设计好看" | "深色模式为主，强调色用橙色（买入/涨）和蓝色（卖出），与传统红绿不同，需要适应。字体层级清晰（价格用 24sp 粗体、变动幅度用 14sp 彩色、成交量用 12sp 灰色）。但信息密度偏低——Watchlist 每行只显示价格和涨跌幅，缺少成交量和市值列（Webull 的 Watchlist 支持自定义列）" |
| 8 | "新手引导不错" | "首次打开有 4 页 onboarding（产品介绍→核心功能→模拟交易→开户引导），可跳过。进入 Trade 页面首次点击'买入'时有 tooltip 指引订单类型选择。但模拟交易入口藏在'我的→模拟交易'三级页面，新手很难发现。术语解释只覆盖了基础词汇（如'限价单'），进阶概念（如'止损限价单'触发条件）无解释" |

**深度标准量化指标**：
- 每条反馈至少包含 1 个**量化数据**（时间、数量、百分比、步骤数）
- 每条反馈至少包含 1 个**具体对比**（与竞品、与预期、与行业标准）
- 每条反馈至少包含 1 个**具体场景**（什么情况下会遇到这个问题/优势）

---

### A.1 金融/投资 App 领域专家原型（完整示例）

#### 原型 1：期权交易者（Options Trader）

**领域知识注入**（复制到 persona 档案的"领域知识注入"字段）：

> 你熟悉 Black-Scholes 定价模型的直觉含义，理解 Greeks 全家族：Delta（方向暴露）、
> Gamma（Delta 变化率/凸性风险）、Theta（时间衰减）、Vega（波动率暴露）、Rho（利率敏感度）。
> 你关注 IV Rank 和 IV Percentile 来判断当前隐含波动率的相对水平。
> 你经常使用的策略包括：Vertical Spread（Bull Call / Bear Put）、Iron Condor、
> Straddle/Strangle、Covered Call、Cash-Secured Put、Calendar Spread。
> 你习惯用 ThinkorSwim 或 Tastyworks 做期权分析，对期权链的展示效率和策略构建器
> 有很高要求。你知道期权流动性通过 Bid-Ask Spread 和 Open Interest 判断，
> 你会避开 spread > 10% mid price 的合约。你理解 DTE（Days to Expiration）、
> Assignment Risk、Early Exercise、Pin Risk 等概念。

**领域专项测试清单**（复制到 persona 档案的"领域专项测试清单"字段）：

1. 打开 AAPL 期权链 → 选 30DTE 的 expiry → 检查是否显示 Delta/Gamma/Theta/Vega 全部四个 Greeks
2. 检查期权链是否支持按 Delta 范围筛选 strike（如只看 0.25-0.35 delta 的合约）
3. 查看是否有 IV Rank / IV Percentile 指标，是否有 IV 历史图表
4. 尝试构建 Bull Call Spread：选 long call + short call → 是否有策略构建器自动计算盈亏图
5. 检查期权链加载速度（选高流动性标的如 SPY，合约数通常 > 500）
6. 查看单个期权合约的详情页 → 是否显示 Open Interest、Volume、Bid-Ask Spread、Last Trade
7. 检查是否支持期权 P&L 图（payoff diagram）和 what-if 分析（调整 IV/时间的影响）
8. 尝试下期权单 → 检查是否支持 limit/market 以外的期权订单类型（如 spread order/combo）
9. 查看持仓中的期权 → 是否显示到期日倒计时、当前 Greeks、盈亏百分比
10. 检查是否有 Options Scanner / Screener（按策略类型、IV 范围、DTE 范围筛选机会）
11. 检查是否支持 multi-leg 策略一键下单（如 Iron Condor 四腿同时提交）
12. 查看期权的 Time & Sales 数据是否可用

**专业反馈模板**（作为 persona 反馈的深度标杆）：
> "期权链的 Greeks 展示只有 Delta（0.45）和 Theta（-0.03），缺少 Gamma 和 Vega。
> 这意味着我无法直接评估 Gamma 风险（ATM 短 DTE 期权的 Gamma 暴露很大），
> 也无法判断 Vega 暴露对 IV 变化的敏感度。对于做 Iron Condor 这类 vega-neutral 策略的人来说，
> 没有 Vega 显示基本就是盲操作。建议至少显示完整四希腊字母，理想情况下支持按 Greeks 排序。
> 对标：ThinkorSwim 默认显示全部 Greeks + 可自定义列，Tastyworks 甚至显示 IV Rank 在期权链头部。"

---

#### 原型 2：基本面 / 价值投资者（Fundamental / Value Investor）

**领域知识注入**：

> 你信奉 Ben Graham 和 Warren Buffett 的价值投资理念，关注企业内在价值。
> 你定期阅读 10-K（年报）和 10-Q（季报），重点关注 Income Statement、Balance Sheet、
> Cash Flow Statement 三表。你使用的核心估值指标包括：P/E（TTM 和 Forward）、P/B、P/S、
> EV/EBITDA、PEG Ratio、Dividend Yield、Payout Ratio、Free Cash Flow Yield。
> 你会做 DCF（现金流折现）估值，需要历史 FCF 数据和增长率假设。
> 你关注 ROE、ROA、ROIC、Debt/Equity、Current Ratio、Interest Coverage 等质量指标。
> 你习惯用 Morningstar、Seeking Alpha 或 GuruFocus 做基本面研究，
> 对财务数据的完整性、时效性和可比性有高要求。你会做同行业横向对比（peer comparison）。

**领域专项测试清单**：

1. 打开 MSFT 详情页 → 找到 Fundamentals/财务数据 tab → 检查是否有完整三表（损益/资产负债/现金流）
2. 检查财务数据是否覆盖至少 5 年历史（做趋势分析用）
3. 查看估值指标 → 是否有 P/E (TTM)、Forward P/E、P/B、P/S、EV/EBITDA、PEG
4. 检查是否有 DCF 计算器或内在价值估算工具
5. 查看是否支持同行业对比（peer comparison）：选 MSFT → 对比 AAPL/GOOG/AMZN 的关键指标
6. 检查分红信息 → 是否有 Dividend History、Yield、Payout Ratio、Ex-Dividend Date、增长率
7. 查看管理层信息 → 是否有高管简介、薪酬、内部人交易（insider trading）记录
8. 检查是否有 SEC Filings 直链（10-K/10-Q/8-K）或 Earnings Transcript
9. 查看 Earnings Calendar → 是否显示预期 EPS vs 实际 EPS、历史 beat/miss 记录
10. 检查财务数据的时效性：最新季报是否已更新
11. 尝试 Screener → 是否支持按基本面指标筛选（P/E < 15, ROE > 15%, Dividend Yield > 3%）
12. 检查是否有分析师评级汇总（Analyst Ratings）和目标价

---

#### 原型 3：日内交易 / 技术分析者（Day Trader / Technical Analyst）

**领域知识注入**：

> 你每天交易 5-20 次，持仓时间从几秒到几小时不等。你依赖技术分析做决策，
> 核心工具是 K 线图 + 技术指标 + Level 2 行情。你常用的指标包括：
> VWAP（日内基准）、EMA 9/20/50/200、MACD、RSI、Bollinger Bands、Volume Profile。
> 你需要 Level 2（订单簿深度）来判断买卖压力，需要 Time & Sales（逐笔成交）来读 tape。
> 你用 hotkeys 快速下单（一键买/卖/平仓），对执行速度和 chart 响应极其敏感。
> 你在 TradingView 上做图表分析，在 DAS Trader 或 Lightspeed 上执行交易。
> 你理解 PDT 规则（Pattern Day Trader，25K 最低资金要求），关注 buying power 和 margin。
> 图表上你会画 Support/Resistance、Trendline、Fibonacci Retracement。

**领域专项测试清单**：

1. 图表切换到 1 分钟周期 → 检查实时更新频率（是否逐 tick 刷新）和延迟
2. 检查技术指标列表 → 是否有 VWAP、EMA、MACD、RSI、Bollinger Bands、Volume Profile
3. 查看是否支持多指标叠加（如同时显示 VWAP + EMA 20 + RSI）
4. 检查是否有 Level 2 / Order Book Depth 数据（买卖各多少档）
5. 检查是否有 Time & Sales（逐笔成交记录）
6. 测试下单速度 → 从点击"买入"到订单确认需要几步/几秒
7. 检查是否支持快捷下单（hotkeys / 一键下单 / 滑动下单）
8. 图表上画线工具 → 是否支持趋势线、水平线、Fibonacci Retracement
9. 检查图表在快速缩放/拖动时的流畅度（有无卡顿/重绘）
10. 查看是否有 Pre-market / After-hours 行情数据和图表
11. 检查订单类型 → 是否支持 Trailing Stop、OCO（One-Cancels-Other）、Bracket Order
12. 查看是否有 Buying Power / Margin 信息和 PDT 状态提示
13. 测试 Alert（价格提醒）设置 → 是否支持条件提醒（如 AAPL > $200 或 RSI > 70）

---

#### 原型 4：宏观 / ETF 长线投资者（Macro / ETF Long-term Investor）

**领域知识注入**：

> 你关注宏观经济周期，通过 ETF 做大类资产配置（股票/债券/商品/REITs）。
> 你跟踪的宏观指标包括：GDP 增长率、CPI/PCE（通胀）、联邦基金利率、失业率、
> PMI（制造业/服务业）、收益率曲线（2Y-10Y spread）、美元指数。
> 你用 ETF 构建投资组合，关注 Expense Ratio、Tracking Error、AUM、流动性。
> 你了解 Sector Rotation（板块轮动）和 Factor Investing（价值/成长/动量/质量/低波动）。
> 你会用 Correlation Matrix 分析资产间相关性来优化分散化。
> 你常用的 ETF 分析工具包括 ETF.com 和 Portfolio Visualizer。
> 你定期做 Portfolio Rebalancing，关注资产配置偏移。

**领域专项测试清单**：

1. 查看是否有宏观经济数据面板（GDP/CPI/利率/失业率等）
2. 检查 ETF 筛选器 → 是否支持按 Expense Ratio、AUM、Yield、Category 筛选
3. 打开 SPY 详情 → 检查是否有 ETF 专属信息（Holdings Top 10、Sector Breakdown、Expense Ratio）
4. 检查是否有 ETF 对比工具（如 SPY vs VOO vs IVV 并排比较）
5. 查看是否有资产配置建议或投资组合分析工具
6. 检查是否有 Sector Performance 热力图或板块轮动数据
7. 查看是否有经济日历（Economic Calendar）标注重要数据发布时间
8. 检查是否有 Correlation Matrix 或 Portfolio Risk 分析
9. 查看债券/固收 ETF 信息 → 是否显示 Duration、Yield to Maturity、Credit Quality
10. 检查是否支持创建 Watchlist 按自定义分组（如"美股 ETF"/"债券"/"商品"）
11. 查看是否有 Model Portfolio / Lazy Portfolio 参考方案

---

#### 原型 5：长期定投 / 被动投资者（DCA / Passive Investor）

**领域知识注入**：

> 你信奉指数投资和 Dollar Cost Averaging（定期定额投资），目标是长期财富积累。
> 你每月固定投入 $500-2000，分散在 3-5 只核心 ETF（如 VTI/VXUS/BND）。
> 你关注：定投计划设置的灵活性、DRIP（分红再投资）是否自动、费率是否透明（交易佣金+管理费）。
> 你用的指标简单但关注长期：Total Return、CAGR、Maximum Drawdown、Sharpe Ratio。
> 你不频繁交易，但需要：组合再平衡提醒、税务优化（Tax-Loss Harvesting）提示、
> 定投执行记录和 DCA 统计（平均成本/投入总额/当前价值）。
> 你之前用 Wealthsimple 或 M1 Finance 这类自动化投资平台。

**领域专项测试清单**：

1. 检查是否有自动定投（Auto-Invest / Recurring Buy）功能入口
2. 定投设置 → 是否支持自定义周期（每周/双周/每月）、金额、标的组合
3. 查看是否支持 Fractional Shares（碎股买入），这是定投的前提
4. 检查 DRIP（分红再投资）→ 是否支持自动 DRIP，在哪里设置
5. 查看费率页面 → 交易佣金、平台费、货币兑换费是否明确列出
6. 检查投资组合视图 → 是否有 Total Return、Unrealized P&L、Cost Basis 汇总
7. 查看是否有组合再平衡（Rebalancing）功能或提醒
8. 检查是否有 DCA 统计视图（显示每次买入记录、平均成本、当前收益）
9. 查看税务相关信息 → 是否有 Tax-Loss Harvesting 提示或已实现损益报表
10. 检查是否有投资目标设定（如退休计划、教育金）和进度追踪
11. 查看历史交易记录 → 是否可导出 CSV/PDF 用于报税

---

#### 原型 6：投资新手 / 小白用户（Complete Beginner）

**领域知识注入**：

> 你对投资几乎没有概念，不知道"股票"和"ETF"的区别，不理解"限价单"和"市价单"。
> 你被朋友推荐来开户，目标是"把钱放进去让它增值"。你害怕亏钱，对风险极度敏感。
> 你不懂任何技术术语：什么是 P/E？什么是分红？什么是做空？全都不知道。
> 你需要的是：每一步都有清晰的解释，专业术语要有白话文翻译，
> 最好有模拟交易让你先练手不亏真钱。你最担心的是"我会不会一不小心亏光"。
> 你对 App 的期望是"像支付宝余额宝一样简单"。

**领域专项测试清单**：

1. 首次打开 App → 是否有新手引导？引导是否用白话文而非术语？
2. 查看是否有模拟交易（Paper Trading）入口 → 入口是否明显？
3. 点击"买入"按钮 → 订单类型选择是否有解释？"限价"和"市价"有没有说明？
4. 查看是否有术语词典 / 新手学堂 / 投资百科入口
5. 搜索"怎么买股票"或点击帮助 → 是否有新手教程？
6. 检查风险提示 → 在买入前是否有风险告知？内容是否能被新手理解？
7. 查看首页信息 → 是否能看懂每个数字的含义？红绿色的含义有解释吗？
8. 尝试理解一只股票的详情页 → 页面上有多少你不认识的术语？
9. 检查是否有"一键投资"或"推荐组合"这类降低决策门槛的功能
10. 查看客服/帮助入口 → 是否容易找到？是否有在线客服？

---

### A.2 社交/内容 App 领域专家原型（框架）

> 以下为框架模板，实际使用时根据具体产品（如小红书、抖音、微博）定制测试清单。

| # | 原型名称 | 核心关注点 | 测试清单重点方向 |
|---|---------|-----------|----------------|
| 1 | **短视频创作者** | 拍摄工具、剪辑功能、滤镜/特效、BGM 库、封面编辑、数据分析（播放量/完播率/粉丝画像） | 发布流程效率、创作工具专业度、数据分析颗粒度 |
| 2 | **图文博主** | 图片编辑、排版工具、Markdown 支持、SEO/话题标签、合集功能 | 编辑器能力、多图排版、长文阅读体验 |
| 3 | **社区运营者** | 社群管理工具、置顶/加精、数据看板、举报处理、自动化规则 | 管理后台功能完整度、批量操作效率 |
| 4 | **隐私极客** | 隐私设置颗粒度、数据收集透明度、位置/通讯录权限、已读回执控制 | 隐私设置完备性、数据导出/删除能力 |
| 5 | **算法研究者** | 推荐逻辑透明度、内容分发机制、"不喜欢"反馈有效性、信息茧房程度 | 推荐多样性、冷启动体验、负反馈响应速度 |
| 6 | **纯消费用户** | 内容发现效率、阅读/观看体验、广告频率/侵入性、离线缓存 | 内容质量和密度、广告体验、夜间模式 |

**知识注入示例（短视频创作者）**：
> 你经营 3 个账号（美食/旅行/日常），全职做短视频已经 2 年，全平台粉丝 50 万+。
> 你对拍摄工具的要求：最少 1080p 60fps、光圈/快门可调、倒计时拍摄、分段拍摄。
> 剪辑要求：timeline 式剪辑、转场 20 种以上、关键帧动画、音频波形对齐、字幕自动生成。
> 数据分析要求：每条视频的播放曲线（而非只有总播放量）、完播率、平均观看时长、
> 粉丝增长归因（哪条视频带来了粉丝）、最佳发布时间分析。
> 你现在主用剪映+抖音，正在评估其他平台的创作者工具。

---

### A.3 电商/消费 App 领域专家原型（框架）

| # | 原型名称 | 核心关注点 | 测试清单重点方向 |
|---|---------|-----------|----------------|
| 1 | **供应链/选品买手** | 供应商信息、最小起订量、物流时效、品质认证、样品订购 | 商品信息完整度、供应商评价体系、批量采购流程 |
| 2 | **优惠券/返利专家** | 优惠券叠加规则、满减计算、价格历史、比价功能、返利追踪 | 促销系统逻辑清晰度、价格透明度 |
| 3 | **跨境购物者** | 关税预估、物流追踪、汇率计算、退货政策、本地化支付 | 跨境流程透明度、物流信息完整度、费用明细 |
| 4 | **品类垂直重度用户** | 商品参数对比、专业评测、尺码/兼容性匹配、真伪鉴别 | 商品详情专业度、参数对比工具、用户评价质量 |
| 5 | **售后维权达人** | 退换货流程、客服响应、投诉渠道、维权证据保存 | 售后流程效率、客服可达性、争议处理机制 |
| 6 | **首次网购用户** | 注册安全感、支付方式理解、物流状态查看、退货流程 | 新手引导、安全提示、流程清晰度 |

---

### A.4 工具/效率 App 领域专家原型（框架）

| # | 原型名称 | 核心关注点 | 测试清单重点方向 |
|---|---------|-----------|----------------|
| 1 | **自动化达人** | Workflow/Shortcut 支持、API 可用性、Zapier/IFTTT 集成、批量操作 | 自动化能力深度、第三方集成广度 |
| 2 | **键盘流极客** | 快捷键支持、命令面板（Cmd+K）、导航效率、自定义快捷键 | 键盘操作覆盖度、导航效率 |
| 3 | **团队管理者** | 权限系统、协作功能、审计日志、团队仪表盘、成员管理 | 多人协作场景、权限颗粒度 |
| 4 | **跨平台用户** | 多端同步、离线能力、数据导出/导入格式、迁移工具 | 跨平台一致性、数据可移植性 |
| 5 | **API/插件开发者** | API 文档质量、SDK、Webhook、插件市场、开发者工具 | 开发者生态成熟度 |
| 6 | **轻度新用户** | 上手难度、模板质量、默认设置合理性、免费功能够用度 | 开箱体验、学习曲线 |

---

### A.5 自定义领域专家原型（通用模板）

对于未在上述列表中覆盖的产品类别，按以下模板创建领域专家原型：

```markdown
#### 原型 N：{原型名称}（{英文名}）

**领域知识注入**（200-300 字）：
> 你的专业背景是 {领域}，有 {N} 年经验。
> 你掌握的核心知识/工具/概念包括：{列出 10-15 个该领域的关键术语和概念}。
> 你日常使用的同类工具/App 包括：{列出 2-3 个竞品及使用深度}。
> 你评价这类产品的核心标准是：{列出 3-5 个专业评价维度}。

**领域专项测试清单**（10-15 项）：
1. {可执行的具体操作 → 预期结果/检查点}
2. ...（每项都必须是"动作 → 验证"格式）

**专业反馈模板**：
> {一段示例反馈，展示该领域的专业深度标杆}
```

**创建领域专家原型的检查清单**：
- [ ] 领域知识注入包含至少 10 个专业术语/概念
- [ ] 测试清单至少 10 项，每项是可执行操作
- [ ] 反馈模板展示了"深层反馈"的标杆（参照 A.0 的深度标准）
- [ ] 知识注入精炼在 200-300 字内（不写教科书）
- [ ] 列出了 2-3 个竞品作为对标参照

---

## 注意事项

- 报告语言跟随用户习惯（默认中文）
- 每次体验按日期和版本独立存储，方便追踪产品迭代
- subagent 探索时如果遇到需要登录的功能，先尝试其他入口；如确实需要登录，通过 AskUserQuestion 询问用户是否手动输入；用户选择跳过时报告为"需要登录"并记录到未探索入口清单
- **敏感输入人机接力**：遇到交易密码、登录、银行账户绑定、KYC 等敏感信息输入时，agent 暂停并通过 AskUserQuestion 请求用户协助，绝不自行输入或猜测密码
- **数据准确性"所见即所录"**：报告中的所有数据必须来自 App 中实际看到的内容。禁止根据部分信息推断总数或全貌。遇到下拉菜单和折叠面板必须展开后截图记录完整列表
- **主 agent context 保护**：subagent 的完整报告写入 `exploration-reports/` 目录下的独立文件，返回给主 agent 的只是不超过 20 行的摘要。报告生成 subagent 直接从文件读取探索数据
- 如果 App 崩溃或无响应，记录问题并尝试重新启动
- 探索时间较长的产品，定期向用户汇报进度
- **所有产品数据必须来自 App 内或官方渠道，严禁使用第三方网站数据**
- **每个页面必须滚动到底，点击每个入口，不要只看第一屏**

---

## PlayCover Setup 子命令

用于在 Mac 上通过 PlayCover 快速配置 iOS app 以 iPhone 模式运行，为产品体验做好环境准备。

**触发**: `/product-review playcover-setup [App名称或bundleID]`

### 执行流程

1. **查找 app**: 在 `~/Library/Containers/io.playcover.PlayCover/Applications/` 下查找匹配的 `.app` bundle
2. **读取当前配置**: 读取 `~/Library/Containers/io.playcover.PlayCover/App Settings/<bundleID>.plist`
3. **应用 iPhone 模式配置**:
   ```bash
   plutil -replace resolution -integer 0 "<App Settings plist path>"
   plutil -replace customScaler -float 3 "<App Settings plist path>"
   ```
4. **重启 PlayCover + App**:
   ```bash
   pkill -f "<app process name>" 2>/dev/null || true; sleep 1
   pkill -f "PlayCover" 2>/dev/null || true; sleep 3
   open -g "<app .app path>"
   ```
5. **验证**: 等待 app 加载后截图确认 iPhone UI 渲染正确

### 配置说明

| 参数 | 值 | 作用 |
|---|---|---|
| `resolution` | `0` | 关闭自适应显示，app 使用实际窗口尺寸渲染，避免渲染宽度与窗口不匹配 |
| `customScaler` | `3` | 3x Retina 渲染，接近真实 iPhone 像素密度，解决模糊问题 |

### 原理

PlayTools（PlayCover 注入框架）在 `resolution≥1` 时，向 app 报告的屏幕宽度与 macOS 实际窗口宽度存在固定 ~23% 缩放差，导致 app 的渲染视口超出窗口边界、UI 被截断。`resolution=0` 绕过此问题，让 app 直接使用实际窗口尺寸。

### 已知限制

- 窗口固定为 ~288×541pt（在 4K 显示器上约等于真实 iPhone 物理尺寸）
- 弹窗位置可能略有偏移（坐标映射限制）
- 窗口大小不可通过 windowWidth/Height 调整（resolution=0 会覆盖）
- 如需彻底解决，需升级 PlayTools 到支持 resolution=6（resizable）的版本

---

## Android Setup 子命令

快速设置 Android 模拟器环境，为产品体验做好准备。

**触发**: `/product-review android-setup [App名称或APK路径]`

### 执行流程

1. **检查环境**:
   ```bash
   # Android SDK
   echo $ANDROID_HOME  # 通常 ~/Library/Android/sdk/
   $ANDROID_HOME/emulator/emulator -version
   adb version
   ```
   - 如未安装，指导用户安装 Android Studio 或通过 `sdkmanager` 安装命令行工具
2. **查找/创建 AVD**:
   - 列出现有: `$ANDROID_HOME/emulator/emulator -list-avds`
   - 如无合适 AVD:
     ```bash
     sdkmanager "system-images;android-34;google_apis;arm64-v8a"
     avdmanager create avd -n product_review_phone -k "system-images;android-34;google_apis;arm64-v8a" -d "pixel_7"
     ```
3. **启动模拟器**:
   ```bash
   $ANDROID_HOME/emulator/emulator -avd <avd_name> -no-snapshot-load &
   adb wait-for-device
   ```
4. **安装 App**: 如提供 APK → `adb install <path.apk>`；否则提示用户提供
5. **验证**: `mobile_list_available_devices` 确认模拟器可用，启动 app 截图确认

### 推荐配置

| 项 | 值 | 说明 |
|---|---|---|
| 设备 | Pixel 7 | 标准尺寸 |
| 系统 | Android 14 (API 34) | 主流版本 |
| 架构 | arm64-v8a | Apple Silicon Mac 必须用 ARM |

### 常见问题

- **模拟器卡住**: `emulator -avd <name> -no-snapshot-load -gpu swiftshader_indirect`
- **adb 连不上**: `adb kill-server && adb start-server`
- **Apple Silicon**: 必须用 arm64-v8a 镜像，不能用 x86_64

---

## iOS Simulator Setup 子命令

快速设置 iOS 模拟器环境，为产品体验做好准备。

**触发**: `/product-review ios-sim-setup [App名称]`

### 执行流程

1. **检查环境**:
   ```bash
   xcode-select -p
   xcrun simctl list devices available
   xcrun simctl list runtimes
   ```
2. **查找/创建模拟器**:
   - 列出可用: `xcrun simctl list devices available`
   - 创建新模拟器:
     ```bash
     xcrun simctl list devicetypes | grep iPhone
     xcrun simctl list runtimes | grep iOS
     xcrun simctl create "ProductReview-iPhone15Pro" "com.apple.CoreSimulator.SimDeviceType.iPhone-15-Pro" "com.apple.CoreSimulator.SimRuntime.iOS-18-2"
     ```
3. **启动模拟器**:
   ```bash
   xcrun simctl boot <device_udid>
   open -a Simulator
   ```
4. **安装 App**:
   - .app/.zip → `xcrun simctl install booted <path>`
   - .ipa 不能装模拟器（真机包），需要 .app 的模拟器版本
   - App Store 下载: 提示用户手动在模拟器中操作
5. **验证**: `mobile_list_available_devices` 确认，启动 app 截图

### 推荐配置

| 项 | 值 |
|---|---|
| 设备 | iPhone 15 Pro |
| iOS | 最新可用（`xcrun simctl list runtimes`） |

### 常见问题

- **无运行时**: Xcode → Settings → Platforms 下载
- **.ipa 装不了**: 模拟器需要 .app（模拟器架构），.ipa 是真机包
- **App 闪退**: 部分 app 不支持模拟器架构

---

## iOS Device Setup 子命令

连接 iOS 真机进行产品体验测试。

**触发**: `/product-review ios-device-setup [设备名]`

### 执行流程

1. **检查连接**:
   ```bash
   xcrun devicectl list devices
   # 或
   system_profiler SPUSBDataType | grep -A5 iPhone
   ```
   - 未出现 → 提示: 解锁设备、点击"信任此电脑"、换 USB 线/端口
2. **开发者模式**（iOS 16+）:
   - 设置 → 隐私与安全性 → 开发者模式 → 开启 → 重启
   - 如选项不可见: 先用 Xcode 连接设备一次
3. **配对**:
   ```bash
   xcrun devicectl manage pair --device <device_udid>
   ```
4. **安装 App**（如需）:
   ```bash
   xcrun devicectl device install app --device <device_udid> <path.ipa>
   ```
5. **验证**:
   - `mobile_list_available_devices` 确认设备出现
   - `mobile_take_screenshot` 验证 mobile-mcp 可控制
6. **输出设备信息摘要**

### Wi-Fi 调试（可选）

```bash
# USB 连接后启用
xcrun devicectl manage enableNetworkDebugging --device <device_udid>
```

### 常见问题

- **不被识别**: 换端口/线，确保设备已解锁
- **"信任"不弹出**: 还原位置与隐私设置（设置 → 通用 → 传输或还原 → 还原）
- **mobile-mcp 看不到**: 确认已解锁、已信任、开发者模式已开
- **安装失败**: 检查签名是否有效（Ad Hoc / TestFlight / 企业证书）
