---
name: product-review
description: |
  Use when the user asks to experience, review, or evaluate an app or product.
  Supports mobile apps (iOS/Android), desktop apps (macOS), and web apps.
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

AI agent 自主体验产品，撰写结构化体验报告。

支持三类产品平台：
- **移动 App**（iOS/Android）→ 通过 mobile-mcp 控制模拟器
- **桌面 App**（macOS）→ 通过 macos-desktop-control 操控（可选安装）
- **Web App** → 通过 browser-control-skill 控制 Chrome（可选安装）

只有 mobile-mcp 是硬依赖。桌面和 Web 能力缺失时优雅降级，不阻断流程。

---

## 模块化架构

本 skill 采用按需加载架构。SKILL.md 只包含路由逻辑和执行流程概览。
详细指令分散在 `modules/` 目录下，执行到对应步骤时才 Read 加载。

```
modules/
  integrations.md              # 外部 skill 集成：能力探测 + 推荐安装 + 场景增强 (210 行)
  step-0-4-setup.md            # 环境检查 → 设备 → App → 模式 → 目录 (192 行)
  step-5-exploration.md        # 自主探索：广度扫描 + One-Page-One-Agent (468 行)
  step-5.5-6-personas.md       # 用户画像 + Persona 模拟三模式 (665 行)
  step-7-reports.md            # 报告模板：UX/手册/优化建议 (734 行)
  step-7.5-8-experience-bank.md # 经验沉淀 + 经验注入 (548 行)
  protocols.md                 # 操作速查 + 数据源约束 + 深度探索协议 (286 行)
  appendix-domain-experts.md   # 领域专家原型库 (296 行)
  setup-commands.md            # 注意事项 + 4 个 Setup 子命令 (202 行)
```

**加载规则**：执行到某个 Step 时，用 Read 工具加载对应的 modules/ 文件。
不要预加载所有文件。每个文件开头有说明注释标明何时加载。

---

## 子命令

| 命令 | 说明 | 输出文件 |
|------|------|----------|
| `/product-review ue-report [App]` | 产品体验报告（UX 五层模型） | `report.md` |
| `/product-review manual core [App]` | 使用说明书 Core 模式（核心功能） | `manual-core.md` |
| `/product-review manual mece [App]` | 使用说明书详尽模式（MECE 穷尽） | `manual-full.md` |
| `/product-review suggestion [App]` | 功能/体验优化建议 | `optimization.md` |
| `/product-review playcover-setup [App]` | 配置 PlayCover iPhone 模式 | 终端输出 |
| `/product-review android-setup [App]` | 设置 Android 模拟器环境 | 终端输出 |
| `/product-review ios-sim-setup [App]` | 设置 iOS 模拟器环境 | 终端输出 |
| `/product-review ios-device-setup` | 连接 iOS 真机测试 | 终端输出 |
| `/product-review [App]` | 无子命令，交互式选择 | 按选择生成 |

---

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
   5.1 如果是 "playcover-setup" → 模式 = PLAYCOVER_SETUP
   5.2 如果是 "android-setup" → 模式 = ANDROID_SETUP
   5.3 如果是 "ios-sim-setup" → 模式 = IOS_SIM_SETUP
   5.4 如果是 "ios-device-setup" → 模式 = IOS_DEVICE_SETUP
6. 其他情况 → 模式 = INTERACTIVE，整个参数 = App 名称
7. 如果提取到 --mode 标志，记录 persona_mode = A/B/C（默认 B）
8. 如果提取到 --personas 标志，记录 persona_count = N（默认由 Step 5.5 决定）
```

**模式映射：**

| 模式 | 探索深度 | 报告类型 | Persona | 需加载的模块 |
|------|----------|----------|---------|-------------|
| UE_REPORT | 完整 | 体验报告 | 可选(默认开) | setup → exploration → personas → reports(7a) → exp-bank |
| MANUAL_CORE | 核心功能 | 说明书 Core | 无 | setup → exploration → reports(7b) → exp-bank |
| MANUAL_MECE | 穷尽 | 说明书 MECE | 无 | setup → exploration → reports(7b) → exp-bank |
| SUGGESTION | 完整+模拟 | 优化建议 | 必须 | setup → exploration → personas → reports(7c) → exp-bank |
| INTERACTIVE | 用户选择 | 用户选择 | 用户选择 | 按选择加载 |
| *_SETUP | — | — | — | setup-commands.md 仅对应段落 |

**Persona 模拟参数**（可选）：

```
/product-review suggestion --mode=A 抖音     → 全自主体验
/product-review suggestion --mode=B 小红书   → 混合体验（默认）
/product-review suggestion --mode=C WeChat   → 全评审体验
/product-review suggestion --personas=8 抖音  → 指定 8 个 persona
```

---

## 执行流程与模块加载

### Setup 子命令（直接跳转）

如果模式是 *_SETUP：
```
→ Read modules/setup-commands.md
→ 执行对应 setup 段落
→ 结束
```

### Review 子命令（主流程）

```
Phase 0: 能力探测
  → Read modules/integrations.md
  → 检测 mobile-mcp / desktop_control / browser_control 可用性
  → 构建能力注册表：CAPABILITIES = { mobile_mcp, desktop_control, browser_control }
  → 如有缺失的可选能力，输出安装建议（不阻断）
  → 根据目标产品类型（移动/桌面/Web）确定主工具集

Phase 1: 初始化
  → Read modules/step-0-4-setup.md
  → 执行 Step 0（环境检查）→ Step 1（设备）→ Step 2（App）→ Step 3（模式选择）→ Step 4（目录）

Phase 2: 探索
  → Read modules/step-5-exploration.md
  → 执行 Step 5（广度扫描 → 探索队列 → One-Page-One-Agent → 验证补扫）
  → 探索过程中，subagent 需要时 Read modules/protocols.md 获取操作速查和滚动协议

Phase 3: Persona 模拟（仅 SUGGESTION / UE_REPORT 需要）
  → Read modules/step-7.5-8-experience-bank.md（检查 experience bank 是否存在，执行 Step 8 经验注入）
  → Read modules/step-5.5-6-personas.md
  → 执行 Step 5.5（画像分析）→ Step 6（Persona 生成 + 体验 + 聚合）
  → 如果产品涉及专业领域：Read modules/appendix-domain-experts.md

Phase 4: 报告生成
  → Read modules/step-7-reports.md（按子命令类型只看对应模板：7a/7b/7c）
  → 生成报告

Phase 5: 经验沉淀
  → Read modules/step-7.5-8-experience-bank.md（如 Phase 3 未加载，此时加载）
  → 执行 Step 7.5（提取经验 → 写入 experience bank → 更新置信度）
```

### 模块加载量对比

| 子命令 | 加载模块数 | 预估行数 | vs 旧版(3466行) |
|--------|----------|---------|----------------|
| `suggestion` | 6-7 | ~2800 | -19% |
| `ue-report` | 5-6 | ~2500 | -28% |
| `manual mece` | 3 | ~1400 | **-60%** |
| `manual core` | 3 | ~1400 | **-60%** |
| `*-setup` | 1 | ~200 | **-94%** |
| SKILL.md 本身 | — | ~200 | **-94%** |

**关键收益**：SKILL.md 从 3466 行降到 ~200 行。任何子命令的首次上下文占用从 143KB 降到 ~8KB。后续模块按需加载，只占用实际需要的上下文。

---

## 注意事项（快速参考）

详细版见 `modules/setup-commands.md`。关键几条：

- 报告语言跟随用户习惯（默认中文）
- subagent 遇到需要登录的功能 → 先尝试其他入口，实在需要 → AskUserQuestion
- 所有产品数据只能来自 App 内或官方渠道，禁止第三方网站
- 每个页面必须执行滚动扫描协议（最少 3 次 swipe，详见 protocols.md）
- 敏感输入（交易密码等）→ 询问用户是否人工接力
- 数据准确性：所见即所录，不要凭记忆描述之前的屏幕
