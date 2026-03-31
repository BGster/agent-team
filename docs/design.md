# Agent Team

多 Agent 协作框架，支持 Pipeline / Parallel / Review 三种工作流模式，支持循环结构。

## 架构

```
┌───────────────────────────────────────────────────────┐
│                   Agent Team Framework                 │
│                                                       │
│  ┌─────────┐                                         │
│  │ Master  │ ← 协调者（调度任务、控制流程、循环管理）   │
│  └────┬────┘                                         │
│       ├──→ Analyzer（需求分析）                        │
│       ├──→ Architect（架构设计）                       │
│       ├──→ Coder（代码实现）                           │
│       ├──→ Tester（测试验证）                         │
│       └──→ Reviewer（评审）                           │
│                                                       │
│  ┌─────────┐                                         │
│  │ Memory  │ ← 共享记忆（各 Agent 读写）              │
│  └─────────┘                                         │
└───────────────────────────────────────────────────────┘
```

## 核心定位

- **独立框架**：不依赖 Project-Manager，可单独运行
- **可集成**：Project-Manager 可作为 Master 调度各角色 Agent
- **插件化**：各角色 Agent 可插拔、单独启用/禁用

## 角色职责

| 角色 | 职责 | 输出物 |
|------|------|--------|
| **Analyzer** | 理解需求、拆解任务、定义验收标准 | 需求文档 |
| **Architect** | 技术方案设计、技术选型、接口定义 | ADR 架构决策 |
| **Coder** | 实现功能、编写注释、遵守规范 | 代码文件 |
| **Tester** | 设计测试用例、执行测试、回归验证 | 测试报告 |
| **Reviewer** | 代码/方案评审、提出修改意见、质量把控 | 评审意见 |

## 工作流类型

### Pipeline（线性流程）

```
需求 → 架构 → 代码 → 测试 → 交付
  │      │      │      │
  ▼      ▼      ▼      ▼
 Doc    ADR   Code   Report
```

适用场景：简单任务、严格流程管控

### Pipeline + Loop（带循环的流程）

```
需求 → 架构 → 代码 → 测试 → 评审
  │      │      │      │      │
  │      │      │      │      ↓
  │      │      │      │    通过？ ──否→ 返回修改
  │      │      │      │      │
  │      │      │      │      ↓
  │      │      │      │     是
  │      │      │      │      ↓
  │      │      │      └──────→ 交付
```

**循环场景**：
- Tester 发现 Bug → 返回 Coder 修复 → 重新测试
- Reviewer 提出修改意见 → 返回 Coder/Architect 调整 → 重新评审
- Reviewer 不通过 → 循环直到通过或终止

**循环控制字段**：

```json
{
  "iteration": 1,
  "max_iterations": 3,
  "loop_back_to": "coder",
  "exit_condition": "reviewer.approved == true"
}
```

### Parallel（并行流程）

```
需求A → 架构A → 代码A → 测试A → 交付A
需求B → 架构B → 代码B → 测试B → 交付B
需求C → 架构C → 代码C → 测试C → 交付C
```

适用场景：批量任务、独立需求并行处理

### Parallel + Loop（并行带循环）

```
需求A → 架构A → 代码A → 测试A → 评审A ──→ 交付A
                      ↑      │
                      │      ↓
                      └──否──┘（循环修复）

需求B → 架构B → 代码B → 测试B → 评审B ──→ 交付B
                      ↑      │
                      │      ↓
                      └──否──┘（循环修复）
```

各任务独立循环，互不影响。

### Review（评审流程）

```
Master → 分发给多个 Agent 独立评审
       → 汇总各方意见
       → 输出评审结论
       → 评审不通过则打回重做
```

适用场景：重要决策、技术方案评审、代码审查

## 消息协议

各 Agent 之间通过结构化消息通信，支持两种传输方案。

### 消息格式

```json
{
  "id": "msg-001",
  "from": "analyzer",
  "to": "architect",
  "type": "task",
  "priority": "P1",
  "content": {
    "demand_id": "DMD-001",
    "summary": "用户需要导出报表功能",
    "acceptance_criteria": ["支持 CSV", "支持 Excel"]
  },
  "iteration": 1,
  "max_iterations": 3,
  "status": "pending",
  "created_at": "2026-03-31T13:00:00+08:00"
}
```

| type | 说明 |
|------|------|
| `task` | 任务分发 |
| `result` | 任务结果返回 |
| `review` | 评审请求 |
| `approve` | 审批通过 |
| `reject` | 审批拒绝（含修改意见） |
| `retry` | 重试请求（触发循环） |
| `sync` | 记忆同步 |

### 循环消息

```json
{
  "id": "msg-002",
  "from": "reviewer",
  "to": "coder",
  "type": "reject",
  "iteration": 1,
  "content": {
    "reason": "代码存在安全漏洞",
    "details": "未对输入进行消毒处理"
  }
}
```

---

### 方案一：SQLite 持久化队列

消息写入数据库，各 Agent 轮询处理，适合持久化和故障恢复。

```sql
CREATE TABLE messages (
    id            TEXT PRIMARY KEY,
    from_agent    TEXT NOT NULL,
    to_agent      TEXT NOT NULL,
    type          TEXT NOT NULL,
    priority      TEXT DEFAULT 'P1',
    iteration     INTEGER DEFAULT 1,
    max_iterations INTEGER DEFAULT 3,
    content       TEXT NOT NULL,
    status        TEXT DEFAULT 'pending',
    created_at    TEXT NOT NULL,
    updated_at    TEXT NOT NULL
);

CREATE INDEX idx_msg_to_status ON messages(to_agent, status);
```

**流程**：
1. Agent A 写入消息 `to_agent=B`
2. Agent B 轮询 `SELECT WHERE to_agent='B' AND status='pending'`
3. 处理完成后更新 `status='processed'`
4. 若需要循环，发送 `type=reject` 并 `iteration++`

---

### 方案二：OpenClaw sessions_send（实时通知）

利用 OpenClaw 的 session 机制做实时消息分发。

```python
# Agent A 向 Agent B 发送实时消息
sessions_send(sessionKey_B, message)

# Agent B 监听自己的收件箱 session
```

**特点**：
- 实时性高（无轮询延迟）
- 依赖 session 存活状态
- 可与 SQLite 方案互补（实时用 session，持久化用 DB）

## 状态流转

### 需求状态（DMD）

```
draft → approved → in_progress → reviewed → verified → delivered
```

| 状态 | 说明 |
|------|------|
| draft | 草稿（Analyzer 输出） |
| approved | 已审批（Master 确认） |
| in_progress | 进行中（Architect/Coder/Teste/Reviewer 处理中） |
| reviewed | 已评审（Reviewer 通过） |
| verified | 已验证（Tester 通过） |
| delivered | 已交付 |

### 质量门禁

| 阶段 | 门禁条件 |
|------|----------|
| 需求 → 架构 | DMD 状态为 `approved` |
| 架构 → 代码 | ADR 存在且被 `approved` |
| 代码 → 测试 | 代码已 commit / PR 已 open |
| 测试 → 评审 | 测试报告已生成 |
| 评审 → 验证 | Reviewer 输出 `approve` |
| 验证 → 交付 | 循环次数未超上限，或 Reviewer 最终通过 |

### 循环控制

| 字段 | 说明 |
|------|------|
| `iteration` | 当前循环次数 |
| `max_iterations` | 最大循环次数（默认 3） |
| `loop_back_to` | 循环返回的角色 |
| `exit_condition` | 退出条件（如 `reviewer.approved == true`） |

**超限处理**：达到 `max_iterations` 后，Master 介入决策（人工干预或终止任务）。

## 存储设计

各 Agent 独立记忆，支持共享：

```
agent-memory/
├── analyzer/      # 需求分析记录
├── architect/     # 架构决策记录
├── coder/         # 代码片段、实现笔记
├── tester/        # 测试用例、测试结果
└── reviewer/      # 评审意见、评审记录
```

## 与 Project-Manager 集成

Project-Manager 可作为 Master 使用 Agent Team：

```
Project-Manager（Master）
  ├─ demands/     ← Analyzer 输出
  ├─ decisions/   ← Architect 输出
  ├─ issues/      ← Tester/Reviewer 发现的问题
  ├─ reviews/     ← Reviewer 评审记录
  └─ daily/      ← 各角色工作日志
```

Agent Team 读取 Project-Manager 的记忆，输出也写入 Project-Manager。
