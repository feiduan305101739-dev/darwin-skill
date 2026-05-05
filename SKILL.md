---
name: darwin-skill
description: 'Darwin Skill (达尔文.skill): autonomous skill optimizer inspired by Karpathy''s autoresearch. Evaluates SKILL.md files using an 8-dimension rubric (structure + effectiveness), runs hill-climbing with git version control, validates improvements through test prompts, and generates visual result cards. Use when user mentions "优化skill", "skill评分", "自动优化", "auto optimize", "skill质量检查", "达尔文", "darwin", "帮我改改skill", "skill怎么样", "提升skill质量", "skill review", "skill打分".'
---

# Darwin Skill

> 借鉴 Karpathy autoresearch 的自主实验循环，对 skills 进行持续优化。
> 核心理念：**评估 → 改进 → 实测验证 → 人类确认 → 保留或回滚 → 生成成果卡片**
> GitHub: <https://github.com/alchaincyf/darwin-skill>

---

## 设计哲学

autoresearch 的精髓：

1. **单一可编辑资产** — 每次只改一个 SKILL.md
2. **双重评估** — 结构评分（静态分析）+ 效果验证（跑测试看输出）
3. **棘轮机制** — 只保留改进，自动回滚退步
4. **独立评分** — 评分用子agent，避免「自己改自己评」的偏差
5. **人在回路** — 每个skill优化完后暂停，用户确认再继续

与纯结构审查的区别：不只看 SKILL.md 写得规不规范，更看改完后**实际跑出来的效果是否更好**。

---

## 评估 Rubric（8维度，总分100）

### 结构维度（60分）— 静态分析

| #   | 维度                | 权重 | 评分标准                                                 |
| --- | ------------------- | ---- | -------------------------------------------------------- |
| 1   | **Frontmatter质量** | 8    | name规范、description包含做什么+何时用+触发词、≤1024字符 |
| 2   | **工作流清晰度**    | 15   | 步骤明确可执行、有序号、每步有明确输入/输出              |
| 3   | **边界条件覆盖**    | 10   | 处理异常情况、有fallback路径、错误恢复                   |
| 4   | **检查点设计**      | 7    | 关键决策前有用户确认、防止自主失控                       |
| 5   | **指令具体性**      | 15   | 不模糊、有具体参数/格式/示例、可直接执行                 |
| 6   | **资源整合度**      | 5    | references/scripts/assets引用正确、路径可达              |

### 效果维度（40分）— 需要实测

| #   | 维度         | 权重 | 评分标准                                            |
| --- | ------------ | ---- | --------------------------------------------------- |
| 7   | **整体架构** | 15   | 结构层次清晰、不冗余不遗漏、与花叔生态一致          |
| 8   | **实测表现** | 25   | 用测试prompt跑一遍，输出质量是否符合skill宣称的能力 |

### 评分规则

- 维度1-7：每个维度打 1-10 分，乘以权重得到该维度得分
- 维度8（实测表现）：跑2-3个测试prompt，按输出质量打1-10分
- **总分 = Σ(维度分 × 权重) / 10**，满分100
- 改进后总分必须 **严格高于** 改进前才保留

### 关于「实测表现」维度

这是与纯结构评分最大的区别。评分方式：

1. 为每个skill设计2-3个**典型用户prompt**（不是边缘case，是最常见的使用场景）
2. 用子agent执行：一个带skill跑，一个不带skill跑（baseline）
3. 对比输出质量，从以下角度打分：
   - 输出是否完成了用户意图？
   - 相比不带skill的baseline，质量提升明显吗？
   - 有没有skill引入的负面影响（过度冗余、跑偏、格式奇怪）？

如果无法跑子agent（时间/资源限制），可以退化为「干跑验证」：读完skill后模拟一个典型prompt的执行思路，判断流程是否合理。但要在results.tsv中标注 `dry_run`。

---

## 自主优化循环

### Phase 0: 初始化

1. **确认优化范围**
   - **输入**: 用户指令 (e.g., "优化所有skills" 或 "优化指定skill")。
   - **动作**: 扫描 `.claude/skills/*/SKILL.md` 确定全部技能列表，或提取用户指定的技能列表。
   - **输出**: 待优化的 `skills` 列表。
2. **创建 Git 分支**
   - **输入**: 当前时间戳。
   - **动作**: 执行 `git checkout -b auto-optimize/YYYYMMDD-HHMM`。
   - **输出**: 新的 Git 工作分支。
3. **初始化结果记录表**
   - **输入**: `.claude/skills/darwin-skill/results.tsv` 文件路径。
   - **动作**: 检查文件是否存在。如不存在，创建并写入表头（包含 `eval_mode` 等9列）。
   - **输出**: 可用的 `results.tsv` 文件。
4. **加载历史记录**
   - **输入**: `results.tsv` 文件内容。
   - **动作**: 读取并解析历史优化记录，了解当前 baseline 分数。
   - **输出**: 历史记录数据对象。

### Phase 0.5: 测试Prompt设计

在评估之前，为每个 skill 设计测试 prompt。这步很关键——没有测试 prompt，「实测表现」维度就打不了分。

针对待优化列表中的每个 skill，循环执行以下步骤：

1. **读取 Skill 详情**
   - **输入**: 当前 skill 的 `SKILL.md` 文件路径。
   - **动作**: 读取全文，理解核心功能和工作流。
   - **输出**: Skill 上下文理解。
2. **设计测试 Prompt**
   - **输入**: Skill 上下文。
   - **动作**: 设计 2-3 个覆盖典型场景 (happy path) 和复杂/歧义场景的测试 prompt。
   - **输出**: JSON 格式的测试用例列表。
3. **保存测试用例**
   - **输入**: JSON 格式的测试用例列表。
   - **动作**: 写入或覆盖该 skill 目录下的 `test-prompts.json`。格式：`[{"id": 1, "prompt": "...", "expected": "..."}]`。
   - **输出**: 落盘的 `test-prompts.json` 文件。

**检查点**：展示所有生成的测试 prompt 给用户，**等待确认后再进入评估**。测试 prompt 的质量决定了优化方向是否正确。

### Phase 1: 基线评估（Baseline）

针对待优化列表中的每个 skill，循环执行以下步骤：

1. **结构评分 (主 Agent 静态分析)**
   - **输入**: 当前 skill 的 `SKILL.md` 文件内容。
   - **动作**: 依据评估 Rubric，对维度 1-7 逐项打分，并记录简短理由。
   - **输出**: 结构维度得分 (满分 60)。
2. **效果评分 (子 Agent 实测)**
   - **输入**: 该 skill 的 `test-prompts.json`。
   - **动作**: 对每个 prompt spawn 独立子 agent 进行两次测试：(1) 带着当前 `SKILL.md` 执行；(2) 不带 skill 执行 (baseline)。对比两组输出打出维度 8 的分数。
   - **Fallback (干跑验证)**: 如果子 agent 不可用（超时、环境限制），则模拟典型 prompt 的执行思路打分，并在记录中标注 `eval_mode = dry_run`。绝不静默跳过此维度。
   - **输出**: 效果维度得分 (满分 40)。
3. **汇总与记录**
   - **输入**: 结构得分与效果得分。
   - **动作**: 计算加权总分，并将结果 (包含 `timestamp`, `commit` 设为 'baseline', `old_score` 设为 '-', `new_score`, `eval_mode` 等) 追加一行到 `results.tsv`。
   - **输出**: 更新后的 `results.tsv`。

**检查点**：基线评估完成后，生成并展示所有被评估 skill 的评分卡对比表格（包含 Skill 名称、总分、结构短板、效果短板）。**暂停等待用户确认，再进入 Phase 2。**

### Phase 2: 优化循环

用户确认后，按基线分数从低到高排序，对每个 skill 依次进行以下优化循环（最大轮次 `MAX_ROUNDS` 默认 = 3）：

1. **短板诊断**
   - **输入**: 当前 skill 的各项维度得分。
   - **动作**: 找出得分最低的维度（结构或效果均可）。
   - **输出**: 需要重点优化的目标维度。
2. **生成改进方案**
   - **输入**: 目标维度及当前 `SKILL.md` 的相关内容。
   - **动作**: 生成 1 个具体的改进方案，明确说明：改什么（具体段落/行）、为什么改（对应 rubric 哪条）、预期提升多少分。
   - **输出**: 改进方案摘要。
3. **执行代码变更**
   - **输入**: 改进方案。
   - **动作**: 使用文件编辑工具修改 `SKILL.md`。修改后执行 `git add SKILL.md` 和 `git commit -m "optimize {skill}: {改进摘要}"`。
   - **输出**: 包含新改进的 Git commit。
4. **重新评估**
   - **输入**: 修改后的 `SKILL.md`。
   - **动作**: 重新执行 Phase 1 中的步骤 1 (结构评分) 和 步骤 2 (效果评分，必须 spawn 独立子 agent 或干跑以保证公正)。
   - **输出**: 改进后的新总分。
5. **棘轮决策 (Ratchet Decision)**
   - **输入**: 新总分 vs 旧总分。
   - **动作**:
     - **Keep**: 如果 新总分 > 旧总分，保留更改，更新当前最佳分数。
     - **Revert**: 如果 新总分 <= 旧总分，执行 `git revert HEAD` 创建新 commit 干净回滚，停止该 skill 的优化（判定为到达局部最优瓶颈），并跳出当前 skill 循环。
   - **输出**: 最终确定的代码状态。
6. **写入日志**
   - **输入**: 评估及决策结果。
   - **动作**: 将本次尝试（无论 keep 还是 revert）的结果追加到 `results.tsv`。
   - **输出**: 更新后的 `results.tsv`。

**检查点**：每个 skill 优化循环结束（触发 revert 或达到 `MAX_ROUNDS`）后，暂停并向用户展示该 skill 的改动摘要：

- `git diff`（改前最优 vs 当前最优）
- 分数变化明细
- 测试 prompt 输出对比（如果进行了实测）
  **等待用户输入 "OK" 后，继续下一个 skill。** 如果用户输入 "不好" 或拒绝，则手动回滚到该 skill 优化前的版本。

### Phase 2.5: 探索性重写（可选）

当 hill-climbing 连续2个skill都在 round 1 就 break（涨不动）时，这可能陷入了局部最优问题。此时提议一次「探索性重写」（**必须征得用户同意后执行**）：

1. **保存当前最优版本**
   - **输入**: 当前工作区状态。
   - **动作**: 执行 `git stash` 暂存当前通过验证的最佳版本。
   - **输出**: 干净的工作区。
2. **从头重写**
   - **输入**: 原 `SKILL.md` 的核心功能意图和约束规则。
   - **动作**: 不是微调，而是完全重新组织结构和表达方式来重写 `SKILL.md` 全文。
   - **输出**: 重写后的 `SKILL.md` 文件。
3. **重新评估与决策**
   - **输入**: 重写后的 `SKILL.md` 和暂存区版本。
   - **动作**: 重新执行 Phase 1（结构+效果评分）。如果 重写版总分 > 暂存版总分，则采用重写版；否则，执行 `git stash pop` 恢复暂存版本。
   - **输出**: 突破瓶颈的新版本或恢复旧版本。

### Phase 3: 汇总报告

所有选中 skills 优化完成后，生成全局报告：

1. **统计全局数据**
   - **输入**: `results.tsv` 和 Git commit 历史。
   - **动作**: 汇总计算总优化 skills 数量、总实验次数、保留比例 (keep vs revert)、实测/干跑次数。
   - **输出**: 汇总统计数据。
2. **生成对比表格**
   - **输入**: `results.tsv` 中每个 skill 的 base 分数和最新 keep 分数。
   - **动作**: 生成分差对比表格（包含 Skill 名称、Before、After、Δ）。
   - **输出**: Markdown 格式的分数变化表。
3. **总结核心改进**
   - **输入**: 历次 keep 状态的改进方案摘要。
   - **动作**: 归纳整理出最显著的几项主要改进点。
   - **输出**: 核心改进列表。
4. **输出最终报告**
   - **输入**: 以上步骤的所有输出数据。
   - **动作**: 组装成完整的《优化报告》Markdown 文档，并展示给用户。
   - **输出**: 向用户显示的最终汇总报告文本。

---

## results.tsv 格式

```tsv
timestamp commit skill old_score new_score status dimension note eval_mode
2026-03-31T10:00 baseline huashu-proofreading - 78 baseline - 初始评估 full_test
2026-03-31T10:05 a1b2c3d huashu-proofreading 78 84 keep 边界条件 补充fallback full_test
2026-03-31T10:10 b2c3d4e huashu-proofreading 84 82 revert 指令具体性 过度细化 dry_run
```

新增 `eval_mode` 列：`full_test`（跑了子agent测试）或 `dry_run`（模拟推演）。
文件位置：`.claude/skills/darwin-skill/results.tsv`

---

## 优化策略库

按优先级排序，每轮只做最高优先级的一个：

### P0: 效果问题（实测发现的）

- 测试输出偏离用户意图 → 检查skill是否有误导性指令
- 带skill比不带还差 → skill可能过度约束，考虑精简
- 输出格式不符合预期 → 补充明确的输出模板

### P1: 结构性问题

- Frontmatter缺少触发词 → 补充中英文触发词
- 缺少Phase/Step结构 → 重组为线性流程
- 缺少用户确认检查点 → 在关键决策处插入

### P2: 具体性问题

- 步骤模糊（"处理图片"）→ 改为具体操作和参数
- 缺少输入/输出规格 → 补充格式、路径、示例
- 缺少异常处理 → 补充 "如果X失败，则Y"

### P3: 可读性问题

- 段落过长 → 拆分+用表格
- 重复描述 → 合并去重
- 缺少速查 → 添加TL;DR或决策树

---

## 异常与边界条件

流程假设环境理想，但实操常遇异常。以下预定义 fallback，保证优化过程不会「一跑就卡住」。

| 场景                     | 触发条件               | 处理动作                                                                                           |
| ------------------------ | ---------------------- | -------------------------------------------------------------------------------------------------- |
| 不在 git 仓库            | `git rev-parse` 失败   | 提示用户「建议 git init」；若拒绝，用 `cp SKILL.md SKILL.md.bak.YYYYMMDD-HHMM` 文件备份代替 revert |
| results.tsv 缺失         | 文件不存在             | 新建并写表头行（9列：含 eval_mode）                                                                |
| results.tsv 损坏         | 列数不匹配 / 非TSV     | 备份为 `.bak.YYYYMMDD-HHMM` 后重建，告知用户                                                       |
| 分支已存在               | `git checkout -b` 失败 | 分支名末尾加 `-2` / `-3`；第3次失败则切回现有分支并询问继续还是新起                                |
| `git revert` 失败        | 冲突 / 工作树脏        | 先 `git stash`，重试；仍失败则从上一个 commit 的 SKILL.md 读出覆盖当前文件手动恢复                 |
| MAX_ROUNDS 触顶（默认3） | 已跑3轮仍有短板        | 不强制 break，展示当前最弱维度问用户「继续加1轮 / 进入Phase 2.5 / 收工」                           |
| 优化后超 150% 体积       | 新文件 > 原 × 1.5      | 拒绝提交，回到改进步骤精简（删冗余/合并重复），再评                                                |
| test-prompts.json 已存在 | 文件已在 skill 目录    | 默认复用并展示，问用户「复用 / 重写 / 追加」三选一                                                 |
| SKILL.md 找不到          | 目录存在但无 SKILL.md  | 该 skill 终止，results.tsv 记 `status=error`，继续下一个                                           |
| 分数计算规则             | 浮点精度漂移           | 总分保留 1 位小数，改进需严格 > 旧分（不靠四舍五入）                                               |

**原则**：异常先告知用户，再按规则处理；绝不静默跳过或静默失败。

---

## 约束规则

1. **不改变skill的核心功能和用途** — 只优化"怎么写"和"怎么执行"，不改"做什么"
2. **不引入新依赖** — 不添加skill原本没有的scripts或references文件
3. **每轮只改一个维度** — 避免多个变更导致无法归因
4. **保持文件大小合理** — 优化后SKILL.md不应超过原始大小的150%
5. **尊重花叔风格** — 中文为主、简洁为上
6. **可回滚** — 所有改动在git分支上，用git revert而非reset --hard
7. **评分独立性** — 效果维度必须用子agent或至少干跑验证，不能在同一上下文里「改完直接评」

---

## 使用方式

### 全量优化（推荐首次使用）

```
用户："优化所有skills"
→ Phase 0-3 完整流程
→ 建议：先基线评估，选择分数最低的5-10个重点优化
```

### 单个优化

```
用户："优化 huashu-slides 这个skill"
→ 只对指定skill执行 Phase 0.5-2
```

### 仅评估不改

```
用户："评估所有skills的质量"
→ 只执行 Phase 0.5-1（设计测试prompt + 基线评估），不进入优化循环
```

### 查看历史

```
用户："看看skill优化历史"
→ 读取并展示 results.tsv
```

---

## 设计灵感

> "You write the goals and constraints in program.md; let an agent generate and test code deltas indefinitely; keep only what measurably improves the objective."
> — Karpathy, autoresearch

本skill的对应关系：

- **program.md** → 本文件（评估rubric和约束规则）
- **train.py** → 每个SKILL.md
- **val_bpb** → 8维加权总分（含实测表现）
- **git ratchet** → 只保留有改进的commit
- **test set** → 每个skill的test-prompts.json

区别：增加了人在回路（autoresearch是全自主的，skill优化需要人的判断力），以及双重评估机制（结构+效果），因为skill的「好坏」比loss数值更微妙。

---

## 成果卡片生成（Result Card）

每个skill优化完成后（或全量汇总后），自动生成视觉成果卡片，截图保存为PNG。

### 卡片模板

模板位置：`templates/result-card.html`

3种风格，每次随机选择一种：

| 风格          | CSS类              | URL hash     | 视觉特点                           |
| ------------- | ------------------ | ------------ | ---------------------------------- |
| Warm Swiss    | `.theme-swiss`     | `#swiss`     | 暖白底+赤陶橙，Inter字体，干净网格 |
| Dark Terminal | `.theme-terminal`  | `#terminal`  | 近黑底+荧光绿，等宽字体，扫描线    |
| Newspaper     | `.theme-newspaper` | `#newspaper` | 暖白纸+深红，衬线字体，双栏编辑风  |

### 生成流程

1. **准备模板文件**
   - **输入**: `templates/result-card.html` 路径。
   - **动作**: 复制该文件到临时工作目录（例如 `/tmp/card.html`）。
   - **输出**: 可编辑的临时 HTML 文件。
2. **填充数据**
   - **输入**: 临时 HTML 文件及当前 skill 的优化数据（分数、改进摘要等）。
   - **动作**: 使用文件编辑工具或正则替换对应的 `data-field` 占位符：
     - `skill-name` → 实际 skill 名
     - `score-before/after/delta` → 实际分数
     - 8 个维度的 `dim-bar-before/after` 的 CSS width → 实际百分比
     - `improvement-1/2/3` → 实际改进摘要
     - `date` → 当前日期
   - **输出**: 已填充数据的 HTML 文件。
3. **随机选择风格**
   - **输入**: 主题列表 `['swiss', 'terminal', 'newspaper']`。
   - **动作**: 随机选择一个主题 hash 追加到 URL 后（例如 `#swiss`）。
   - **输出**: 带主题 hash 的本地 URL (e.g., `file:///tmp/card.html#swiss`)。
4. **生成截图**
   - **输入**: 带主题 hash 的本地 URL。
   - **动作**: 运行 Node 脚本 `node scripts/screenshot.mjs /tmp/card.html /tmp/output.png`。
     - _Fallback (回退方案)_: 若脚本失败，尝试执行 Playwright 截图：`npx playwright screenshot "file:///tmp/card.html#swiss" output.png --viewport-size=960,1280 --wait-for-timeout=2000`。
   - **输出**: 生成的 PNG 成果卡片文件。
5. **展示给用户**
   - **输入**: PNG 文件路径。
   - **动作**: 调用图片查看工具展示 PNG 文件，或将生成的卡片直接返回/提示给用户查看。
   - **输出**: 用户确认看到成果卡片。

### 资源文件速查

| 路径                                              | 用途                                              |
| ------------------------------------------------- | ------------------------------------------------- |
| `templates/result-card.html`                      | 3风格主模板（swiss/terminal/newspaper，hash切换） |
| `templates/result-card-dark.html` / `-white.html` | 单一风格替代模板（需要锁定风格时用）              |
| `scripts/screenshot.mjs`                          | 2x 高清截图，只截 .card，自动 open                |
| `results.tsv`                                     | 历次优化日志（9列含 eval_mode）                   |
| `{skill目录}/test-prompts.json`                   | 每个 skill 的测试 prompt 集（用于维度8实测）      |

### 何时生成

- **单skill卡片**：每个skill优化完成后，展示该skill的分数变化
- **总览卡片**：全部优化完成后（Phase 3），展示全局战绩

### 品牌元素

- 顶部：Darwin.skill 品牌标识 + 日期
- 底部：「Train your Skills like you train your models」+ github.com/alchaincyf/darwin-skill
