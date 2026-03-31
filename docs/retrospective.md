# Agent Team 阶段性回顾 — Project-Manager 开发

**项目**：Project-Manager（CLI + Skill）  
**时间**：2026-03-31  
**状态**：impl 分支代码评审完成，发现 3 个阻塞项待修复

---

## 一、项目概述

| 项目 | 说明 |
|------|------|
| **目标** | 开发 Project-Manager CLI + Skill，实现项目记忆管理、向量检索、用户隔离 |
| **技术栈** | Python + Typer + Rich + SQLite + sqlite-vec |
| **分支** | master（设计）/ impl（实现） |
| **总耗时** | ~2 小时（Analyzer + Architect + Coder + Tester + Reviewer 并行） |

---

## 二、Agent 协作流程

### 2.1 工作流

```
设计阶段（master 分支）
  └─ Zeki 提出需求 → 多次迭代设计 → 最终方案

实现阶段（impl 分支，并行）
  ├─ Analyzer ───→ tech-spec.md（CLI 命令规格、文件模板）
  ├─ Architect ──→ adr-001-technical-architecture.md（DB schema、嵌入模型）
  ├─ Coder ──────→ 完整 CLI 实现（commit 5432bab）
  ├─ Tester ─────→ test-cases.md（20 个测试用例）
  └─ Reviewer ───→ review.md（评审报告，3 阻塞项 + 8 改进项）
```

### 2.2 Agent 表现评估

| Agent | 任务 | 产出质量 | 时间 | 问题 |
|-------|------|----------|------|------|
| **Analyzer** | CLI 规格、文件模板 | ✅ 高 | ~3min | 依赖设计稿清晰度 |
| **Architect** | 技术架构 ADR | ✅ 高 | ~3min | 输出较冗长 |
| **Coder** | 完整实现 | ⚠️ 中 | ~16min | 3 个阻塞级 bug |
| **Tester** | 测试用例 | ✅ 高 | ~3min | 发现 Python 环境问题 |
| **Reviewer** | 代码评审 | ✅ 高 | ~5min | 需人工介入理解上下文 |

### 2.3 关键问题

**Coder 出现 3 个阻塞级 bug 的根因**：
1. **跨模块调用未检查**：`_is_tmp_expired` 在 `gc.py` 定义但 `db.py` 未导入
2. **Schema 与代码假设不匹配**：ADR 设计了 `memories_vec` schema，但实现时未对照检查
3. **异常被静默吞掉**：`except: pass` 导致 vec 删除失败无感知

**教训**：
- Agent 生成代码后，需要 Reviewer 严格把关
- 跨模块接口应在设计阶段明确定义
- `except: pass` 是危险模式，应避免

---

## 三、去场景化经验

### 3.1 多 Agent 协作

| 经验 | 说明 |
|------|------|
| **任务分配粒度** | 以「产出明确文档」为单位，而非「完成某个功能」 |
| **交接节点** | 前一个 Agent 完成后，再唤醒下一个 Agent |
| **设计文档先行** | Analyzer + Architect 的输出是 Coder 的输入，设计质量决定实现质量 |
| **评审不可省略** | Reviewer 发现的 3 个阻塞项说明代码生成必须有评审环节 |

### 3.2 CLI 项目结构规范

```bash
pm/                         # 项目根目录
├── __init__.py             # CLI 入口（Typer app）
├── pyproject.toml          # 依赖 + 入口点
├── config.py                # 配置文件加载（Pydantic）
├── db.py                    # 数据库操作
├── storage.py               # 文件读写（YAML front-matter）
├── idgen.py                 # ID 生成器
├── gc.py                    # 垃圾回收
├── embedding.py             # 向量嵌入
└── commands/                # 命令模块化
    ├── init.py
    └── add.py
```

**教训**：
- 入口点配置容易出错：`pm = "pm:app"` 需要 `pm/` 目录可 import
- uv 的 editable 安装在某些环境有 bug，需准备降级方案

### 3.3 向量检索落地

| 经验 | 说明 |
|------|------|
| **sqlite-vec 虚拟表** | `CREATE VIRTUAL TABLE USING vec0()` 默认只有 `embedding` 列 |
| **联接键** | 必须用 `memory_id TEXT` 作联接键，不能用 `rowid` |
| **降级方案** | Ollama/OpenAI 不可用时，自动降级到 LIKE 文本搜索 |

### 3.4 文档模板

| 文档 | 用途 | 关键要素 |
|------|------|----------|
| SKILL.md | Skill 定义 | trigger、commands、examples |
| ADR | 架构决策 | status、context、decision、consequences |
| tech-spec | 技术规格 | 接口定义、数据格式、错误处理 |
| test-cases | 测试用例 | 前置条件、步骤、预期结果 |

---

## 四、设计调整（对 agent-team 的改进）

### 4.1 工作流调整

| 原设计 | 改进 |
|--------|------|
| Coder 直接实现 | 改为：Coder 实现 → Reviewer 评审通过 → 才算完成 |
| 评审是可选的 | 改为：评审是必选项，阻塞项必须修复 |
| 单次任务 | 改为：支持迭代修复（Review → Coder → Review → ...） |

### 4.2 新增阶段：交接检查

在 Agent 之间传递任务时，增加「交接检查清单」：

```
[ ] 产出文档存在且格式正确
[ ] 接口定义清晰，无歧义
[ ] 依赖的模块/函数已导入
[ ] 异常处理已考虑边界情况
```

### 4.3 质量门禁

| 阶段 | 门禁条件 |
|------|----------|
| Architect → Coder | ADR 文档存在且被 Master 确认 |
| Coder → Reviewer | 代码已 commit，附上自测结果 |
| Review → Coder | 无阻塞项（BLOCKING） |

---

## 五、Skill 推荐（待集成）

| Agent | 推荐 Skill | 用途 |
|-------|-----------|------|
| **Analyzer** | skill-creator | 生成规范的需求文档模板 |
| **Architect** | skill-creator | 生成 ADR 格式架构文档 |
| **Coder** | python, coding-agent | 代码生成、复杂任务委托 |
| **Tester** | python | 测试代码审查 |
| **Reviewer** | python | 代码质量检查、安全扫描 |

详见 `agent-definitions.md`。

---

## 六、后续行动

- [ ] 修复 Reviewer 发现的 3 个阻塞项（B-1、B-2、B-3）
- [ ] 实现未完成的命令（`pm gc`、`pm delete`、`pm update`、`pm daemon`）
- [ ] 统一 `.pm.yaml` 字段名与 ADR spec
- [ ] 集成推荐 Skills 到 Agent 定义
