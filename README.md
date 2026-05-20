# Hermes + Codex 协作工作流

Hermes（DeepSeek V4）作为大脑负责需求分析与任务编排，Codex（GPT-5.5）作为双手负责代码实现与质量验收的**全自动多 Agent 协作框架**。

## 核心理念

需求与实现彻底分离。Hermes 产出规范，Codex 执行规范，不交叉。

## 架构总览

```
用户 → Hermes(V4) 商议定稿
  → cronjob 自动派发 Codex
  → Codex(GPT-5.5): 开发 → 代码验收 → 业务验收 → UI验收
  → cronjob 读报告 → 验收结论推送
```

## 角色分工

### Hermes 侧（DeepSeek V4）
| 角色 | 职责 |
|------|------|
| 计划 Agent | 唯一对外接口，商议需求、定稿 |
| 文档生成 Sub-Agent | 产出需求规格说明书 + 验收标准 |
| 子 Agent 设计 Sub-Agent | 产出 Codex 侧子 Agent 提示词 |
| 审核 Agent | 文档完整性、一致性检查 + 代码预审 |
| Dispatcher cronjob | 每分钟扫描 outbox，派发 Codex |
| Reviewer cronjob | 每分钟扫描 inbox，读报告验收 |

### Codex 侧（GPT-5.5）
| 角色 | 职责 |
|------|------|
| 开发统筹 | 接收规范，调度子 Agent |
| AI 程序员 | 复杂逻辑、后端开发 |
| 前端开发 | UI 层：组件、样式、响应式、交互 |
| 开发执行 | 简单标准化任务 |
| 代码技术验收 | 语法、运行、接口、稳定性 |
| 功能业务验收 | 对照需求逐项检查 |
| 用户侧 UI 验收 | 模拟用户视角检查界面与体验 |

## 共享目录结构

```
.hermes-codex-protocol/
├── hermes_outbox/          ← Hermes 写，Codex 读
│   ├── task-{id}.json
│   ├── task-{id}_spec.md
│   ├── task-{id}_acceptance.md
│   ├── task-{id}_pitfalls.md
│   └── prompts/            ← 6 个子 Agent 提示词
├── codex_inbox/            ← Codex 写，Hermes 读
│   ├── task-{id}.result.json
│   ├── reports/
│   └── 交接日志/
└── status.json             ← 实时状态
```

## 验收链

```
代码技术验收 → 功能业务验收 → 用户侧UI验收
    ↓ 不通过           ↓ 不通过         ↓ 不通过
    驳回修正 → 从头重走全部三层
```

不含 GLM 依赖——UI 验收由 Codex GPT-5.5 内部子 Agent 自主完成。

## 用户操作

```
1. Hermes 窗口说需求 → 商议定稿
2. 等通知（全自动）
3. 看验收结论
```

全程只参与第一步，其余由 cronjob + Codex 自动完成。

## 文件说明

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Hermes 技能文件，加载即用 |
| `hermes-codex-workflow-v1.4.md` | 完整协议文档 |
| `prompts/` | Codex 子 Agent 提示词模板 |

## 版本

v1.4 — 新增前端开发子 Agent，验收分发机制，砍掉 GLM 依赖
