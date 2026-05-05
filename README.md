# prd-debate

通过 `Proposer` vs `Reviewer` 的对抗式辩论，逐层推进产出高质量 PRD。

这个 skill 适合用于从模糊产品想法出发，生成结构化、可落地、带取舍逻辑的需求文档，避免单 agent 直接写 PRD 时常见的浅层展开和自洽性不足。

## 核心机制

仓库中的 `prd-debate` skill 采用三方协作：

- `Host`：负责定义问题、控制辩论流程、判断收敛、提炼共识
- `Proposer`：提出方案、理想态和推进路径
- `Reviewer`：持续提出约束、质疑和反例，逼近真实可行方案

整个流程固定为四层：

1. 产品定位
2. 理想态
3. 差距分析
4. 分期策略

每层最多进行 3 轮辩论，最后由 Host 汇总形成 PRD。

## 仓库内容

- `prompt.md`：主 skill 入口
- `templates/proposer.md`：Proposer 子 agent 模板
- `templates/reviewer.md`：Reviewer 子 agent 模板
- `skill.json`：skill 元信息
- `CLAUDE.md`：补充说明

## 安装

将这个仓库中的 `prompt.md` 注册为 Claude Code slash command：

```bash
mkdir -p .claude/commands
cp prompt.md .claude/commands/prd-debate.md
```

如果你是从其他目录引用这个 skill，确保 `templates/` 目录也一并保留，因为运行时会读取对应模板文件。

## 依赖

- Claude Code
- `codex` CLI，可通过 `codex exec` 调用 sub-agent

如果本机还没有安装 Codex CLI，可按你的环境自行安装并确保 `codex` 命令可用。

## 使用方式

在 Claude Code 中直接调用：

```text
/prd-debate 一个面向独居年轻人的 AI 陪伴宠物硬件产品
```

你也可以把任意产品方向、功能想法、业务命题作为输入主题。

## 运行产物

每次运行都会在当前工作目录下创建一个独立会话目录：

```text
prd-debate-{session_id}/
```

其中包含：

- `debate-log.md`：完整辩论记录
- `final-prd.md`：最终整理后的 PRD
- 各轮 `prompt`、`proposer`、`reviewer` 中间文件

## 适用场景

- 从 0 到 1 定义新产品
- 对模糊需求做结构化澄清
- 在理想体验与现实约束之间寻找可执行方案
- 为 MVP 范围和迭代路线建立更清晰的推导过程

## 版本

当前 `skill.json` 版本为 `0.1.0`。
