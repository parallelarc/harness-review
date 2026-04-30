# Claude Code Harness

## 背景

从 prompt engineering（单次交互）到 context engineering 再到 harness，背后是：

1. 将不确定的智能应用于确定的生产流程
2. 信息的更高效组织
3. 反映了更强的模型能力
4. 人类的角色变化

harness 还代表一种观念转变：默认模型能力已经很强，而人类逐渐成为系统效率的瓶颈。

---

## Agent Loop

Agent 的本质就是一个 `while true` 循环：**感知 → 决策 → 行动 → 反馈**。

大多数 Agent 系统都是以下 5 种模式的组合：

1. **提示链 Prompt Chaining**：任务拆成顺序步骤，每步 LLM 处理上一步的输出，中间可加代码检查点。适合生成后翻译、先写大纲再写正文这类线性流程。
2. **路由 Routing**：对输入分类，定向到对应的专用处理流程。简单问题走轻量模型，复杂问题走强模型；技术咨询和账单查询走不同逻辑。
3. **并行 Parallelization**：两种变体——分段法把任务拆成独立子任务并发跑，投票法把同一任务跑多次取共识。适合高风险决策或需要多视角的场景。
4. **编排器-工作者 Orchestrator-Workers**：中央 LLM 动态分解任务，委派给工作者 LLM，综合结果。Claude Code 的 spawn 工具和子 Agent 模式都是这个原型。
5. **评估器-优化器 Evaluator-Optimizer**：生成器产出，评估器给反馈，循环直到达标。适合翻译、创意写作这类质量标准难以用代码精确定义的任务。

![Agent Loop 模式](assets/img1.svg)

![Agent 模式组合](assets/img2.svg)

---

## 权限

### Claude Code 可见内容

- **项目文件**：目录及其子目录中的文件，以及经许可的其他位置的文件
- **终端**：任何你能运行的命令——构建工具、git、包管理器、系统实用程序、脚本
- **Git 状态**：当前分支、未提交的更改和最近的提交历史记录
- **CLAUDE.md**：人类引导 Claude 的行为
- **自动记忆**：Claude 在工作时会自动保存学习成果（项目模式和偏好设置），每次会话开始时加载 MEMORY.md 的前 200 行或 25KB

### 权限模式

共有 7 种权限模式，常用的有 4 种：

```typescript
type PermissionMode =
  | 'default'           // 敏感操作始终询问
  | 'acceptEdits'       // 自动批准文件编辑，其他询问
  | 'bypassPermissions' // 自动批准一切（危险）
  | 'dontAsk'           // 自动拒绝需要询问的操作
  | 'plan'              // 计划模式限制（只读 + 计划文件）
  | 'auto'              // AI 分类器自动审批（实验性）
  | 'bubble';           // 冒泡到父 Agent（子 Agent 用）
```

Claude Code 有完善的机制避免越界。

![权限系统](assets/image_3.png)

---

## Context Engineering

### 上下文构成

> Agent 看不到的内容等于不存在。

| 加载方式 | 机制 | 用途 |
|---------|------|------|
| 始终常驻 | CLAUDE.md | 项目契约 / 构建命令 / 禁止事项 |
| 按路径加载 | rules | 语言 / 目录 / 文件类型特定规则 |
| 按需加载 | Skills | 工作流 / 领域知识 |
| 隔离加载 | Subagents | 大量探索 / 并行研究 |
| 不进上下文 | Hooks | 确定性脚本 / 审计 / 阻断 |

![上下文构成](assets/image_4.png)

### 记忆系统

通用四级记忆系统：

| 层级 | 类型 | 说明 |
|------|------|------|
| 上下文窗口 | 工作记忆 | 当前任务所需的最小信息，token 有限，需主动管理 |
| Skills | 程序性记忆 | 怎么做某件事——操作流程、领域规范，按需加载不默认常驻 |
| JSONL 会话历史 | 情景记忆 | 发生了什么——磁盘持久化，支持跨会话检索 |
| MEMORY.md | 语义记忆 | Agent 主动写入认为重要的事实，每次启动时注入系统提示 |

![记忆系统](assets/img5.svg)

### Claude Code 记忆

Claude Code 拥有两个互补的记忆系统：**CLAUDE.md** 和 **Auto Memory**。这两个系统会在每次对话开始时加载。Claude 将它们视为上下文信息，而非强制配置。指令越具体、越简洁，Claude 就越能始终如一地执行。

**OpenClaw 混合检索：**

- `memory/YYYY-MM-DD.md`：追加写日志，保留原始细节
- `MEMORY.md`：精选事实，Agent 主动维护
- `memory_search`：70% 向量相似度 + 30% 关键词权重的混合检索

---

## Skill

记忆用于存储**事实**：你的环境、偏好、项目地点，以及 Agent 了解到的关于你的信息。技能用于存储**流程**：多步骤工作流程、工具特定指令和可复用的方案。用记忆来记住"是什么"，用技能来记住"怎么做"。

1. **Tools 是原子操作**（读文件、运行命令），**Skills 是工作流**（"帮我做 Code Review"、"帮我部署到 staging"）
2. Skill 描述要足够短：`Use when` / `Don't use when`
3. 没有反例时准确率从基准 73% 掉到 53%，加上反例后升到 85%，响应时间还降了 18.1%。**反例是必要的**

![Skill 示例](assets/img6.webp)

4. 常驻系统提示的只放高频 Skill，低频的不要塞进默认列表，需要时再手动引入。极低频的直接用文档替代就够了，不必做成 Skill

---

## Subagent

1. 主 Agent 作为 **Orchestrator** 统筹全局，下挂多个子 Agent 独立并行工作（比如读大文件、web search 时）
2. Orchestrator 和 subagent 之间**上下文隔离**，每个子 Agent 从空白消息列表开始，完成后只返回摘要
   - 避免权限泄露
   - 避免上下文污染

---

## 最佳实践

大多数最佳实践都基于一个限制：Claude 的上下文窗口很快就会填满，上下文窗口在超过 50% 之后，性能会指数级下降。

1. **会话管理，防止 context rot**
   - rewind 回到之前对话
   - 及时 `/compact` 上下文
   - HANDOFF.md + 新开对话
   - 一旦工作方向不对，直接新开对话比纠正错误更加高效
2. **Explore first, then plan, then code**
   - 标准工作流：spec → plan → implement → test
   - 提供明确的验收标准
3. **bypass mode**：`claude --dangerously-skip-permissions`
4. **确定性的逻辑交给确定性的工程**
5. **渐进式披露**
   - 把 Agent 当做人类来看，你需要提供足够明确的信息它才能工作
   - 明确 Agent 的能力边界
6. **CLAUDE.md 太长不如不写**

> Ultimately, we conclude that unnecessary requirements from context files make tasks harder, and human-written context files should describe only minimal requirements.
> — [Context Files in Software Engineering](https://arxiv.org/abs/2602.11988)

7. **Entropy Management**：时常整理代码库避免熵增

---

## Plugin 推荐

- [Claude Marketplace](https://claudemarketplaces.com/marketplaces)
- [Waza](https://github.com/tw93/waza)
- [Superpowers](https://github.com/obra/superpowers)
- [Claude Mem](https://github.com/thedotmack/claude-mem)
- [Claude HUD](https://github.com/jarrodwatts/claude-hud)

---

## 参考

- [Awesome CC Harness](https://wanlanglin.github.io/-awesome-cc-harness/zh/)
- [Agent 架构演进](https://tw93.fun/2026-03-21/agent.html)
- [How Claude Code Works](https://code.claude.com/docs/en/how-claude-code-works)
- [Hermes Agent Overview](https://hermes-agent.nousresearch.com/docs/user-guide/features/overview)
