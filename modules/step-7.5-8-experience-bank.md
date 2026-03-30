# Step 7.5 + Step 8: Experience Bank（经验沉淀与注入）

> 本文件由 SKILL.md 按需加载。所有子命令的 review 完成后加载 Step 7.5 沉淀经验。
> 非首次 review 时，Step 8 在 Step 5.5 之前加载以注入历史经验。

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
