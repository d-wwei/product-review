# Step 0-4: 环境检查、设备选择、App 确认、模式选择、工作目录

> 本文件由 SKILL.md 按需加载。主 agent 在初始化阶段 Read 此文件。

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
- **设备类型**（simulator / emulator / device）

**多设备检测**：
```
如果返回的可用设备 > 1 台：
  记录 DEVICES = [device_1, device_2, ...]
  MULTI_DEVICE = true
  用 AskUserQuestion 询问：
    "检测到 {N} 台可用设备: {列表}。是否启用多设备并行探索？
     并行模式可将探索时间缩短约 {N}x。"
  如果用户同意 → PARALLEL_MODE = true
  否则 → 使用用户选择的单台设备
```

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

## Step 2.5: 触控能力验证（iOS 模拟器必须执行）

> **为什么需要这一步**：iOS 模拟器存在已知的触控映射 Bug——在可滚动列表区域点击 cell 会被
> 误识别为 swipe-to-home 手势，导致 App 退出到桌面。此 Bug 会阻断 40%+ 的探索覆盖。
> 真机和 Android 模拟器通常无此问题，但仍建议快速验证。

**触发条件**：App 启动成功（Step 2 完成）后立即执行。

### 设备类型快速路径

```
如果设备类型 = "device"（iOS 真机）:
  → 跳过触控验证，真机无此映射问题
  → 输出: "真机模式 ✅ — 跳过触控验证"
  → 直接进入 Step 3

如果设备类型 = "emulator"（Android 模拟器）:
  → 执行轻量验证（仅测 2 个点：屏幕中心 + 列表区域）
  → 如果通过 → 跳过完整测试
  → 如果失败 → 执行完整 5 点测试

如果设备类型 = "simulator"（iOS 模拟器）:
  → 执行完整 5 点触控测试（见下方）
```

### 验证流程（iOS 模拟器完整版）

```
TOUCH_BUG_DETECTED = false

# 1. 获取屏幕尺寸和当前元素
screen = mobile_get_screen_size(device)
elements = mobile_list_elements_on_screen(device)
mobile_save_screenshot(device, "{screenshots}/00-touch-test-before.png")

# 2. 执行 5 点触控测试
test_points = [
  ("屏幕中心",     screen.width/2,  screen.height/2),
  ("顶部导航栏",    screen.width/2,  screen.height * 0.08),
  ("底部 Tab 栏",   screen.width/2,  screen.height * 0.95),
  ("列表区域上部",   screen.width/2,  screen.height * 0.35),
  ("列表区域下部",   screen.width/2,  screen.height * 0.65),
]

for (name, x, y) in test_points:
    mobile_click_on_screen_at_coordinates(device, x, y)
    sleep(0.5)
    mobile_take_screenshot(device)  # 检查是否仍在 App 内

    # 判断 App 是否退出到桌面
    current_elements = mobile_list_elements_on_screen(device)
    if 看到桌面元素（如 App 图标网格、Dock）而非 App 内容:
        TOUCH_BUG_DETECTED = true
        bug_zone = name
        # 重新启动 App
        mobile_launch_app(device, packageName)
        sleep(2)
        break
```

### 检测结果处理

**如果 `TOUCH_BUG_DETECTED = false`**：
```
输出: "触控验证通过 ✅ — 5 个测试点均正常响应，继续正常流程。"
继续 Step 3
```

**如果 `TOUCH_BUG_DETECTED = true`**：
```
输出:
"⚠️ 检测到 iOS 模拟器触控映射 Bug
 - 故障区域: {bug_zone}
 - 现象: 点击列表/可滚动区域时 App 退出到桌面
 - 影响: 约 40% 的页面（股票详情页、搜索结果、持仓列表等）无法通过直接点击进入"

用 AskUserQuestion 提供选项:
  A. 切换到 iOS 真机（推荐）
     → 执行 ios-device-setup 流程，然后重新开始
  B. 尝试 xcrun simctl 注入触控事件
     → 测试: xcrun simctl io <device_id> sendSimulatedTouch <x> <y>
     → 如果可行，后续所有列表区域点击改用 simctl 注入
  C. 继续使用模拟器（降级模式）
     → 记录 TOUCH_BUG_DETECTED=true
     → 探索策略自动适配（见 Step 5 触控降级协议）
     → 预估覆盖损失: 30-40%
  D. 降低模拟器分辨率后重试
     → xcrun simctl shutdown <device_id>
     → 创建低分辨率模拟器并重试触控测试
```

### 降级模式标志

如果用户选择 C（继续降级），设置以下标志供后续步骤使用：

```
TOUCH_BUG_DETECTED = true
RESTRICTED_ZONES = ["scrollable_list_cells", "search_result_items"]
ALTERNATIVE_NAV_STRATEGIES = ["search_navigation", "deep_link", "manual_handoff"]
```

这些标志会在 Step 5 的 subagent prompt 中注入，指导 subagent 使用替代导航路径。

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
