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
- **桌面 App**（macOS）→ 通过 macos-desktop-control 操控（可选安装，同时提供截图压缩/切片能力）
- **Web App** → 通过 browser-control-skill 控制 Chrome（可选安装）

只有 mobile-mcp 是硬依赖。桌面和 Web 能力缺失时优雅降级，不阻断流程。
**强烈推荐**安装 macos-desktop-control：Retina 高清屏截图经压缩后传入 LLM，token 节省 95%+，详见 `protocols.md` 截图优化协议。

---

## 模块化架构

本 skill 采用按需加载架构。SKILL.md 只包含路由逻辑和执行流程概览。
详细指令分散在 `modules/` 目录下，执行到对应步骤时才 Read 加载。

```
modules/
  integrations.md              # 外部 skill 集成：能力探测 + 推荐安装 + 场景增强 (238 行)
  step-0-4-setup.md            # 环境检查 → 设备 → 触控验证 → App → 模式 → 目录 (310 行)
  step-5-exploration.md        # 自主探索：广度扫描 + 细粒度分组 + 完成驱动机制 (700 行)
  step-5.5-6-personas.md       # 用户画像 + Persona 模拟三模式 (665 行)
  step-6.5-expert-review.md    # 专家评审：工作流缺口分析 + 功能机会 + 产品创新洞察 (280 行)
  step-7-reports.md            # 报告模板：UX/手册/优化建议/专家评审 + 主agent直写 (800 行)
  step-7.5-8-experience-bank.md # 经验沉淀 + 经验注入 (548 行)
  protocols.md                 # 操作速查 + list_elements纪律 + 数据压缩 + 深度探索协议 (352 行)
  appendix-domain-experts.md   # 领域专家原型库 (296 行)
  setup-commands.md            # 注意事项 + 4 个 Setup 子命令 + 多模拟器指南 (247 行)
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
| `/product-review expert-review [App]` | 领域专家级产品评审（工作流缺口+功能机会+创新建议） | `expert-review.md` |
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
1. 从参数中提取 --mode=X、--personas=N 和 --expert-review 标志（如有），记录后从参数中移除
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
   5.5 如果是 "expert-review" → 模式 = EXPERT_REVIEW，剩余参数 = App 名称
6. 其他情况 → 模式 = INTERACTIVE，整个参数 = App 名称
7. 如果提取到 --mode 标志，记录 persona_mode = A/B/C（默认 B）
8. 如果提取到 --personas 标志，记录 persona_count = N（默认由 Step 5.5 决定）
9. 如果提取到 --expert-review 标志，记录 EXPERT_REVIEW_ENABLED = true
```

**模式映射：**

| 模式 | 探索深度 | 报告类型 | Persona | 专家评审 | 需加载的模块 |
|------|----------|----------|---------|---------|-------------|
| UE_REPORT | 完整 | 体验报告 | 可选(默认开) | 可选 | setup → exploration → personas → [expert-review] → reports(7a) → exp-bank |
| MANUAL_CORE | 核心功能 | 说明书 Core | 无 | 可选 | setup → exploration → [expert-review] → reports(7b) → exp-bank |
| MANUAL_MECE | 穷尽 | 说明书 MECE | 无 | 可选 | setup → exploration → [expert-review] → reports(7b) → exp-bank |
| SUGGESTION | 完整+模拟 | 优化建议 | 必须 | 推荐 | setup → exploration → personas → [expert-review] → reports(7c) → exp-bank |
| EXPERT_REVIEW | 完整 | 专家评审报告 | 无 | **必须** | setup → exploration → expert-review → reports(7e) → exp-bank |
| INTERACTIVE | 用户选择 | 用户选择 | 用户选择 | 用户选择 | 按选择加载 |
| *_SETUP | — | — | — | — | setup-commands.md 仅对应段落 |

**Persona 模拟参数**（可选）：

```
/product-review suggestion --mode=A 抖音     → 全自主体验
/product-review suggestion --mode=B 小红书   → 混合体验（默认）
/product-review suggestion --mode=C WeChat   → 全评审体验
/product-review suggestion --personas=8 抖音  → 指定 8 个 persona
/product-review suggestion --expert-review moomoo → 优化建议 + 专家评审
/product-review expert-review moomoo         → 仅专家评审（跳过 Persona 模拟）
/product-review manual mece --expert-review moomoo → 详尽说明书 + 专家评审
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
  → 执行 Step 0（环境检查）→ Step 1（设备+多设备检测）→ Step 2（App）
  → Step 2.5（触控能力验证）→ Step 3（模式选择）→ Step 4（目录）

Phase 2: 探索
  → Read modules/step-5-exploration.md
  → 执行 Step 5（广度扫描 → 复杂度估算+细粒度分组 → 完成驱动的 subagent 分派 → 验证补扫）
  → 如 TOUCH_BUG_DETECTED=true，subagent 使用触控降级策略
  → 如 PARALLEL_MODE=true，多设备并行分派
  → 探索过程中，subagent 需要时 Read modules/protocols.md 获取操作速查和滚动协议

Phase 3: Persona 模拟（仅 SUGGESTION / UE_REPORT 需要）
  → Read modules/step-7.5-8-experience-bank.md（检查 experience bank 是否存在，执行 Step 8 经验注入）
  → Read modules/step-5.5-6-personas.md
  → 执行 Step 5.5（画像分析）→ Step 6（Persona 生成 + 体验 + 聚合）
  → 如果产品涉及专业领域：Read modules/appendix-domain-experts.md

Phase 3.5: 专家评审（EXPERT_REVIEW 模式必须 / --expert-review 标志 / 高专业度模块推荐）
  → Read modules/step-6.5-expert-review.md
  → 执行 Step 6.5（识别评审模块 → 生成专家 Persona → 执行评审 → 输出 expert-review.md）
  → 如需领域专家原型参考：Read modules/appendix-domain-experts.md

Phase 4: 报告生成
  → Read modules/step-7-reports.md（按子命令类型只看对应模板：7a/7b/7c/7e）
  → **规模预判**：统计 exploration-reports/ 文件数和总行数
    → 总行数 > 2000 或文件数 > 6：启用分批读取 + 拆分 agent 模式
  → **拆分策略**（大型报告必须拆分，不可交给单个 agent）：
    → **通用原则——按"推理复杂度"而非"行数"拆分**：
      - 描述性内容（ASCII 布局图/配色表/性能列表）：单 agent 可处理 400+ 行
      - 分析性内容（多模块×多维度交叉推理）：**≤3 个异构分析模块/agent**
      - 判断标准：独立推理任务数 = 分析模块数 × 评估维度数，超过 18 必须拆分
      - 合成性内容（汇总评分/竞品矩阵/改进排序）：单 agent 可处理，但需提供上游数据
    → **失败恢复协议**：
      1. agent 报错后，先 `ls` 检查输出文件是否已部分写入
      2. 如已写入：Read 检查质量，补写缺失部分
      3. 如未写入：**不要重试同一策略**（精简 prompt 无效，问题在复杂度），直接拆分为更小单元
      4. 拆分后每个子 agent 独立输出文件（如 partC1.md / partC2.md），最后 cat 合并
    → **数据传递模式**：
      - 主 agent 先读取所有源数据，构建精简数据摘要文件（≤100 行）
      - 每个 subagent 只读这一个摘要文件，不读原始大文件
      - 关键数据点通过 prompt 嵌入，补充摘要文件覆盖不到的细节
    → expert-review：按评审模块拆分为多个 subagent，每个读 1-2 个报告，主 agent 合并
    → manual-full：**v3 总-分流程**（详见 step-7-reports.md 7b-B 节）：
      1. 主 agent 先读取所有 exploration-reports，构建功能全景图
      2. 主 agent **亲自撰写**一~三章（产品概述 + 安装注册 + 主界面导航）
      3. 主 agent 确定一级模块列表和章节编号（每个底部 Tab = 一个独立章节）
      4. 按一级模块拆分 subagent：
         - **每个 subagent 只负责一个一级模块**（即一个底部 Tab）
         - 禁止把多个 Tab 合并到同一个 subagent
         - subagent 输出以 "## X.0 模块概览" 开头，不含全局标题
      5. 主 agent 合并：一~三章 + 各模块章节 + 尾部章节（账户设置/FAQ/附录）
      6. **合并后校验**：一~三章完整 + 章节编号连续 + 无 Part 分割痕迹 + 只有一个一级标题
    → optimization：主 agent 在 prompt 中内嵌精简摘要（<4000字），不让 agent 读大文件
    → ue-report（7a 体验报告）：**必须拆分**（详见 step-7-reports.md 7a 拆分策略）
  → 每份报告独立生成（串行），不在同一 agent 中同时写多份
  → 生成后验证：Read 报告确认行数和章节完整性
  → 如有专家评审结果，suggestion 模式的 optimization.md 中引用专家评审洞察

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
