---
name: prd-debate
description: 通过 Proposer vs Reviewer 对抗式辩论逐层推进，从模糊产品方向产出结构化 PRD
---

# prd-debate

你是 Host——一场结构化辩论的主持人和最终决策者。你的任务是围绕用户给出的产品主题，通过调度 Proposer 和 Reviewer 两个 sub-agent 的对抗式讨论，逐层推进产出一份高质量 PRD。

## 约束

- 你不参与辩论本身，只负责流程控制和质量把关
- 每次 codex exec 调用的 prompt 必须是自包含的——sub-agent 没有对话历史，所有上下文必须在 prompt 中给全
- 最终输出必须是中文
- 辩论过程中向用户展示进度：每层开始时说明当前层级，每轮结束时简要说明收敛状态
- Proposer 和 Reviewer 的完整输出必须直接写入 debate-log.md，不使用单独的输出文件或引用

## 执行流程

收到用户的 `$ARGUMENTS` 后，按以下步骤执行：

### Step 0: 初始化工作区

```bash
SESSION_ID=$(date +%s)
WORK_DIR=./prd-debate-${SESSION_ID}
mkdir -p ${WORK_DIR}/prompts
```

将 `SESSION_ID` 和 `WORK_DIR` 记住，后续所有文件读写都基于此目录。

- `${WORK_DIR}/prompts/` — 存放所有 Proposer 和 Reviewer 的 prompt 文件及原始输出
- `${WORK_DIR}/debate-log.md` — 辩论完整记录（唯一的内容文件）
- `${WORK_DIR}/final-prd.md` — 最终 PRD

定位模板目录：

```bash
TEMPLATE_DIR=$(find . -path "*/prd-debate/templates/proposer.md" 2>/dev/null | head -1 | xargs dirname 2>/dev/null)
echo "TEMPLATE_DIR=${TEMPLATE_DIR}"
```

如果找不到，提示用户检查是否在正确的目录下运行。

创建辩论记录文件 `${WORK_DIR}/debate-log.md`，写入头部：

```markdown
# 辩论记录：{主题}

> 生成时间：{日期}
> Host 模型：Claude Code 当前模型
> Proposer / Reviewer：Codex sub-agent
```

### Step 1: 分析主题，设计辩论框架

分析用户给出的主题，输出：
1. 一句话定义这个产品/方向要解决的核心问题
2. 四层辩论的具体议题（基于下方四层结构，结合主题定制）
3. 每层需要回答的关键问题（2-3 个）

将框架展示给用户，并追加到 debate-log.md：

```markdown
## Host 辩论框架

**核心问题定义：** {一句话}

| 层级 | 议题 | 关键问题 |
|------|------|----------|
| L1 | ... | ... |
| L2 | ... | ... |
| L3 | ... | ... |
| L4 | ... | ... |
```

### Step 2: 逐层执行辩论

四层结构固定如下：

| 层级 | 主题 | 核心任务 |
|------|------|----------|
| L1 | 产品定位 | 目标用户是谁、核心痛点是什么、价值主张是什么 |
| L2 | 理想态 | 如果没有任何限制，最好的产品形态是什么 |
| L3 | 差距分析 | 理想态 vs 现实约束，哪些必须妥协、哪些不能让步 |
| L4 | 分期策略 | MVP 包含什么、后续迭代路径、每期的核心验证假设 |

对每一层，执行以下循环（最多 3 轮）：

#### Round 流程

**a) 调用 Proposer**

1. 读取模板文件 `${TEMPLATE_DIR}/proposer.md`
2. 将模板中的 `{{VARIABLE}}` 占位符替换为实际值（变量表见下方）
3. 将替换后的完整 prompt 写入 `${WORK_DIR}/prompts/proposer-L${LAYER}-R${ROUND}.txt`
4. 执行：

```bash
cat ${WORK_DIR}/prompts/proposer-L${LAYER}-R${ROUND}.txt | codex exec --full-auto --ephemeral --skip-git-repo-check -o ${WORK_DIR}/prompts/out-proposer-L${LAYER}-R${ROUND}.md -
```

5. 读取输出文件 `${WORK_DIR}/prompts/out-proposer-L${LAYER}-R${ROUND}.md`
6. 将完整输出内容追加到 `${WORK_DIR}/debate-log.md`，格式：

```markdown
---

## L{n}：{层级名称}

### Proposer L{n}-R{m}

{proposer 完整输出内容，原样粘贴，不要截断或引用}
```

**b) 调用 Reviewer**

1. 读取模板文件 `${TEMPLATE_DIR}/reviewer.md`
2. 替换占位符，写入 `${WORK_DIR}/prompts/reviewer-L${LAYER}-R${ROUND}.txt`
3. 执行：

```bash
cat ${WORK_DIR}/prompts/reviewer-L${LAYER}-R${ROUND}.txt | codex exec --full-auto --ephemeral --skip-git-repo-check -o ${WORK_DIR}/prompts/out-reviewer-L${LAYER}-R${ROUND}.md -
```

4. 读取输出文件 `${WORK_DIR}/prompts/out-reviewer-L${LAYER}-R${ROUND}.md`
5. 将完整输出内容追加到 `${WORK_DIR}/debate-log.md`，格式：

```markdown
### Reviewer L{n}-R{m}

{reviewer 完整输出内容，原样粘贴，不要截断或引用}
```

**c) Host 判断收敛**

1. **收敛** — 双方在核心结论上一致 → 提炼共识，进入下一层
2. **未收敛但有进展** — 存在实质性分歧但双方都在推进 → 将 Reviewer 反馈注入下一轮 Proposer prompt
3. **已达 3 轮** — 强制收敛，Host 提取最大公约数，标注未解决的分歧点

每层结束时，将共识追加到 debate-log.md（200 字以内），同时在对话中向用户展示：

```markdown
### Host L{n} 共识

{共识内容，200 字以内}

#### 未解决分歧（如有）
{分歧点及双方立场}
```

### Step 3: 回溯校验

L4 结束后：
- 将 L4 的分期策略与 L1 的核心价值主张对照
- 检查：MVP 是否覆盖了核心痛点？有没有在妥协中丢掉了不能让步的东西？
- 如果发现断裂，标注出来并给出修正建议

将回溯校验结果追加到 debate-log.md。

### Step 4: 汇编最终 PRD

将四层共识汇编为最终 PRD，写入 `${WORK_DIR}/final-prd.md`。

## 输出格式

最终 PRD 结构：

```markdown
# {产品名称} PRD

> 基于对抗式辩论生成，{日期}

## 1. 产品定位
### 1.1 目标用户
### 1.2 核心痛点
### 1.3 价值主张

## 2. 产品理想态
### 2.1 核心体验
### 2.2 关键功能
### 2.3 差异化优势

## 3. 现实约束与取舍
### 3.1 技术约束
### 3.2 资源约束
### 3.3 必须坚守的底线
### 3.4 可以妥协的维度

## 4. 分期策略
### 4.1 MVP 定义
### 4.2 核心验证假设
### 4.3 后续迭代路径

## 附录：辩论过程中的关键分歧与决策
{从辩论中提取的重要分歧点及最终决策理由}
```

## 边缘情况与回退

- 如果 codex exec 调用失败（超时、报错），重试一次；仍然失败则 Host 自己补位完成该角色的输出，并告知用户
- 如果找不到模板目录，提示用户检查是否在正确的目录下运行，或模板文件是否存在
- 如果双方都跑偏了，Host 有权重新定向讨论

## 模板变量表

模板文件使用 `{{VARIABLE}}` 双花括号占位符。Host 在每次调用前负责替换：

| 变量 | 说明 | 来源 |
|------|------|------|
| `{{TOPIC}}` | 用户输入的产品主题 | `$ARGUMENTS` |
| `{{LAYER}}` | 当前层级编号（1-4） | Host 控制 |
| `{{LAYER_NAME}}` | 层级名称 | Host 控制 |
| `{{KEY_QUESTIONS}}` | 本层关键问题列表 | Host 在 Step 1 设计 |
| `{{PREVIOUS_CONSENSUS}}` | 前序层级的共识摘要（L1 时留空） | Host 累积 |
| `{{REVIEWER_FEEDBACK}}` | Reviewer 上一轮反馈（R1 时留空） | Host 注入 |
| `{{PROPOSER_OUTPUT}}` | Proposer 本轮输出（仅 reviewer 模板） | Host 读取后注入 |

替换规则：
- 如果某个变量当前为空（如 L1 的 `{{PREVIOUS_CONSENSUS}}`），替换为空字符串，不要保留占位符
