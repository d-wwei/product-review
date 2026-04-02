# 外部 Skill 集成（可选能力扩展）

> 本文件由 SKILL.md Step 0 加载。检测外部能力并注册可用工具集。
> product-review 不硬依赖任何外部 skill——它们是能力增强，不是前置条件。

---

## 能力探测

Step 0 环境检查时，在 mobile-mcp 检查之后执行以下探测：

### 1. macOS Desktop Control（MCP Server）

**检测方法**：

```bash
# 检查 MCP server 是否已注册
if grep -q "macos-desktop-control" ~/.claude/settings.json 2>/dev/null || \
   grep -q "macos-desktop-control" ~/.claude/settings.local.json 2>/dev/null; then
  echo "DESKTOP_CONTROL=configured"
else
  echo "DESKTOP_CONTROL=not_found"
fi
```

**如果 `DESKTOP_CONTROL=configured`**：

记录能力标志 `HAS_DESKTOP_CONTROL=true`，后续步骤可使用以下工具：

| 能力 | 工具 | 用途 |
|------|------|------|
| 桌面截图 | `mcp__macos-desktop-control__screenshot` | 截取桌面 App / PlayCover App / 模拟器窗口 |
| **截图压缩** | `screenshot` + `compression` 参数 | **高清屏截图压缩后传入 LLM，节省 95%+ token**（详见 protocols.md 截图优化协议） |
| **截图切片** | `screenshot` + `tile` 参数 | 密集数据页面分片分析（期权链、财务报表等） |
| 鼠标点击 | `mcp__macos-desktop-control__click` | 桌面 UI 交互（支持后台模式） |
| 键盘输入 | `mcp__macos-desktop-control__type_text` | 表单填写 |
| 按键操作 | `mcp__macos-desktop-control__key_press` | 快捷键、导航 |
| 滚动 | `mcp__macos-desktop-control__scroll` | 页面/列表滚动 |
| 窗口列表 | `mcp__macos-desktop-control__list_windows` | 查看所有窗口 |
| 打开应用 | `mcp__macos-desktop-control__open_app` | 启动被测应用 |
| iOS 模拟器 | `mcp__macos-desktop-control__sim_*` | 模拟器控制（如已安装 Xcode） |
| Android 模拟器 | `mcp__macos-desktop-control__emu_*` | 模拟器控制（如已安装 adb） |

**如果 `DESKTOP_CONTROL=not_found`**：

在 Step 0 输出中提示（不阻断执行）：

```
💡 推荐安装：macos-desktop-control 可解锁截图优化和桌面 App 测试能力。

  git clone https://github.com/d-wwei/macos-desktop-control.git ~/mcp-servers/macos-desktop-control
  cd ~/mcp-servers/macos-desktop-control && npm install
  claude mcp add macos-desktop-control -- node ~/mcp-servers/macos-desktop-control/src/index.js

  关键能力：
  · 📸 截图压缩 — Retina 高清屏截图压缩后再传入 LLM，token 节省 95%+（强烈推荐）
  · 🔲 截图切片 — 密集数据页面分片分析，避免压缩丢失细节
  · 🖥️ 桌面 App 控制 / 后台模式（不抢焦点）/ PlayCover iOS App / 模拟器直控
```

### 2. Browser Control Skill（Claude Code Skill）

**检测方法**：

```bash
# 检查 skill 是否已安装
if [ -f ~/.claude/skills/browse/SKILL.md ] || \
   [ -f ~/.claude/skills/browser-control/SKILL.md ]; then
  echo "BROWSER_CONTROL=installed"
else
  echo "BROWSER_CONTROL=not_found"
fi
```

**如果 `BROWSER_CONTROL=installed`**：

记录能力标志 `HAS_BROWSER_CONTROL=true`，后续步骤可通过 `/browse` 命令使用：

| 能力 | 调用方式 | 用途 |
|------|---------|------|
| 前台浏览 | `/browse here` | 操控用户当前 Chrome 页面 |
| 后台浏览 | `/browse bg` + CDP API | 独立 Tab 并行操作 |
| 交互元素索引 | AppleScript 扫描 | 获取页面所有可点击元素 |
| 表单填写 | click + type 组合 | 测试 Web 表单 |
| DOM 提取 | DOM→Markdown 转换 | 结构化提取页面内容 |
| 多 Tab 并行 | subagent 各开一个 Tab | 竞品网站并行研究 |
| 站点经验 | `references/site-patterns/` | 已知站点的操作模式 |

**如果 `BROWSER_CONTROL=not_found`**：

在 Step 0 输出中提示（不阻断执行）：

```
💡 可选增强：安装 browser-control-skill 可解锁 Web 产品测试和竞品网站研究能力。

  git clone https://github.com/d-wwei/browser-control-skill.git ~/.claude/skills/browser-control-skill
  ln -sf ~/.claude/skills/browser-control-skill/skills/browse ~/.claude/skills/browse
  ln -sf ~/.claude/skills/browser-control-skill/skills/browser-control ~/.claude/skills/browser-control

  能力：真实 Chrome 会话控制 / 认证页面操作 / 多 Tab 并行 / DOM 结构化提取
```

---

## 能力注册表

探测完成后，主 agent 维护一个能力注册表（内存中，不写文件）：

```
CAPABILITIES = {
  mobile_mcp: true/false,           # 移动端模拟器控制（core 依赖）
  desktop_control: true/false,      # macOS 桌面控制（可选）
  browser_control: true/false,      # Chrome 浏览器控制（可选）
}
```

后续各 Step 根据此表决定可用的操作方式。

---

## 能力增强场景

### 场景 1: 桌面 App 产品测试（需 desktop_control）

当用户要测试的是 macOS 桌面应用（非移动 App）时：

```
检测逻辑：
- 用户说 "体验一下 Figma 桌面版" / "测评 Notion 桌面客户端"
- 或 Step 2 中发现目标 App 不在模拟器的 app list 中但在 macOS 上已安装

如果 HAS_DESKTOP_CONTROL:
  → 使用 desktop_control 工具集替代 mobile-mcp
  → open_app 启动 → screenshot 截图 → click/type 交互 → scroll 滚动
  → 后台模式：所有操作加 target: { app: "AppName" }，不抢用户焦点
  → Step 5 的 subagent prompt 中替换工具列表为 desktop_control 工具

如果 !HAS_DESKTOP_CONTROL:
  → 告知用户：此 App 是桌面应用，需要安装 macos-desktop-control 才能自动操控
  → 提供安装命令
  → 或建议用户手动操作，agent 通过截图分析提供评价（降级模式）
```

### 场景 2: Web 产品测试（需 browser_control）

当用户要测试的是 Web 应用时：

```
检测逻辑：
- 用户说 "体验一下 Notion 网页版" / "测评 linear.app"
- 或给了一个 URL

如果 HAS_BROWSER_CONTROL:
  → 使用 /browse 能力控制 Chrome
  → 前台模式：操控用户可见的 Chrome 页面
  → 后台模式：CDP 开独立 Tab，不干扰用户浏览
  → Step 5 的 subagent 用 /browse bg 在独立 Tab 中探索各功能模块
  → 竞品对比时：多个 subagent 并行开 Tab 研究不同竞品

如果 !HAS_BROWSER_CONTROL:
  → 降级到 WebFetch + WebSearch（只能读取公开页面，不能交互）
  → 告知用户：安装 browser-control-skill 可解锁完整 Web 交互测试
  → 提供安装命令
```

### 场景 3: 移动 App 测试 + Web 版对比（联合使用）

最有价值的联合场景：同一产品的移动端和 Web 端对比。

```
如果 HAS_MOBILE_MCP && HAS_BROWSER_CONTROL:
  → 主 agent 可以安排：
    1. mobile-mcp subagent 群探索 App 移动端
    2. /browse bg subagent 并行探索 Web 版
    3. 汇总后对比：功能差异、交互差异、信息密度差异
  → 在报告中新增"跨平台对比"章节
```

### 场景 4: 官方帮助中心深度抓取（需 browser_control）

Step 7b（说明书生成）需要检索官方帮助中心。有些帮助中心需要登录或有反爬。

```
如果 HAS_BROWSER_CONTROL:
  → 用 /browse bg 打开帮助中心，利用用户 Chrome 的登录态
  → DOM 提取获取结构化内容
  → 比 WebFetch 更完整（能拿到 SPA 渲染后的内容）

如果 !HAS_BROWSER_CONTROL:
  → 降级到 WebFetch（当前行为，已够用大多数情况）
```

### 场景 5: PlayCover App 测试（需 desktop_control）

用 PlayCover 在 Mac 上运行 iOS App 进行测试：

```
如果 HAS_DESKTOP_CONTROL:
  → 用 playcover-setup 子命令配置 iPhone 模式
  → 然后用 desktop_control 工具集操控 PlayCover 窗口
  → 本质上和桌面 App 测试相同，但目标是 iOS App 在 Mac 上的运行实例

如果 !HAS_DESKTOP_CONTROL:
  → 建议用户用 iOS 模拟器 + mobile-mcp 替代
```

---

## Step 5 Subagent 工具集动态选择

subagent prompt 中的工具列表应根据能力注册表动态选择：

```
如果目标是移动 App（默认）:
  工具集 = mobile-mcp tools（mobile_click, mobile_swipe, mobile_save_screenshot...）

如果目标是桌面 App 且 HAS_DESKTOP_CONTROL:
  工具集 = desktop_control tools（click, type_text, screenshot, scroll...）
  所有工具加 target: { app: "{app_name}" } 参数（后台模式）

如果目标是 Web App 且 HAS_BROWSER_CONTROL:
  工具集 = browser_control（/browse bg + CDP API）
  subagent 内部用 Bash 调用 CDP 端点
```

主 agent 在构建 subagent prompt 时，根据目标平台和能力注册表选择对应工具集。
工具集的具体命令参考：
- mobile-mcp: 见 modules/protocols.md 的操作速查
- desktop_control: 见本文件上方的工具表
- browser_control: 通过 /browse 命令调用，详见 browser-control-skill 的 SKILL.md

---

## 降级策略总结

| 能力缺失 | 影响 | 降级方案 |
|---------|------|---------|
| mobile-mcp 缺失 | 无法测试移动 App | **阻断**——提示安装，无法降级 |
| desktop_control 缺失 | 无法测试桌面 App | 提示安装；或人工操作+截图分析 |
| browser_control 缺失 | 无法交互测试 Web | 降级到 WebFetch 只读模式 |
| 全部缺失 | 仅能做基于截图的评审 | 模式 C（评审模式）仍可用 |

**核心原则**：mobile-mcp 是唯一硬依赖。其余两个能力缺失时优雅降级，不阻断流程。
