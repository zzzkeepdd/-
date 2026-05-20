# Hermes + Codex 协作工作流 v1.4

> v1.3 → v1.4 核心变更：
> 1. 新增「前端开发」子Agent（独立负责 UI 层，与 AI程序员边界清晰）
> 2. 验收不通过由开发统筹按问题类型分发到对应子Agent
> 3. 砍掉 GLM 视觉验收——用户侧 UI 验收改为 Codex GPT-5.5 内部子Agent执行
> 4. Reviewer cronjob 降级为纯读报告，不再做视觉分析

---

## 一、总架构

| 组件 | 模型 | 角色定位 |
|---|---|---|
| Hermes | DeepSeek V4 | 大脑：需求澄清、文档生成、cronjob 调度、状态机 |
| Codex | GPT-5.5 | 双手：开发 + 三层验收（代码技术 / 功能业务 / 用户侧UI） |

核心原则：

- Hermes 通过 **cronjob** 做定时轮询，不是常驻 daemon。
- Codex 不做 while-true 循环，每次只执行一个明确任务。
- Codex 内部 spawn subagents，Hermes 不可直接交互。
- 所有跨系统状态落盘到共享目录。

推荐链路：

```text
用户 -> Hermes 需求澄清/定稿
  -> Hermes 写 task 文件
  -> cronjob-dispatcher 检测到新任务
  -> cronjob-dispatcher 调用 codex exec 下发
  -> Codex 开发 + 三层验收 + 写回结果
  -> cronjob-reviewer 检测到 done 结果
  -> cronjob-reviewer 读报告，通知用户
```

---

## 二、Hermes 侧角色

### 1. 计划 Agent
- 唯一用户入口。
- 负责需求澄清、范围控制、优先级排序、任务拆分。
- 用户未确认"定稿"前，不写入项目任务目录。
- 定稿后生成商议结论、需求规格、验收标准和防坑指南。

### 2. 文档生成 Agent
- 根据商议结论生成：
  - `task-{id}_spec.md`
  - `task-{id}_acceptance.md`
  - `task-{id}_pitfalls.md`

### 3. Codex 子 Agent 设计 Agent
- 生成 Codex 侧全部提示词模板。
- 六个角色：开发统筹、AI程序员、**前端开发**、开发执行、代码技术验收、功能业务验收、用户侧UI验收。

### 4. 审核 Agent
- 审核需求文档、验收标准、任务边界、防坑指南。
- 确认无冲突后写入 `hermes_outbox/`。

### 5. Hermes 统筹 Agent（调度逻辑）
- 触发 cronjob 注册（一次性）。
- 读取 Codex 写回的 result/status/report。
- 管理失败重试、超时、人工介入和最终汇报。
- **不是常驻进程**——调度逻辑分布在 cronjob 和按需触发中完成。

---

## 三、Codex 侧角色

### 1. 开发统筹（主 Agent）
- 由 `codex exec` 触发启动。
- 读取任务文件、规范文档、提示词。
- 分析任务类型，按需 spawn 子Agent。
- **自己不写代码**，只做调度和验收结果分发。

### 2. AI 程序员 subagent
- 适用场景：复杂模块、需独立架构判断、后端逻辑。
- 输出代码变更、技术决策、风险提示、工程交接单。
- 不碰 UI 层（CSS/组件/样式）。

### 3. 前端开发 subagent（v1.4 新增）
- 适用场景：**任务涉及 UI 层时必启用**。
- 只管：组件结构、HTML/CSS、响应式布局、交互行为、可访问性、样式系统。
- **绝对不碰**：业务逻辑、API 调用、数据模型。
- 与 AI程序员边界：前端开发做 UI 壳，AI程序员做数据/逻辑核。

### 4. 开发执行 subagent
- 适用场景：明确、局部、低歧义的简单任务。
- 按指令完成指定文件或模块，不做技术决策。

### 5. 代码技术验收 subagent
- 检查：语法、运行报错、接口连通性、逻辑漏洞、运行稳定性。
- 输出结构化技术验收报告。
- 这是 Codex 内部自检中最严格的一环。

### 6. 功能业务验收 subagent
- 对照需求规格和验收标准，逐项检查功能完整性。
- 输出结构化业务验收报告。

### 7. 用户侧 UI 验收 subagent
- 浏览器打开页面 → 截图 → 视觉分析。
- 模拟用户视角检查：功能是否可用、入口是否可找到、布局是否合理、体验是否合格。
- 注意：这是"用户侧体验底线"而非像素级 QA——颜色差一点、间距差一点放过；白屏、乱码、按钮不触发必卡。

### Codex 内部 subagent 重要限制
- Hermes 不能直接与 Codex 内部 subagent 交互。
- Hermes 只能通过 Codex 主任务入口（codex exec）间接影响。

---

## 四、验收链与问题分发

### 验收链（固定顺序，不通过则从头重走）
```
代码技术验收 → 功能业务验收 → 用户侧 UI 验收
```

### 验收结果分发（开发统筹执行）

#### 技术验收不通过
读验收报告 → 判断错误类型：
- 语法/运行时错误 → 驳回给**原开发**子Agent
- 接口连通性错误 → 驳回给 AI程序员
- 逻辑漏洞 → 驳回给 AI程序员

#### 功能业务验收不通过
读验收报告 → 判断缺失功能归属：
- 后端功能缺失 → 驳回给 AI程序员
- 前端功能缺失 → 驳回给 前端开发
- 前后端都有问题 → 分别驳回

#### 用户侧 UI 验收不通过
读验收报告 → 判断问题归属：
- 样式问题（配色/布局/响应式）→ 驳回给 前端开发
- 交互问题（按钮不触发）→ 判断是前端交互层还是业务层
  - 前端交互层 → 驳回给 前端开发
  - 业务层（数据处理错误导致 UI 异常）→ 驳回给 AI程序员
- 可访问性问题 → 驳回给 前端开发

### 整改回流流程
1. 读验收报告，定位责任人
2. 将验收报告 + 该角色的工程交接单注入对应子Agent
3. 子Agent 针对性修改，更新交接单
4. 修改完成后 **从第一级代码技术验收重新走完整验收链**
5. 同类问题连续整改满 5 次仍未通过 → 换全新实现思路
6. 换思路后再 5 次仍失败 → 写《已尝试方案及失败原因报告》，写回 codex_inbox/，status = blocked

### 验收尺度
| 层级 | 严格度 | 卡什么 | 放什么 |
|------|--------|--------|--------|
| 代码技术验收 | 严格 | 语法错、白屏、运行时崩溃、接口不通 | — |
| 功能业务验收 | 中等 | 核心功能缺失（如没写删除按钮） | 边缘情况 |
| 用户侧UI验收 | 用户底线 | 白屏、乱码、按钮不触发、布局崩溃 | 颜色偏差、间距偏差、没动画 |

---

## 五、监听与触发层（cronjob 双引擎）

### cronjob 1: hermes-dispatcher
| 项目 | 说明 |
|---|---|
| 频率 | 每 1 分钟（Hermes cronjob 最小粒度） |
| 模式 | agent（需做文件校验和 codex exec 调用） |
| 触发条件 | `hermes_outbox/` 中有 `task-{id}.json` 且状态为 pending |

执行逻辑：
```text
1. 获取锁文件（hermes_outbox/.dispatch.lock）
   - 锁已被持有 → 跳过本次
2. 扫描 hermes_outbox/task-*.json
   - 跳过已在 task_registry.json 中标记为 dispatched/done/failed 的任务
3. 对每个 pending 任务：
   a. 校验文件完整性（spec + acceptance + pitfalls 都存在）
   b. 校验不通过 → 标记 task 为 invalid，写错误日志
   c. 校验通过 → 更新 status.json 为 queued
   d. 调用 codex exec，注入 task 文件路径
   e. 在 task_registry.json 中标记 dispatched + 时间戳
   f. 更新 status.json 为 running
4. 释放锁文件
```

### cronjob 2: hermes-reviewer
| 项目 | 说明 |
|---|---|
| 频率 | 每 1 分钟 |
| 模式 | agent |
| 触发条件 | `codex_inbox/` 中有 `task-{id}.result.json` 且 status 为 done |

执行逻辑：
```text
1. 获取锁文件（codex_inbox/.review.lock）
2. 扫描 codex_inbox/task-*.result.json
   - 跳过已在 review_registry.json 中标记为 reviewed 的任务
3. 对每个 done 任务：
   a. 读取 result.json → 检查三层验收是否全部通过
   b. 全部通过 → 标记 reviewed，通知用户"通过"
   c. 任一不通过 → 检查重试次数
      - 未达上限 → 生成修复任务写回 hermes_outbox/
      - 已达上限 → 标记 blocked，通知用户
4. 释放锁文件
```

注意：v1.4 的 reviewer 不做视觉分析——UI 验收已由 Codex 内部 GPT-5.5 完成，reviewer 只读报告。

---

## 六、共享目录结构

```text
项目根目录/.hermes-codex-protocol/
├── hermes_outbox/
│   ├── .dispatch.lock
│   ├── task-{id}.json
│   ├── task-{id}_spec.md
│   ├── task-{id}_acceptance.md
│   ├── task-{id}_pitfalls.md
│   └── prompts/
│       ├── 开发统筹.md
│       ├── AI程序员.md
│       ├── 前端开发.md          ← v1.4 新增
│       ├── 开发执行.md
│       ├── 代码技术验收.md
│       ├── 功能业务验收.md
│       └── 用户侧UI验收.md
├── codex_inbox/
│   ├── .review.lock
│   ├── task-{id}.result.json
│   ├── reports/
│   │   ├── 代码技术验收.md
│   │   ├── 功能业务验收.md
│   │   └── 用户侧UI验收.md
│   ├── 交接日志/
│   │   └── task-{id}_handoff.md
│   └── 成品/
├── status.json
├── task_registry.json
└── review_registry.json
```

---

## 七、文件格式

### `task-{id}.json`
```json
{
  "task_id": "task-001",
  "status": "pending",
  "goal": "一句话任务目标",
  "project_root": "C:/path/to/project",
  "spec_file": "task-001_spec.md",
  "acceptance_file": "task-001_acceptance.md",
  "pitfalls_file": "task-001_pitfalls.md",
  "prompts_dir": "prompts/",
  "acceptance": ["验收项1", "验收项2"],
  "max_retries": 3,
  "retry_count": 0,
  "timeout_minutes": 30,
  "created_at": "2026-05-20T10:00:00+08:00"
}
```

### `task-{id}.result.json`
```json
{
  "task_id": "task-001",
  "status": "done",
  "summary": "一句话总结",
  "artifacts": {
    "code_dir": "C:/path/to/project",
    "tech_review": "codex_inbox/reports/代码技术验收.md",
    "business_review": "codex_inbox/reports/功能业务验收.md",
    "ui_review": "codex_inbox/reports/用户侧UI验收.md",
    "handoff": "codex_inbox/交接日志/task-001_handoff.md"
  },
  "tech_review_passed": true,
  "business_review_passed": true,
  "ui_review_passed": true,
  "risks": [],
  "completed_at": "2026-05-20T10:30:00+08:00"
}
```

### `status.json`
```json
{
  "current_task": "task-001",
  "phase": "idle",
  "subagents_active": 0,
  "blocked_reason": "",
  "updated_at": "2026-05-20T10:30:00+08:00"
}
```

---

## 八、状态机流转

```
任务生命周期状态：

  pending ──→ queued ──→ running ──→ done ──→ reviewed
    │                      │          │
    └──→ invalid           └──→ failed └──→ blocked

状态转换触发条件：

  pending   : Hermes 写入 task-{id}.json 时初始状态
  queued    : dispatcher 校验通过，准备下发
  running   : dispatcher 调用 codex exec 后
  done      : Codex 写完 result.json
  reviewed  : reviewer 验收通过
  failed    : 重试次数超限且结果不通过
  blocked   : 外部故障或整改超限，需人工介入
  invalid   : 任务文件不完整或格式错误
```

---

## 九、超时与重试

### 超时处理
```text
cronjob-reviewer 同时检查超时：
  1. 扫描 task_registry.json，找 dispatched 但超时的任务
  2. 检查 status.json 中 phase 是否为 running 且超过 task.timeout_minutes
  3. 超时 → 更新 status 为 failed，写 error_log
  4. 未超时 → 跳过
```

### 重试策略
```text
修复任务生成逻辑（reviewer）：
  1. result.json 任一验收不通过
  2. retry_count < max_retries
  3. → 生成 task-{id}-fix-{N}.json 写入 hermes_outbox/
  4. → 更新原任务 retry_count += 1
  5. retry_count >= max_retries → 标记 blocked，通知用户
```

---

## 十、完整协作链路

```text
用户只说一次需求
  -> Hermes 计划 Agent 澄清需求
  -> 用户确认定稿
  -> Hermes 生成规格、验收标准、防坑指南、提示词（含 前端开发.md）
  -> Hermes 写入 hermes_outbox/（task 初始状态: pending）
  -> cronjob-dispatcher 每分钟扫描，检测到 pending 任务
  -> 获取锁 → 校验文件 → 标记 queued → codex exec 下发 → 标记 running → 释放锁
  -> Codex 开发统筹 读取任务，判断任务类型
     -> 有 UI → spawn 前端开发
     -> 有后端逻辑 → spawn AI程序员
     -> 简单任务 → spawn 开发执行
  -> Codex 内部：开发 → 代码技术验收
     -> 不通过 → 开发统筹分发到对应子Agent → 修改 → 重走验收链
  -> Codex 内部：功能业务验收
     -> 不通过 → 开发统筹分发到对应子Agent → 修改 → 从代码验收重走
  -> Codex 内部：用户侧 UI 验收（GPT-5.5 视觉）
     -> 不通过 → 开发统筹判断是前端还是后端问题 → 分发 → 修改 → 重走
  -> 全部通过 → 写回 codex_inbox/（result.json + 三份验收报告 + 交接单 + 成品）
  -> Codex 更新 status.json → done
  -> cronjob-reviewer 每分钟扫描，检测到 done 结果
  -> 获取锁 → 读 result → 三层验收全部通过
     -> 通过：标记 reviewed → 通知用户"task-001 验收通过"
     -> 不通过且 retry < max：生成修复任务 → 回流
     -> 不通过且 retry >= max：标记 blocked → 通知用户
  -> 全部任务完成 → 汇报用户项目完成
```

---

## 十一、任务循环与终点

```text
task-001
  -> dispatcher 下发
  -> Codex 开发统筹 → AI程序员 + 前端开发（按需）
  -> 代码验收 → 业务验收 → UI验收
  -> 通过 → reviewer 通知 → task-002
  -> 不通过 → 开发统筹按问题类型分发
              → 前端问题 → 前端开发修改
              → 后端问题 → AI程序员修改
              → 改完重走三层验收
  -> 超限 → blocked → 通知用户
```

终点：
- Codex 内部三层验收全部通过 ≠ 最终终点。
- reviewer 确认并通知用户 = 单任务终点。
- 用户确认项目完成 = 最终终点。

---

## 十二、断点续跑

断点续跑基于文件事实，不依赖会话记忆。

Codex 每次启动时读取：
- `status.json`
- 当前 `task-{id}.json`
- 历史 `task-{id}.result.json`
- `交接日志/`
- 当前 Git 状态和测试结果

Hermes cronjob 每次运行时检查：
- `task_registry.json` → 任务是否已下发
- `review_registry.json` → 任务是否已验收
- `status.json` → 当前阶段
- 锁文件超时 → 是否有僵尸锁

---

## 十三、工程交接单

每个子Agent 完成任务后必须写交接单，归档到 `codex_inbox/交接日志/`。

内容控制在 300-500 tokens：
- 修改了哪些文件
- 核心技术决策
- 组件树 / 样式变量表（前端开发专用）
- 已知风险
- 后续任务建议

---

## 十四、Token 成本控制

原则：
- Hermes 负责需求、计划、拆分、状态机，使用低成本模型（D4）。
- Codex 只处理明确、可执行的工程任务（5.5）。
- 前端开发和 AI程序员按任务类型启用，避免不必要的 spawn。
- 每轮 Codex 启动前注入最小必要上下文。

避免：
- 把全部项目历史塞给 Codex。
- 让 Codex 长期空转等待任务。
- 后端任务 spawn 前端开发子Agent（纯后端不需要）。

---

## 十五、Codex 侧子Agent 任务分发规则

开发统筹在读取任务后执行以下判断：

```
1. 读需求规格说明书，识别涉及哪些层
2. 纯后端任务（API、数据库、脚本）→ spawn AI程序员 或 开发执行
3. 纯前端任务（页面、组件、样式）→ spawn 前端开发
4. 全栈任务（前后端都有）→ 分别 spawn AI程序员 + 前端开发

并行策略：
- 前后端无依赖 → 并行 spawn
- 前端依赖后端 API 定义 → 先 AI程序员产出接口定义，再前端开发接入
- 多个独立前端页面 → 并行 spawn 多个前端开发
```

---

## 十六、提示词模板复用

Hermes 侧「子Agent设计」生成的提示词分两类：
- 模板（首次项目生成）：六个角色的完整提示词，存入 `prompts/` 作为模板库。
- 差异指令（后续项目）：只输出与当前项目的差异（技术栈、风格要求、特殊约束），由 Codex 开发统筹用模板 + 差异拼装最终提示词。

---

## 最终推荐方案

```text
Hermes cronjob 负责调度、监听、重试、状态机。
Codex exec 负责一次性高质量工程执行（含三层内部验收）。
共享文件夹 + 锁文件 负责跨系统状态同步。
前端开发子Agent 独立负责 UI 层，与 AI程序员边界清晰。
验收不通过由开发统筹按问题类型分发到对应子Agent。
```

分阶段落地：
```text
Phase 1：Hermes cronjob + codex exec + 文件协议 + 锁文件 + 前端开发
Phase 2：Hermes + Codex SDK（提升集成深度）
Phase 3：Hermes + Codex App Server + Dashboard（全可视化）
```

---

v1.4 | 2026-05-20
| 核心变更：新增前端开发子Agent、验收分发逻辑、Codex内部UI验收替代GLM、Reviewer降级
