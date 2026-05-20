---
name: hermes-codex-workflow
description: Hermes + Codex 全自动协作开发工作流。Hermes(V4)商议需求定稿，cronjob自动派发Codex(GPT-5.5)开发+三层自检验收，全程用户只在一开始说需求。
version: 1.4
tags: [workflow, codex, collaboration, automation]
---

# Hermes + Codex 全自动协作工作流

## 架构总览

```
用户说需求 → Hermes(V4)商议定稿 → 写入共享目录
  → cronjob(dispatcher)每分钟扫描 → codex exec 派发
  → Codex(GPT-5.5): 开发 → 代码技术验收 → 功能业务验收 → 用户侧UI验收
  → 写回 codex_inbox/
  → cronjob(reviewer)每分钟扫描 → 读报告 → 验收通过/不通过
```

## 共享目录

项目根目录下 `.hermes-codex-protocol/`：

```
hermes_outbox/          ← Hermes写，Codex读
  task-{id}.json        ← 任务描述（主文件）
  task-{id}_spec.md     ← 需求规格说明书
  task-{id}_acceptance.md ← 验收标准
  task-{id}_pitfalls.md ← 代码实现防坑指南（可选）
  prompts/              ← Codex子Agent提示词
codex_inbox/            ← Codex写，Hermes读
  task-{id}.result.json ← 完成结果
  reports/              ← 验收报告
  交接日志/             ← 工程交接单
  成品/                 ← 交付代码
status.json             ← Codex实时状态
task_registry.json      ← Hermes派发登记
review_registry.json    ← Hermes验收登记
```

## 文件格式

### task-{id}.json (Hermes → Codex)

```json
{
  "task_id": "task-001",
  "goal": "一句话任务目标",
  "spec_file": "task-001_spec.md",
  "acceptance_file": "task-001_acceptance.md",
  "pitfalls_file": "task-001_pitfalls.md",
  "prompts_dir": "prompts/",
  "acceptance": ["验收项1", "验收项2"],
  "created_at": "ISO时间戳"
}
```

### task-{id}.result.json (Codex → Hermes)

```json
{
  "task_id": "task-001",
  "status": "success | failed",
  "tech_review_passed": true,
  "business_review_passed": true,
  "ui_review_passed": true,
  "risks": [],
  "completed_at": "ISO时间戳"
}
```

### status.json (Codex实时更新)

```json
{
  "current_task": "task-001",
  "phase": "executing | testing | reviewing | done | idle",
  "updated_at": "ISO时间戳"
}
```

## 任务循环

1. 用户在Hermes说需求，商议后定稿
2. Hermes写 hermes_outbox/（task.json + spec + 验收标准 + 防坑指南 + 提示词）
3. dispatcher cronjob 每分钟扫描 outbox → 发现新任务 → codex exec 下发
4. Codex 开发 → 代码技术验收 → 功能业务验收 → 用户侧UI验收
5. 全部通过 → 写 codex_inbox/ → status.json = done
6. reviewer cronjob 每分钟扫描 inbox → 读报告 → 验收

## Codex 侧角色（共6个）

| 角色 | 类型 | 职责 |
|------|------|------|
| 开发统筹 | 主Codex常驻 | 读任务→拆分→spawn→收结果→调度验收→驳回分发 |
| AI程序员 | spawn子Agent | 后端/复杂逻辑、API、数据层 |
| 前端开发 | spawn子Agent | UI层：组件结构、HTML/CSS、响应式、交互、可访问性 |
| 开发执行 | spawn子Agent | 简单标准化任务 |
| 代码技术验收 | spawn子Agent | 语法/运行/接口/稳定性 |
| 功能业务验收 | spawn子Agent | 对照需求逐项检查功能完整性 |
| 用户侧UI验收 | spawn子Agent | GPT-5.5视觉：浏览器交互测试、体验评估 |

## 三层验收定义

| 层级 | 执行者 | 严格度 | 卡什么 |
|------|--------|--------|--------|
| 代码技术验收 | Codex子Agent | 严格 | 语法错误、白屏、运行时崩溃、控制台报错 |
| 功能业务验收 | Codex子Agent | 中等 | 核心功能缺失（如没写删除按钮） |
| 用户侧UI验收 | Codex子Agent(GPT-5.5视觉) | 用户侧 | 页面能用、不丑、交互顺——不卡像素级 |

## cronjob 配置

### Dispatcher (`70672abe810f`)
- 频率：每分钟
- 模式：no_agent（纯Python脚本）
- 脚本：`hermes-codex-workflow/scripts/dispatcher.py`
- 职责：扫描outbox → 取锁 → 校验完整性 → codex exec下发 → 写registry

### Reviewer (`6e008e391a2b`)
- 频率：每分钟
- 模式：no_agent（纯Python脚本）
- 脚本：`hermes-codex-workflow/scripts/reviewer.py`
- 职责：扫描inbox → 读result.json + 验收报告 → 验收 → 通知用户

## 脚本位置

脚本位于 `~/AppData/Local/hermes/scripts/hermes-codex-workflow/scripts/`：

- `dispatcher.py` — 派发引擎
- `reviewer.py` — 验收引擎
- `init_project.py` — 初始化项目目录

## 已知限制

1. cronjob最小粒度1分钟（非30秒），不影响功能
2. GLM子Agent自动验收不可行（delegate_task不支持指定模型，cronjob model override不生效）
3. UI验收改为Codex内部GPT-5.5视觉能力处理，已脱离GLM依赖
4. codex exec需要 `--sandbox workspace-write` 才能写入文件
5. codex是npm shim，dispatcher脚本须用 `shell=True`

## 初始化新项目

```bash
python hermes-codex-workflow/scripts/init_project.py <项目路径> <项目名>
```

## 工作流触发

用户在Hermes主会话中说需求 → Hermes商议定稿 → 写outbox → cronjob自动接管

## 验收失败处理

**Codex内部自检（用户无感知）：**

验收链不通过 → 开发统筹读验收报告定位责任：
- 样式/布局/可访问性/响应式问题 → 驳回给前端开发子Agent
- 接口/逻辑/数据层问题 → 驳回给AI程序员子Agent
- 简单问题 → 驳回给开发执行子Agent

子Agent整改后，从头重走：代码技术验收 → 功能业务验收 → 用户侧UI验收

**最终用户反馈（需用户触发）：**

用户在Hermes说"UI太丑"或"XX功能不触发"
→ Hermes写修正需求到outbox
→ 开发统筹读修正需求，重新派发给对应子Agent

**熔断机制：**
- 同类问题连续5次失败 → Codex换思路重试
- 换思路后再5次失败 → 上报用户，暂停等待指令