# product-review skill CHANGELOG

---

## v3: 截图安全协议强制化 + 真机分辨率检测修复（2026-04-12）

> 基于 moomoo 技术面视频制作 session 中真机截图累积导致 session 崩溃的经验。

### 问题根因

v2 协议检测设备分辨率时使用 `mobile_get_screen_size`，返回的是**逻辑分辨率**而非物理像素。
iPhone 16 Pro Max 返回 440×956（逻辑），但实际截图为 1320×2868（3x Retina），远超 2000px 限制。
系统误判为低分辨率设备 → 绕过安全截图流程 → 30 次 `take_screenshot` 累积 101 张 inline 图片（3.4MB）→ session 被迫中断。

### 解决方案

1. **检测逻辑改为实际截图像素**：先 `save_screenshot` 一张测试图，再用 `sips -g pixelWidth -g pixelHeight` 获取真实像素
2. **删除 LOW_RES 豁免**：无论设备分辨率高低，product-review 流程中始终强制安全截图流程
3. **消除所有条件分支**：subagent 注入模板和广度扫描首截图不再依赖 `HIGH_RES_DEVICE` 条件判断

### 改动文件

| 文件 | 改动 |
|------|------|
| `step-0-4-setup.md` | 重写高分辨率检测：save→sips -g pixelWidth 实测，删除条件截图分支 |
| `protocols.md` | 核心规则改为无条件禁止 take_screenshot；删除 LOW_RES 豁免；更新 subagent 注入模板 |
| `step-5-exploration.md` | 广度扫描首截图改为无条件安全流程；subagent 注入段落删除 if 条件 |
| `SKILL.md` | 注意事项更新为强制截图安全 |

---

## v2: 双图截图协议 originals/thumbnails（2026-04-11）

> 基于 moomoo 产品力素材制作中两次 session 崩溃的经验，升级截图安全协议。

### 问题根因

v1 协议用 `sips --resampleHeightWidthMax 1800` **原地覆盖**原图，存在两个问题：
1. **原图丢失**：sips 直接修改源文件，最终文档/视频生成时无法使用完整分辨率图片
2. **1800px 仍然偏大**：每张缩放后仍 ~300KB，积累 15+ 张就撑爆 context

### 解决方案

引入 originals/thumbnails 双目录架构：
- 原图保存到 `originals/`，不做任何处理
- 缩略图通过 `sips --resampleHeightWidthMax 800 -s format jpeg -s formatOptions 70` 生成到 `thumbnails/`
- 会话内只 Read 缩略图（~25KB vs 旧版 ~300KB，压缩 12 倍）
- 最终文档/视频生成时通过同名文件映射替换为原图

| 指标 | v1 (1800px PNG) | v2 (800px JPEG q70) | 提升 |
|------|----------------|---------------------|------|
| 单张大小 | ~300KB | ~25KB | 12x |
| 安全容纳截图数 | ~15 张 | ~100+ 张 | 7x+ |
| 原图保留 | 否 | 是 | — |

### 改动文件

| 文件 | 改动 |
|------|------|
| `protocols.md` | 重写「多图尺寸安全协议」：双目录+三步流程+场景映射+subagent 模板 |
| `step-0-4-setup.md` | Step 1 输出更新；Step 2 改为双图流程；Step 4 创建双子目录 |
| `step-5-exploration.md` | 广度扫描首截图改为双图流程；subagent 注入更新 |

---

## v1: 初始多图安全协议 + Subagent 增量写入 + Persona 精简 + 内容评估（2026-04-07）

> 基于一轮完整的富途牛牛内容板块评测中发现的实际问题。

### 1. 多图尺寸安全协议（修复 session 崩溃 bug）

Setup 阶段检测设备分辨率，超 2000px 时设 `HIGH_RES_DEVICE=true`，所有截图走 `save_screenshot → sips → Read` 三步流程。

验证结果：4 个 subagent 共数百次截图操作，未再触发 2000px 限制错误。

### 2. Subagent 增量写入协议（防崩溃数据丢失）

- 分段写入：每探索一个页面后立即 Edit 追加报告段落
- 进度心跳：每完成子任务更新 exploration-queue.md 状态
- Context 预估：Read 图片 ≥12 次后只 save 不 Read，工具调用 ≥250 时强制退出
- 内部错误标记：先 ls 检查文件再标记，支持增量续派

### 3. Persona 模块精简（-68% context 占用）

分离数据与逻辑：`step-5.5-6-personas.md` 742→236 行，`persona-templates.md` 新增 148 行数据文件。

### 4. 内容评估模块（新增 Step 6.8）

六维内容评估框架（质量/生态/分发/架构/消费/透明度），支持 `content-review` 子命令。
新增 `step-6.8-content-evaluation.md` (~400 行) + `content-expert-roles.md` (120 行)。

### 改动文件

| 文件 | 状态 | 说明 |
|------|------|------|
| `SKILL.md` | 修改 | 注意事项 + 模块列表 + content-review 子命令 |
| `protocols.md` | 修改 | +多图安全协议 +增量写入协议 |
| `step-0-4-setup.md` | 修改 | +分辨率检测 +安全截图流程 |
| `step-5-exploration.md` | 修改 | +Fail-Fast 增量续派 +subagent 截图安全注入 |
| `step-5.5-6-personas.md` | **重写** | 742→236 行，调度逻辑精简 |
| `step-6.8-content-evaluation.md` | 修改 | 专家角色分离 478→408 行 |
| `persona-templates.md` | **新增** | Persona 完整档案+行为指令数据文件 |
| `content-expert-roles.md` | **新增** | 内容评估专家角色库数据文件 |
| `step-7-reports.md` | 修改 | 新增 7f 内容评价报告模板 |
