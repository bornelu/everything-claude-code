# Everything Claude Code - 核心原理与设计思想深度分析

> 基于 Anthropic 黑客马拉松获奖项目的实战经验总结

---

## 目录

- [一、核心原理：配置即微调](#一核心原理配置即微调)
- [二、核心设计思想深度解析](#二核心设计思想深度解析)
- [三、可借鉴的核心思路](#三可借鉴的核心思路)
- [四、特性设计与开发的最佳实践](#四特性设计与开发的最佳实践)
- [五、可用于你项目的具体建议](#五可用于你项目的具体建议)
- [六、总结：核心设计原则](#六总结核心设计原则)

---

## 一、核心原理：配置即微调

这个项目的根本哲学是：**将配置视为微调而非架构**。这不是要构建一个复杂的系统，而是通过精心设计的约束和自动化，让 AI 能够在正确的轨道上工作。

### 1.1 分层约束系统

```
┌─────────────────────────────────────────────┐
│           Rules（始终遵循的原则）             │
│  - 编码风格（不可变性、小文件）              │
│  - Git 工作流（提交规范、PR 流程）           │
│  - 测试要求（TDD、80% 覆盖率）              │
│  - 安全规范（无硬编码密钥、输入验证）         │
└─────────────────────────────────────────────┘
              ↓ 基础原则
┌─────────────────────────────────────────────┐
│         Skills（工作流知识）                  │
│  - TDD 工作流（测试先行、覆盖率要求）         │
│  - 前端/后端模式                            │
│  - 安全审查流程                              │
│  - 特定语言最佳实践                          │
└─────────────────────────────────────────────┘
              ↓ 执行细节
┌─────────────────────────────────────────────┐
│       Hooks（事件驱动自动化）                 │
│  - PreToolUse: 前置验证（tmux 提醒）         │
│  - PostToolUse: 后置处理（格式化、类型检查）  │
│  - SessionStart/End: 上下文持久化           │
└─────────────────────────────────────────────┘
              ↓ 自动执行
┌─────────────────────────────────────────────┐
│     Agents（专业化子代理）                   │
│  - Planner: 复杂功能规划                     │
│  - TDD-Guide: 测试驱动开发                   │
│  - Code-Reviewer: 代码质量审查               │
│  - Security-Reviewer: 安全漏洞扫描           │
└─────────────────────────────────────────────┘
```

**核心设计思想：**

- **Rules** 告诉你"做什么"
- **Skills** 告诉你"怎么做"
- **Hooks** 自动执行"何时做"
- **Agents** 提供"谁来做的专业化"

### 1.2 项目结构概览

```bash
everything-claude-code/
├── .claude-plugin/
│   └── plugin.json          # 插件清单（12个agents）
├── agents/                  # 12个专业化子代理
│   ├── planner.md           # 功能规划
│   ├── architect.md         # 架构设计
│   ├── tdd-guide.md         # TDD工作流
│   ├── code-reviewer.md     # 代码审查
│   ├── security-reviewer.md # 安全审查
│   └── ...
├── skills/                  # 16+ 工作流定义
│   ├── tdd-workflow/        # TDD完整流程
│   ├── continuous-learning/ # 持续学习系统
│   ├── verification-loop/   # 验证循环
│   └── ...
├── commands/                # 23个斜杠命令
│   ├── plan.sh              # /plan
│   ├── tdd.sh               # /tdd
│   ├── code-review.sh       # /code-review
│   └── ...
├── hooks/                   # 事件驱动自动化
│   └── hooks.json           # 完整钩子配置
├── rules/                   # 始终遵循的规则
│   ├── common/              # 通用原则
│   │   ├── coding-style.md  # 不可变性、文件组织
│   │   ├── testing.md       # TDD、覆盖率
│   │   ├── security.md      # 安全检查清单
│   │   └── git-workflow.md  # Git工作流
│   ├── typescript/          # TypeScript特定规则
│   ├── python/              # Python特定规则
│   └── golang/              # Go特定规则
├── scripts/                 # 跨平台Node.js工具
│   └── hooks/               # Hook脚本
└── mcp-configs/             # MCP服务器配置
    └── mcp-servers.json     # 14+预配置MCP
```

---

## 二、核心设计思想深度解析

### 2.1 上下文窗口是稀缺资源

#### 问题

```javascript
// 工具太多导致上下文窗口急剧缩减
200k tokens（理论值）
  ↓ 启用 14+ MCP 服务器
70k tokens（实际可用）
```

#### 解决方案

```javascript
// 按需启用，保持 <10 个活跃 MCP
{
  "github": true,      // 当前项目需要
  "supabase": true,    // 当前项目需要
  "memory": false,     // 暂时禁用
  "clickhouse": false  // 暂时禁用
}
```

**设计原则：**

- 只启用当前项目需要的 MCP
- 保持 <80 个活跃工具
- 使用 `/plugins` 或 `/mcp` 快速切换

**关键数字：**

| 指标 | 推荐值 | 最大值 |
|-----|--------|--------|
| 启用的 MCP | <10 | 15 |
| 活跃工具数 | <80 | 100 |
| 有效上下文 | >150k | 200k |

### 2.2 不可变性优先

#### 代码示例

```javascript
// ❌ 错误：可变模式
function addUser(users, user) {
  users.push(user);  // 修改原数组
  return users;
}

// ✅ 正确：不可变模式
function addUser(users, user) {
  return [...users, user];  // 返回新数组
}
```

#### 为什么要坚持不可变性

1. **防止隐藏副作用**：函数不会意外修改外部状态
2. **易于调试**：数据变化可追踪
3. **安全并发**：多线程/多进程环境下无竞态条件
4. **时间旅行调试**：可以回滚到任何状态

#### 实际应用

```typescript
// React 中的不可变性
const [state, setState] = useState({ items: [] });

// ❌ 错误
state.items.push newItem);  // 直接修改

// ✅ 正确
setState(prev => ({
  ...prev,
  items: [...prev.items, newItem]
}));
```

### 2.3 小文件 > 大文件

#### 推荐结构

```
- 200-400 行（典型）
- 800 行（上限）
- 按功能/领域组织，而非按类型
```

#### 示例对比

```bash
❌ 避免大文件：
utils.ts (2000 行)
controllers/
  ├── user.controller.ts (1500 行)
  └── auth.controller.ts (1200 行)

✅ 推荐：
utils/
  ├── array.ts (150 行)
  ├── string.ts (200 行)
  └── date.ts (180 行)
controllers/
  ├── user/
  │   ├── user.controller.ts (300 行)
  │   ├── user.service.ts (350 行)
  │   └── user.model.ts (150 行)
  └── auth/
      ├── auth.controller.ts (280 行)
      └── auth.service.ts (320 行)
```

#### 设计理由

- **高内聚低耦合**：每个文件职责单一
- **易于导航**：快速定位功能
- **减少冲突**：多人协作时减少合并冲突
- **提升可测试性**：小模块易于编写单元测试

### 2.4 事件驱动自动化

#### Hooks 事件类型

| 事件类型 | 触发时机 | 用途 | 示例 |
|---------|---------|------|------|
| **PreToolUse** | 工具执行前 | 验证、提醒 | tmux提醒、git push前确认 |
| **PostToolUse** | 工具执行后 | 格式化、反馈 | 自动格式化、类型检查 |
| **UserPromptSubmit** | 用户发送消息时 | 输入验证 | 命令参数检查 |
| **Stop** | Claude完成响应后 | 最终检查 | console.log检查 |
| **PreCompact** | 上下文压缩前 | 保存状态 | 记忆持久化 |
| **SessionStart** | 会话开始时 | 初始化 | 加载上下文、检测包管理器 |
| **SessionEnd** | 会话结束时 | 清理、学习 | 持久化状态、提取模式 |

#### 实际示例

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && command matches \"npm run dev\"",
      "hooks": [{
        "type": "command",
        "command": "echo '[Hook] 必须在 tmux 中运行开发服务器'"
      }]
    }
  ],
  "PostToolUse": [
    {
      "matcher": "tool == \"Edit\" && file matches \"\\.(ts|tsx)$\"",
      "hooks": [
        "prettier --write",     // 自动格式化
        "tsc --noEmit"          // 类型检查
      ]
    }
  ]
}
```

### 2.5 专业化子代理系统

#### 子代理类型

| 子代理 | 工具权限 | 模型 | 职责 |
|-------|---------|------|------|
| **planner** | Read, Grep, Glob | Opus | 复杂功能规划 |
| **architect** | Read, Grep, Glob | Opus | 系统架构设计 |
| **tdd-guide** | Write, Edit, Bash | Sonnet | TDD工作流指导 |
| **code-reviewer** | Read, Grep, Glob, Bash | Opus | 代码质量审查 |
| **security-reviewer** | Read, Grep, Glob | Opus | 安全漏洞扫描 |
| **build-error-resolver** | Read, Write, Edit, Bash | Sonnet | 构建错误修复 |
| **e2e-runner** | Read, Write, Bash | Sonnet | E2E测试自动化 |
| **refactor-cleaner** | Read, Edit, Bash | Sonnet | 死代码清理 |

#### 设计原则

```yaml
planner:
  tools: [Read, Grep, Glob]  # 只读工具，不修改代码
  model: opus
  philosophy: >
    深度分析需求，创建详细实施计划。
    不写代码，只规划。

code-reviewer:
  tools: [Read, Grep, Glob, Bash]  # 可以运行测试
  model: opus
  philosophy: >
    质量审查、安全扫描、性能分析。
    不修改代码，只提供建议。
```

---

## 三、可借鉴的核心思路

### 3.1 事件驱动自动化

#### 核心价值

这个项目最精妙的设计之一是 **Hooks 系统**，它在正确的时机自动执行任务：

**借鉴价值：**

- **质量门禁**：在工具执行前验证条件
- **自动格式化**：代码修改后立即整理风格
- **即时反馈**：错误在引入时就被捕获
- **零心智负担**：开发者无需记住手动执行

#### 实际应用场景

```javascript
// 你的项目可以这样设计
{
  "PreToolUse": {
    "git commit": "检查是否有 TODO",
    "npm publish": "自动运行测试套件",
    "docker push": "验证镜像标签"
  },
  "PostToolUse": {
    "Edit .ts": "运行 ESLint",
    "Edit .py": "运行 Black 格式化",
    "Write file": "更新文件索引"
  }
}
```

#### Hook 模式库

```json
{
  "PreToolUse": [
    {
      "name": "block-todo-in-commits",
      "matcher": "git commit",
      "hook": "grep -r 'TODO' src/ && exit 1"
    },
    {
      "name": "require-tests-before-publish",
      "matcher": "npm publish",
      "hook": "npm test || exit 1"
    }
  ],
  "PostToolUse": [
    {
      "name": "auto-format-typescript",
      "matcher": "Edit .ts",
      "hook": "prettier --write && eslint --fix"
    },
    {
      "name": "type-check",
      "matcher": "Edit .ts",
      "hook": "tsc --noEmit"
    }
  ]
}
```

### 3.2 专业化子代理

#### 核心价值

将复杂任务委托给专业化子代理，每个子代理有：

- **有限的工具权限**：只授予完成任务所需的工具
- **专注的领域知识**：加载特定的 skills
- **明确的职责边界**：避免职责混乱

#### 借鉴价值

- **减少上下文切换**：主代理保持高层视角，细节工作委托出去
- **并行执行**：多个子代理可以同时工作（使用 git worktrees）
- **降低 token 消耗**：子代理只加载相关技能，不污染主上下文

#### 实际应用

```python
# 在你的项目中可以这样设计
agents/
  ├── api-designer.md      # API 设计专家（只读，生成 OpenAPI 规范）
  ├── database-migrator.md # 数据库迁移专家（只生成 SQL，不执行）
  ├── test-generator.md    # 测试生成专家（只写测试）
  └── performance-tuner.md # 性能优化专家（分析瓶颈，提供建议）
```

#### 子代理模板

```markdown
---
name: api-designer
description: API 设计专家，生成 RESTful API 规范
tools: ["Read", "Grep", "Glob"]
model: sonnet
---

# API 设计专家

## 职责
- 分析业务需求
- 设计 RESTful API 端点
- 生成 OpenAPI 3.0 规范
- 定义请求/响应 schema

## 工作流程
1. 理解业务需求
2. 分析现有 API 模式
3. 设计新端点结构
4. 生成 OpenAPI 规范
5. 编写集成测试

## 约束
- 只读操作，不修改代码
- 遵循 RESTful 最佳实践
- 保持 API 版本兼容性
```

### 3.3 持续学习系统

#### 核心价值

```javascript
// 工作流程
SessionEnd
  → 分析会话（提取 10+ 条消息的模式）
  → 识别可复用模式
  → 保存到 ~/.claude/skills/learned/
  → 未来会话自动应用
```

**借鉴价值：**

- **知识积累**：从每次交互中学习，避免重复错误
- **团队共享**：learned skills 可以提交到代码仓库，团队共享
- **渐进式改进**：随着使用时间增长，AI 越来越了解项目

#### 检测的模式类型

```javascript
patterns_to_detect: [
  "error_resolution",      // 如何解决特定错误
  "user_corrections",      // 用户修正模式
  "workarounds",          // 框架/库的变通方案
  "debugging_techniques", // 有效调试方法
  "project_specific"      // 项目特定约定
]
```

#### 实际应用

```bash
# 在你的项目中
learned/
  ├── error-fixes.md       # 特定错误修复模式
  ├── api-conventions.md   # API 命名约定
  ├── database-patterns.md # 数据库查询模式
  └── debugging-tips.md    # 调试技巧
```

#### v2 增强方向（基于 Homunculus）

| 特性 | v1 | v2（建议） |
|-----|-----|----------|
| 观察方式 | Stop hook | PreToolUse/PostToolUse |
| 分析方式 | 主上下文 | 后台 Haiku agent |
| 粒度 | 完整 skills | 原子 "instincts" |
| 置信度 | 无 | 0.3-0.9 加权 |
| 演化路径 | 直接到 skill | Instincts → cluster → skill |

### 3.4 TDD 强制执行

#### 核心价值

这个项目通过 skill + rules + hooks 的组合，**强制执行 TDD**：

```yaml
# tdd-workflow/skill.md
when: 写新功能、修复 bug、重构
steps:
  1. 编写用户故事
  2. 生成测试用例（单元、集成、E2E）
  3. 运行测试（应该失败）
  4. 实现代码
  5. 运行测试（应该通过）
  6. 重构
  7. 验证覆盖率 ≥80%

# rules/testing.md
always:
  - 测试先行，代码后写
  - 80% 覆盖率要求（单元 + 集成 + E2E）
  - 每个功能必须有测试

# hooks/hooks.json
PreToolUse:
  - Write/Edit 代码 → 检查是否有对应测试
```

#### TDD 工作流步骤

```typescript
// Step 1: 编写用户故事
As a user, I want to search for markets semantically,
so that I can find relevant markets even without exact keywords.

// Step 2: 生成测试用例
describe('Semantic Search', () => {
  it('returns relevant markets for query', async () => {
    // 测试实现
  })

  it('handles empty query gracefully', async () => {
    // 测试边界情况
  })

  it('falls back to substring search when Redis unavailable', async () => {
    // 测试降级行为
  })
})

// Step 3: 运行测试（失败）
npm test  // ❌ Failed: Function not implemented

// Step 4: 实现代码
export async function searchMarkets(query: string) {
  // 实现代码
}

// Step 5: 运行测试（通过）
npm test  // ✅ All tests passed

// Step 6: 重构
// 改进代码质量，保持测试绿色

// Step 7: 验证覆盖率
npm run test:coverage  // 85% coverage ✅
```

#### 借鉴要点

- **质量保证**：测试不是可选的，是强制性的
- **文档作用**：测试即文档，描述功能的预期行为
- **重构信心**：有测试保护，可以安全重构

---

## 四、特性设计与开发的最佳实践

### 4.1 规划优先

从 `planner.md` 学习到的规划方法：

#### 完整示例

```markdown
# 实施计划：语义搜索市场

## 概述
使用 OpenAI embeddings 实现 Redis 向量搜索，提升市场搜索体验

## 需求
- 用户可以用语义描述搜索市场（不需要精确关键词）
- 降级到子字符串搜索（Redis 不可用时）
- 按相似度分数排序结果

## 架构变更
- lib/embeddings.ts: 新建 - OpenAI embedding 生成
- lib/redis-vector.ts: 新建 - Redis 向量搜索
- app/api/markets/route.ts: 修改 - 添加语义搜索端点

## 实施步骤

### Phase 1: 基础设施
1. **设置 Redis Stack** (docker-compose.yml)
   - Action: 添加 redis-stack 服务
   - Why: 需要 RediSearch 模块支持向量搜索
   - Risk: Low
   - Dependencies: None

2. **生成 Embeddings** (lib/embeddings.ts)
   - Action: 创建 generateEmbedding 函数
   - Why: 将市场描述转换为向量
   - Risk: Low
   - Dependencies: Phase 1.1

### Phase 2: API 集成
3. **语义搜索端点** (app/api/markets/route.ts)
   - Action: 添加 /api/markets/search/semantic
   - Why: 暴露语义搜索功能
   - Risk: Medium
   - Dependencies: Phase 1

### Phase 3: 测试与优化
4. **单元测试** (lib/embeddings.test.ts)
   - Action: 测试 embedding 生成
   - Why: 确保正确性
   - Risk: Low
   - Dependencies: Phase 1.2

5. **集成测试** (app/api/markets/route.test.ts)
   - Action: 测试 API 端点
   - Why: 验证端到端流程
   - Risk: Medium
   - Dependencies: Phase 2.3

6. **E2E 测试** (e2e/semantic-search.spec.ts)
   - Action: 测试用户搜索流程
   - Why: 验证用户体验
   - Risk: Medium
   - Dependencies: Phase 2.3

## 测试策略
- 单元测试: embedding 生成、向量相似度计算
- 集成测试: API 端点、Redis 交互
- E2E 测试: 用户搜索流程

## 风险与缓解
- **Risk**: OpenAI API 限流
  - **Mitigation**: 批量处理、缓存 embeddings
- **Risk**: Redis 内存溢出
  - **Mitigation**: 设置最大内存策略、定期清理

## 成功标准
- [ ] 语义搜索返回相关结果
- [ ] 降级机制正常工作
- [ ] 80%+ 测试覆盖率
- [ ] 响应时间 <500ms
- [ ] 通过所有 E2E 测试
```

#### 借鉴要点

1. **渐进式实施**：分阶段，每阶段可验证
2. **明确依赖**：步骤之间的依赖关系清晰
3. **风险意识**：提前识别风险并准备缓解措施
4. **成功标准**：可量化的验收标准

### 4.2 代码审查清单

从 `code-reviewer.md` 学习到的审查方法：

#### 完整清单

```yaml
checklist:
  # 安全检查（关键）
  security:
    - 无硬编码密钥
    - SQL 注入防护
    - XSS 防护
    - 输入验证
    - 依赖漏洞检查
    - 路径遍历防护
    - CSRF 防护
    - 认证绕过检查

  # 代码质量（高）
  quality:
    - 函数 <50 行
    - 文件 <800 行
    - 嵌套 <4 层
    - 错误处理完整
    - 无 console.log
    - 无重复代码
    - 变量命名清晰
    - 类型注解完整

  # 性能（中）
  performance:
    - 算法复杂度优化
    - React memoization
    - 缓存策略
    - N+1 查询检查
    - Bundle 大小优化
    - 图片优化

  # 最佳实践（中）
  best_practices:
    - 无 emoji（代码/注释）
    - 公共 API 有 JSDoc
    - 无障碍支持
    - 魔法数字有解释
    - 一致的格式化
```

#### 输出格式

```
[CRITICAL] 硬编码 API key
File: src/api/client.ts:42
Issue: API key 暴露在源代码中
Fix: 移动到环境变量

const apiKey = "sk-abc123";  // ❌ 错误
const apiKey = process.env.API_KEY;  // ✅ 正确

---

[HIGH] 函数过长
File: src/utils/data.ts:123
Issue: 函数 processData 有 120 行，超过 50 行限制
Fix: 拆分为多个小函数

function processData(data) { /* 120 行 */ }  // ❌ 错误

function validateData(data) { /* 20 行 */ }    // ✅ 正确
function transformData(data) { /* 30 行 */ }
function formatOutput(data) { /* 25 行 */ }

---

[MEDIUM] 缺少错误处理
File: src/api/fetch.ts:15
Issue: fetch 调用没有 try-catch
Fix: 添加错误处理

const response = await fetch(url);  // ❌ 错误

try {                                // ✅ 正确
  const response = await fetch(url);
  if (!response.ok) throw new Error(...);
} catch (error) {
  handleError(error);
}
```

#### 借鉴要点

- **分级审查**：Critical/High/Medium，优先级清晰
- **具体示例**：不仅指出问题，还提供修复方案
- **可操作**：每个问题都有明确的 action item

### 4.3 自动化质量门禁

从 `hooks.json` 学习到的自动化模式：

#### 完整配置

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && command matches \"npm run dev\"",
      "hooks": [{
        "type": "command",
        "command": "node -e \"console.error('[Hook] 必须在 tmux 中运行');process.exit(1)\""
      }],
      "description": "阻止在 tmux 外运行开发服务器"
    },
    {
      "matcher": "tool == \"Bash\" && command matches \"git push\"",
      "hooks": [{
        "type": "command",
        "command": "echo '[Hook] 请在推送前审查更改'"
      }],
      "description": "推送前提醒审查"
    },
    {
      "matcher": "tool == \"Write\" && file matches \"\\\\.md$\"",
      "hooks": [{
        "type": "command",
        "command": "check-if-allowed-md-file"
      }],
      "description": "阻止创建不必要的 .md 文件"
    }
  ],
  "PostToolUse": [
    {
      "matcher": "tool == \"Edit\" && file matches \"\\\\.(ts|tsx)$\"",
      "hooks": [
        {
          "type": "command",
          "command": "npx prettier --write",
          "description": "自动格式化 TypeScript"
        },
        {
          "type": "command",
          "command": "npx tsc --noEmit",
          "description": "TypeScript 类型检查"
        }
      ]
    },
    {
      "matcher": "tool == \"Edit\" && file matches \"\\\\.(ts|tsx|js|jsx)$\"",
      "hooks": [{
        "type": "command",
        "command": "grep console.log || echo '无 console.log'",
        "description": "检查 console.log"
      }]
    }
  ],
  "Stop": [
    {
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "check-modified-files-for-console.log",
        "description": "响应结束后检查 console.log"
      }]
    }
  ]
}
```

#### 借鉴要点

- **即时反馈**：问题在引入时立即发现
- **零摩擦**：自动化执行，开发者无需记住
- **渐进式**：从警告开始，逐步加强到阻止

### 4.4 测试金字塔

从 `tdd-workflow` 学习的测试策略：

```
        /\
       /  \
      / E2E \        少量关键用户流程
     /________\
    /          \
   / Integration \    中等数量，测试交互
  /______________\
 /                \
/    Unit Tests    \  大量小测试，快速反馈
/__________________\
```

#### 测试类型对比

| 测试类型 | 数量 | 速度 | 覆盖范围 | 示例 |
|---------|-----|------|---------|------|
| **单元测试** | 大量 | 快 (<50ms) | 单个函数 | 计算折扣、格式化日期 |
| **集成测试** | 中等 | 中等 (<5s) | 模块交互 | API 端点、数据库操作 |
| **E2E 测试** | 少量 | 慢 (<30s) | 完整流程 | 用户注册、结账流程 |

#### 覆盖率要求

```json
{
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

---

## 五、可用于你项目的具体建议

### 5.1 建立分层规则体系

#### 目录结构

```bash
your-project/
├── .claude/
│   ├── rules/
│   │   ├── common/
│   │   │   ├── coding-style.md      # 不可变性、小文件
│   │   │   ├── testing.md           # TDD、80% 覆盖率
│   │   │   ├── security.md          # 输入验证、密钥管理
│   │   │   └── git-workflow.md      # 提交规范、PR 流程
│   │   └── typescript/
│   │       ├── react-patterns.md    # React 最佳实践
│   │       └── nextjs-patterns.md   # Next.js 约定
│   ├── skills/
│   │   ├── tdd-workflow/            # TDD 工作流
│   │   ├── api-design/              # API 设计指南
│   │   └── error-handling/          # 错误处理模式
│   ├── agents/
│   │   ├── planner.md               # 功能规划
│   │   ├── code-reviewer.md         # 代码审查
│   │   └── test-generator.md        # 测试生成
│   └── hooks.json                   # 自动化钩子
```

#### coding-style.md 示例

```markdown
# Coding Style Rules

## 不可变性（关键）

ALWAYS 创建新对象，NEVER 修改现有对象：

```typescript
// ❌ 错误
function addItem(items, item) {
  items.push(item);
  return items;
}

// ✅ 正确
function addItem(items, item) {
  return [...items, item];
}
```

## 文件组织

- 200-400 行（典型）
- 800 行（上限）
- 按功能/领域组织

## 错误处理

- 在每个层级显式处理错误
- 在 UI 代码中提供友好的错误消息
- 在服务器端记录详细的错误上下文
- 永远不要静默吞没错误

## 代码质量清单

在标记工作完成前：
- [ ] 代码可读且命名良好
- [ ] 函数很小（<50 行）
- [ ] 文件专注（<800 行）
- [ ] 没有深层嵌套（>4 层）
- [ ] 正确的错误处理
- [ ] 没有硬编码值
- [ ] 使用了不可变模式
```

### 5.2 设计质量门禁

#### hooks.json 示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "tool == \"Bash\" && command matches \"git commit\"",
        "hooks": [{
          "type": "command",
          "command": "npm test && npm run lint || exit 1",
          "description": "提交前必须通过测试和 lint"
        }]
      },
      {
        "matcher": "tool == \"Write\" && file matches \"\\\\.md$\"",
        "hooks": [{
          "type": "command",
          "command": "node -e \"const p=process.stdin;let d='';p.on('data',c=>d+=c);p.on('end',()=>{const i=JSON.parse(d);const f=i.tool_input?.file_path||'';if(!f.match(/(README|CLAUDE|CONTRIBUTING)\\.md$/)){console.error('[Hook] 只允许创建必要的文档');process.exit(1)}})\"",
          "description": "阻止创建不必要的 .md 文件"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "tool == \"Edit\" && file matches \"\\\\.(ts|tsx)$\"",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write",
            "description": "自动格式化 TypeScript"
          },
          {
            "type": "command",
            "command": "npx eslint --fix",
            "description": "自动修复 ESLint 问题"
          },
          {
            "type": "command",
            "command": "npx tsc --noEmit",
            "description": "TypeScript 类型检查"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "command",
          "command": "check-console-logs",
          "description": "检查是否有 console.log"
        }]
      }
    ]
  }
}
```

### 5.3 创建专业化子代理

#### planner.md 示例

```markdown
---
name: planner
description: 功能规划专家，创建详细实施计划
tools: ["Read", "Grep", "Glob"]
model: opus
---

# 功能规划专家

## 职责
- 分析需求并创建详细实施计划
- 将复杂功能分解为可管理步骤
- 识别依赖关系和潜在风险
- 建议最佳实施顺序

## 规划流程

### 1. 需求分析
- 完全理解功能需求
- 如有需要，提出澄清问题
- 识别成功标准
- 列出假设和约束

### 2. 架构审查
- 分析现有代码库结构
- 识别受影响的组件
- 审查类似实现
- 考虑可复用模式

### 3. 步骤分解
创建详细步骤：
- 清晰、具体的行动
- 文件路径和位置
- 步骤之间的依赖
- 预估复杂度
- 潜在风险

### 4. 实施顺序
- 按依赖关系排序
- 分组相关更改
- 最小化上下文切换
- 启用增量测试

## 计划格式

```markdown
# 实施计划：[功能名称]

## 概述
[2-3 句话总结]

## 需求
- [需求 1]
- [需求 2]

## 架构变更
- [变更 1: 文件路径和描述]
- [变更 2: 文件路径和描述]

## 实施步骤

### Phase 1: [阶段名称]
1. **[步骤名称]** (文件: path/to/file.ts)
   - 行动: 具体行动
   - 原因: 此步骤的原因
   - 依赖: 无 / 需要步骤 X
   - 风险: 低/中/高

## 测试策略
- 单元测试: [要测试的文件]
- 集成测试: [要测试的流程]
- E2E 测试: [要测试的用户旅程]

## 风险与缓解
- **风险**: [描述]
  - 缓解: [如何处理]

## 成功标准
- [ ] 标准 1
- [ ] 标准 2
```

## 最佳实践

1. **具体**：使用精确的文件路径、函数名、变量名
2. **考虑边界情况**：思考错误场景、null 值、空状态
3. **最小化更改**：优先扩展现有代码而非重写
4. **保持模式**：遵循现有项目约定
5. **启用测试**：结构化更改以易于测试
6. **增量思考**：每个步骤都应可验证
7. **记录决策**：解释为什么，而不仅仅是是什么
```

#### code-reviewer.md 示例

```markdown
---
name: code-reviewer
description: 代码审查专家，检查质量和安全
tools: ["Read", "Grep", "Glob", "Bash"]
model: opus
---

# 代码审查专家

## 职责
- 确保代码质量和安全性
- 识别潜在问题并提供修复建议
- 强制执行项目标准

## 触发时机
- 编写或修改代码后立即使用
- 必须对所有代码更改使用

## 工作流程

当被调用时：
1. 运行 git diff 查看最近的更改
2. 专注于修改的文件
3. 立即开始审查

## 审查清单

### 安全检查（关键）
- 硬编码凭证（API 密钥、密码、令牌）
- SQL 注入风险（查询中的字符串拼接）
- XSS 漏洞（未转义的用户输入）
- 缺少输入验证
- 不安全的依赖（过时、有漏洞）
- 路径遍历风险（用户控制的文件路径）
- CSRF 漏洞
- 认证绕过

### 代码质量（高）
- 大函数（>50 行）
- 大文件（>800 行）
- 深层嵌套（>4 层）
- 缺少错误处理
- console.log 语句
- 变异模式
- 新代码缺少测试

### 性能（中）
- 低效算法（O(n²) 而非 O(n log n) 可能）
- React 中不必要的重新渲染
- 缺少 memoization
- 大的 bundle 大小
- 未优化的图像
- 缺少缓存
- N+1 查询

### 最佳实践（中）
- 代码/注释中的 emoji
- 没有 ticket 的 TODO/FIXME
- 公共 API 缺少 JSDoc
- 无障碍问题（缺少 ARIA 标签、对比度差）
- 变量命名不佳（x、tmp、data）
- 没有解释的魔法数字
- 格式不一致

## 输出格式

对于每个问题：
```
[CRITICAL] 硬编码 API key
文件: src/api/client.ts:42
问题: API key 暴露在源代码中
修复: 移动到环境变量

const apiKey = "sk-abc123";  // ❌ 错误
const apiKey = process.env.API_KEY;  // ✅ 正确
```

## 批准标准

- ✅ 批准：没有 CRITICAL 或 HIGH 问题
- ⚠️ 警告：仅 MEDIUM 问题（可谨慎合并）
- ❌ 阻止：发现 CRITICAL 或 HIGH 问题
```

### 5.4 建立持续学习机制

#### evaluate-session.sh 示例

```bash
#!/bin/bash
# 在会话结束时提取模式

SESSION_LENGTH=$(claude session info | grep "messages" | awk '{print $2}')

if [ $SESSION_LENGTH -gt 10 ]; then
  echo "会话足够长，提取模式..."

  # 调用 Claude 分析会话
  claude analyze-session \
    --extract-patterns \
    --save-to ~/.claude/skills/learned/ \
    --patterns error_resolution,user_corrections,workarounds
fi
```

#### learned-skills 结构

```bash
~/.claude/skills/learned/
├── error-fixes.md          # 错误修复模式
├── api-conventions.md      # API 约定
├── database-patterns.md    # 数据库模式
└── debugging-tips.md       # 调试技巧
```

### 5.5 实施检查清单

#### 第一阶段：基础设置（Week 1）

- [ ] 创建 `.claude/rules/` 目录
- [ ] 添加 `coding-style.md`（不可变性、小文件）
- [ ] 添加 `testing.md`（TDD、覆盖率要求）
- [ ] 添加 `security.md`（输入验证、密钥管理）
- [ ] 创建 `hooks.json`（基本自动化）

#### 第二阶段：工作流（Week 2-3）

- [ ] 创建 `tdd-workflow/` skill
- [ ] 添加 `planner.md` agent
- [ ] 添加 `code-reviewer.md` agent
- [ ] 创建斜杠命令（`/plan`、`/tdd`、`/review`）

#### 第三阶段：高级功能（Week 4+）

- [ ] 设置持续学习系统
- [ ] 添加语言特定规则（TypeScript、Python、Go）
- [ ] 配置 MCP 服务器（GitHub、Supabase、Vercel）
- [ ] 创建项目特定 skills

---

## 六、总结：核心设计原则

这个项目最值得借鉴的**五大核心原则**：

### 1. 配置即微调

**不是构建复杂架构，而是精心设计约束**

```yaml
理念: >
  通过 rules、skills、hooks、agents 的组合，
  在不修改 AI 模型的情况下，
  实现类似微调的效果。

关键:
  - Rules: 定义做什么
  - Skills: 定义怎么做
  - Hooks: 自动化何时做
  - Agents: 定义谁来做
```

### 2. 上下文优先

**一切设计考虑 token 效率**

```yaml
优化策略:
  - 只启用当前需要的 MCP（<10 个）
  - 保持活跃工具 <80 个
  - 使用专业化子代理减少主上下文负担
  - 定期手动压缩上下文

数字:
  理论上下文: 200k tokens
  实际可用: 150k+ tokens（优化后）
  vs
  实际可用: 70k tokens（未优化）
```

### 3. 自动化一切

**用 hooks 消除重复性任务**

```yaml
自动化层级:
  PreToolUse:  验证和提醒（执行前）
  PostToolUse: 格式化和检查（执行后）
  Stop:        最终审查（响应后）
  Session:     上下文管理（会话级）

效果:
  - 零心智负担
  - 即时反馈
  - 质量门禁
```

### 4. 专业化分工

**用 agents 实现关注点分离**

```yaml
子代理类型:
  - 只读代理:  Planner、Architect（分析、规划）
  - 修改代理:  TDD-Guide、Refactor-Cleaner（写代码）
  - 审查代理:  Code-Reviewer、Security-Reviewer（检查）

优势:
  - 减少上下文切换
  - 并行执行（git worktrees）
  - 降低 token 消耗
```

### 5. 持续改进

**从每次交互中学习**

```yaml
学习流程:
  SessionEnd
    → 分析会话（10+ 条消息）
    → 识别可复用模式
    → 保存到 learned skills
    → 未来会话自动应用

进化方向:
  v1: Stop hook + 完整 skills
  v2: PreToolUse/PostToolUse + atomic instincts + 置信度
```

---

## 对于特性开发的启示

### 开发流程

```
1. 需求分析
   ↓
2. 使用 /plan 创建详细计划
   ↓
3. 使用 /tdd 开始测试驱动开发
   ↓
4. 实现代码（hooks 自动格式化、类型检查）
   ↓
5. 使用 /review 进行代码审查
   ↓
6. Git 提交（hooks 自动运行测试）
   ↓
7. 会话结束（自动提取模式到 learned skills）
```

### 质量保证体系

```
┌─────────────────────────────────────┐
│         PreToolUse Hooks             │
│  - 验证前置条件                      │
│  - 阻止不良实践                      │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│         Rules & Skills               │
│  - 强制 TDD 工作流                   │
│  - 不可变性优先                      │
│  - 80% 覆盖率要求                    │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│         PostToolUse Hooks            │
│  - 自动格式化                        │
│  - 类型检查                          │
│  - Lint 检查                         │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│         Code Reviewer Agent          │
│  - 安全审查                          │
│  - 性能分析                          │
│  - 最佳实践检查                      │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│         Stop Hooks                   │
│  - 最终检查（console.log 等）        │
└─────────────────────────────────────┘
```

---

## 最终建议

### 如果你想借鉴这个项目：

#### 从小处开始

1. **Week 1**: 设置基础 rules（coding-style、testing、security）
2. **Week 2**: 添加基本 hooks（自动格式化、类型检查）
3. **Week 3**: 创建第一个 agent（code-reviewer）
4. **Week 4**: 添加 skills（tdd-workflow、api-design）

#### 逐步优化

1. **Month 1**: 建立基础工作流
2. **Month 2**: 添加专业 agents（planner、security-reviewer）
3. **Month 3**: 实施持续学习
4. **Month 4+**: 根据项目需求定制

#### 避免的陷阱

- ❌ 一次性设置所有东西（会不堪重负）
- ❌ 复制粘贴而不理解（需要根据项目调整）
- ❌ 启用太多 MCP（上下文窗口会急剧缩小）
- ❌ 忽视团队反馈（需要根据实际使用调整）

---

## 结语

**Everything Claude Code** 项目展示了如何通过精心设计的约束和自动化，让 AI 在软件开发的正确轨道上工作。这些思想不仅适用于 AI 辅助开发，也适用于**任何需要高质量代码交付的软件项目**。

**核心启示：**

> 好的软件不是通过复杂的架构实现的，而是通过：
> 1. 清晰的原则
> 2. 合理的约束
> 3. 智能的自动化
> 4. 持续的改进

这些思想可以帮助你构建更高质量、更易维护、更可测试的软件系统。

---

**参考资源：**

- [Everything Claude Code GitHub](https://github.com/affaan-m/everything-claude-code)
- [Claude Code 官方文档](https://code.claude.com/docs)
- [Shorthand Guide](https://x.com/affaanmustafa/status/2012378465664745795)
- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352)

---

*本文档基于 Everything Claude Code 项目的深度分析，总结了其核心设计思想和可借鉴的最佳实践。*
