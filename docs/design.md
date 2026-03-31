# Agent Team

多 Agent 协作框架，支持 Pipeline / Parallel / Review 三种工作流模式。

## 架构

```
┌─────────────────────────────────────────────┐
│              Agent Team Framework            │
│                                             │
│  ┌─────────┐                               │
│  │ Master  │ ← 协调者（调度任务、控制流程）   │
│  └────┬────┘                               │
│       ├──→ Analyzer（需求分析）              │
│       ├──→ Architect（架构设计）             │
│       ├──→ Coder（代码实现）                │
│       └──→ Tester（测试验证）               │
│                                             │
│  ┌─────────┐                               │
│  │ Memory  │ ← 共享记忆（各 Agent 读写）     │
│  └─────────┘                               │
└─────────────────────────────────────────────┘
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

## 工作流类型

### Pipeline（线性流程）

```
需求 → 架构 → 代码 → 测试 → 交付
  │      │      │      │
  ▼      ▼      ▼      ▼
 Doc    ADR   Code   Report
```

适用场景：串行任务、严格流程管控

### Parallel（并行流程）

```
需求A → 架构A → 代码A → 测试A → 交付A
需求B → 架构B → 代码B → 测试B → 交付B
需求C → 架构C → 代码C → 测试C → 交付C
```

适用场景：批量任务、独立需求并行处理

### Review（评审流程）

```
Master → 分发给多个 Agent 独立评审
       → 汇总各方意见
       → 输出评审结论
```

适用场景：重要决策、技术方案评审

## 消息协议

各 Agent 之间通过结构化消息通信：

```json
{
  "from": "analyzer",
  "to": "architect",
  "type": "task",
  "content": {
    "demand_id": "DMD-001",
    "summary": "用户需要导出报表功能",
    "acceptance_criteria": ["支持 CSV", "支持 Excel"]
  }
}
```

| type | 说明 |
|------|------|
| `task` | 任务分发 |
| `result` | 任务结果返回 |
| `review` | 评审请求 |
| `approve` | 审批通过 |
| `reject` | 审批拒绝 |
| `sync` | 记忆同步 |

## 状态流转

### 需求状态（DMD）

```
draft → approved → in_progress → verified → delivered
```

| 状态 | 说明 |
|------|------|
| draft | 草稿（Analyzer 输出） |
| approved | 已审批（Master 确认） |
| in_progress | 进行中（Architect/Coder/Tester 处理中） |
| verified | 已验证（Tester 通过） |
| delivered | 已交付 |

### 质量门禁

| 阶段 | 门禁条件 |
|------|----------|
| 需求 → 架构 | DMD 状态为 `approved` |
| 架构 → 代码 | ADR 存在且被 `approved` |
| 代码 → 测试 | 代码已 commit / PR 已 open |
| 测试 → 交付 | 测试报告 100% 通过 |

## 存储设计

各 Agent 独立记忆，支持共享：

```
agent-memory/
├── analyzer/      # 需求分析记录
├── architect/     # 架构决策记录
├── coder/         # 代码片段、实现笔记
└── tester/        # 测试用例、测试结果
```

## 与 Project-Manager 集成

Project-Manager 可作为 Master 使用 Agent Team：

```
Project-Manager（Master）
  ├─ demands/     ← Analyzer 输出
  ├─ decisions/   ← Architect 输出
  ├─ issues/      ← Tester 发现的问题
  └─ daily/       ← 各角色工作日志
```

Agent Team 读取 Project-Manager 的记忆，输出也写入 Project-Manager。
