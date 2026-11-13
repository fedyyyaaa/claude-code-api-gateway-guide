# 8. 双 Runtime 与 Single Runtime：两种架构的取舍

> **仅个人体验。**
>
> 这一章来自我们自己的项目演进和实际维护经历，只讨论两种架构在工程上的取舍。不同项目的现有代码、工具系统、上下文设计、部署方式和目标都不同，没有一种方案适合所有人。

先把这一章最容易被误解的一点说清楚：

> **我们的目标一直是“自建前端能在 Claude 订阅和第三方 API 之间随时切换”。**

Single Runtime 和双 Runtime 是实现这个产品目标的两种工程路线，不是两个不同的产品目标。

前面的章节一直在讲我们最后选中的这条路线：

~~~text
Claude 订阅 OAuth ──┐
                    ├→ Claude Code runtime → tools / session / context
第三方 API Gateway ─┘
~~~

也就是让订阅和 API 共用 Claude Code，只在启动边界切换底层 provider。

但这只是其中一种做法。

我们自己的项目早期更接近下面的结构：

~~~text
API
 ↓
自己的 API runtime
 ↓
自己的 tool loop
 ↓
共享后端工具


Claude Code
 ↓
Claude Code runtime
 ↓
共享后端工具
~~~

两条路都能工作，而且双 Runtime 有一些很实际的优势。后来我们才逐步把它收成现在的 Single Runtime。

这一章把两种做法放在一起比较。

---

## 8.1 两种方案分别在维护什么

为了方便说明，这里先约定两个名字。

### 双 Runtime

API 和 Claude Code 各自拥有自己的生成运行时。

~~~text
                     ┌→ API Messages runtime
前端 → 后端 provider ┤
                     └→ Claude Code runtime
~~~

两边可以共用数据库、工具实现、MCP、Memory 和前端，但 agent loop、session、历史投影、工具调用协议、错误恢复等能力可能各自维护。

### Single Runtime

无论底层使用订阅还是 API，都先进入 Claude Code：

~~~text
                    ┌→ subscription OAuth
Claude Code runtime ┤
                    └→ API Gateway
~~~

于是 agent loop、session / resume、tools、Context Plan、Continuity 等能力继续只有一套。

provider 主要负责回答一个问题：

> Claude Code 这次应该通过哪条线路访问模型？

---

## 8.2 双 Runtime 的优势

双 Runtime 很适合已经拥有成熟 API runtime 的项目。

### API 路径更直接

API 模式可以直接走：

~~~text
后端
 ↓
Messages API
 ↓
模型
~~~

后端可以精确控制：

- 请求体；
- streaming；
- `max_tokens`；
- thinking；
- prompt cache；
- tool loop；
- retry；
- timeout；
- usage 统计。

如果系统本来就围绕 Messages API 构建，这种控制力很有价值。

### 两条线路可以独立演进

例如：

~~~text
API → 第三方 Gateway
订阅 → Claude Code
~~~

其中一边升级、故障或者行为改变时，另一边受到的影响相对有限。

如果产品计划长期支持很多模型供应商，或者 API 路径里存在大量 Claude Code 无法表达的 provider-specific 能力，保留独立 runtime 会更灵活。

### API 兼容要求更低

一个 Gateway 只要能满足你自己的 Messages API 客户端，就可能已经够用。

Single Runtime 要求更高：Gateway 还得让 Claude Code 自己满意，包括 streaming、tool use、多轮 agent loop、thinking、cache、session 等行为。

因此有些“普通 API 调用没问题”的 Gateway，在双 Runtime 下可以正常使用，在 Single Runtime 下却可能过不了完整验收。

### 调试边界更清楚

当 API runtime 完全由自己控制时，可以直接记录每一轮请求和响应，定位某个字段为什么错、某个 tool result 为什么被拒绝。

Claude Code 内部行为升级以后，双 Runtime 的 API 路径仍然可以保持原来的实现。

---

## 8.3 双 Runtime 的维护成本

我们的项目后来开始觉得重，主要发生在“工具已经共用，但工具上面的运行时仍然有两套”之后。

### tool loop 会重复

工具本身可以只有一份：

~~~text
search_memory
send_email
create_file
...
~~~

但 API 还需要自己维护：

~~~text
model
 ↓
tool_use
 ↓
后端执行
 ↓
tool_result
 ↓
再次请求 model
~~~

Claude Code 同时拥有自己的 agent loop。

于是每增加一种工具行为，都需要确认两条路是否保持一致。

简单工具通常没问题，复杂情况会逐渐出现差异，例如：

- 工具调用后立即结束这一轮；
- 用户确认后继续；
- 一轮连续调用多个工具；
- 流中途断开；
- 工具有副作用，需要防重复执行；
- tool use / tool result 在历史里的排列方式不同。

### 上下文入口容易逐渐分叉

我们的项目已经有自己的 ContextPlan、Continuity、换窗和历史登记。

当 API 拥有单独 runtime 后，很自然会继续补出：

~~~text
API 专用 history projection
API 专用 ContextPlan adapter
API 消息 binding
API 后台生成路径
~~~

单个模块看起来都不大，但时间久了以后，同一个“这一轮应该看到什么历史”的问题会有两个入口。

以后改 ContextPlan 时，就要同时问：

> Claude Code 路径改了吗？
>
> API 路径有没有跟上？

这种维护成本通常不会在第一天出现，而是随着系统功能增加慢慢累积。

### session 语义需要自己对齐

Claude Code 原生拥有 session / resume。

API runtime 如果也要连续多轮，可以选择每次从数据库重新组 history，也可以自己维护 session-like 状态。

这两种方式都能实现，但只要产品要求两个 provider 随时切换，就要决定：

- 两边以谁的历史为准；
- 当前会话是否还能继续；
- 切换后缓存怎么办；
- 中途工具结果怎么回放；
- 换窗后的第一轮如何恢复；
- 某边失败以后是否允许另一边接手。

成熟的双 Runtime 系统完全可以把这些问题处理好，只是它们确实需要实现和测试。

### 后台任务也可能长出第二条路

我们当时有正式 Chat，也有 Continuity Chunk、压缩等后台生成。

很容易形成：

~~~text
订阅 → Claude Code -p
API  → Relay API
~~~

这样一来，模型身份冻结、provider authority、错误处理、usage、兼容性又需要考虑两套 executor。

---

## 8.4 Single Runtime 的优势

我们最后选择 Single Runtime，最主要的原因是现有系统已经高度依赖 Claude Code runtime。

### agent loop 只保留一份

API Gateway 放到 Claude Code 底下以后：

~~~text
                    ┌→ OAuth
前端 → Claude Code ─┤
                    └→ API Gateway
             ↓
        backend tools
~~~

tool use、tool result、多轮调用继续交给 Claude Code。

后端原生工具仍然只有一份，也不需要另外维护 API tool loop。

### session / resume 继续沿用

底层 provider 改变以后，上层仍然是 Claude Code。

因此已有的：

- resident；
- session；
- resume；
- staged resident；
- ContextPlan；
- Continuity；
- 换窗；

都可以沿用原来的生命周期。

对已经围绕 Claude Code 做了很多连续性工程的项目，这一点会明显减少重复实现。

### 上下文系统与 provider 解耦

我们最终更希望结构保持成：

~~~text
数据库
 ↓
ContextPlan / Continuity
 ↓
Claude Code runtime
 ↓
OAuth / API Gateway
~~~

“这一轮应该看到哪些历史”继续由同一套上下文系统决定。

provider 只影响最下层模型请求从哪里出去。

这个边界一旦稳定，新增一个兼容 Gateway 时，通常不需要再给 ContextPlan 写一套新的投影。

### 后台任务可以统一

如果 Gateway 能稳定运行 Claude Code 的 `-p`，后台生成也可以继续复用同一个入口：

~~~text
background job
      ↓
claude -p
      ↓
OAuth / API Gateway
~~~

正式 Chat 和后台任务至少共享同一种 runtime 语义。

---

## 8.5 Single Runtime 的代价

Single Runtime 简化了自己的代码，同时把一部分要求交给 Claude Code 和 Gateway。

### Gateway 必须真的兼容 Claude Code

普通文本请求成功还不够。

至少应该逐步验证：

~~~text
文本
 ↓
stream
 ↓
tool use
 ↓
多轮 agent loop
 ↓
thinking
 ↓
cache
 ↓
session / resume
~~~

我们真正决定采用这条路之前，先实测了普通 `claude -p`，随后又让 Claude Code 通过 Gateway 完整执行了一次 Bash 工具回合。

工具回合成功以后，才说明：

~~~text
Claude Code
 ↓
Gateway
 ↓
model tool_use
 ↓
Claude Code 执行工具
 ↓
tool_result
 ↓
Gateway
 ↓
最终回复
~~~

整条 agent loop 能闭合。

### 更依赖 Claude Code 的版本行为

双 Runtime 的 API 请求体由自己控制。

Single Runtime 下，Claude Code 会参与：

- 请求格式；
- system 内容；
- 工具协议；
- session 文件；
- cache 行为；
- 部分 thinking 行为。

Claude Code 升级后，最好重新跑兼容性探针。

### 直接控制 API 的能力会减少

如果你特别需要精细控制某家 API 的独有字段，Claude Code 未必能完整暴露这些能力。

这类项目继续保留独立 API runtime 可能更合适。

### 冷启动和基础上下文会更重

Claude Code 自己有 runtime system、工具信息和运行环境。

短短一句用户输入，并不意味着请求本身也很小。

使用按量 API 时，需要认真观察 Gateway 的真实账单、cache creation 和 cache read，再决定长期成本是否合适。

---

## 8.6 Provider 切换：两种架构处理的问题不同

双 Runtime 切换时，核心问题通常是：

> 两套 runtime 的历史是否还对得上？

可能需要 checkpoint、history hash、recast 或重新组历史。

Single Runtime 切换时，我们主要处理：

> 当前活着的 Claude Code 进程还是用旧 provider 启动的。

环境变量在进程启动时已经确定，不能指望一个活着的 resident 自动换掉 OAuth 或 Gateway Key。

所以我们的做法很简单：

~~~text
provider identity changed
        ↓
retire old resident
        ↓
next turn cold start
        ↓
new provider env
~~~

这里的关键点是把 provider 差异尽量限制在进程边界。

---

## 8.7 我们为什么从双 Runtime 收到 Single Runtime

我们早期已经把工具放在后端共用了，因此最初的双通道并没有重复实现工具本身。

重复主要出现在工具上方：

~~~text
API
 ↓
API tool loop
 ↓
API history / ContextPlan 接入
 ↓
共享工具


Claude Code
 ↓
CC agent loop
 ↓
CC session / ContextPlan
 ↓
共享工具
~~~

随着 ContextPlan、Continuity、resident、后台压缩等能力增加，我们需要持续保证两条运行路径理解同一份历史、同一份模型身份和同一份连续性状态。

其中一些旧 API 消息后来还需要一次性补做历史 Context mapping，才能完整进入 canonical Continuity。

当我们确认第三方 Gateway 可以直接作为 Claude Code 的底层模型入口，并且真实工具回合也能完整跑通以后，继续维护第二套 agent runtime 的收益就变小了。

于是最后收成：

~~~text
                         ┌→ subscription OAuth
frontend → Claude Code ──┤
                         └→ API Gateway
               ↓
        same tools / session
        ContextPlan / Continuity
~~~

这只是我们的项目条件下更合算的选择。

如果 API runtime 已经非常成熟、Claude Code 只占系统的一小部分，或者产品需要支持大量非 Anthropic-compatible provider，双 Runtime 依然可能更合适。

---

## 8.8 怎么选

可以先问自己几个问题。

| 情况 | 更值得优先考虑 |
|---|---|
| 已经有成熟的 API tool loop 和 session 系统 | 双 Runtime |
| 需要很多 provider-specific API 能力 | 双 Runtime |
| Gateway 只保证普通 Messages API 兼容 | 双 Runtime |
| Claude Code 只是可选的一条渠道 | 双 Runtime |
| 工具、session、换窗已经高度依赖 Claude Code | Single Runtime |
| API 和订阅希望共用同一套 ContextPlan / Continuity | Single Runtime |
| 不想长期维护两个 agent loop | Single Runtime |
| Gateway 已通过 Claude Code 的真实工具和 session 验收 | Single Runtime |

如果项目刚开始，建议先选最符合现有系统的一条，不必为了“以后也许会需要”同时建设两套完整 runtime。

如果项目已经是双 Runtime，也没有必要仅仅为了形式统一就重构。

只有当重复维护已经变成真实成本，并且 Gateway 的 Claude Code 兼容性已经通过实测时，Single Runtime 才真正有价值。

---

## 8.9 最后一个经验

我们这次最大的体会可以概括成一句话：

> **先判断差异到底属于“模型来源”，还是已经真的需要“两套运行时”。**

如果两条线路最终都需要相同的：

~~~text
tools
session
context
memory
continuity
window lifecycle
~~~

那么把 provider 差异压到运行时边界，通常会比较省维护。

如果两条线路需要的能力本来就不同，让它们各自演进也完全合理。

两种架构都能做得很好。真正值得比较的是：**哪一种让你的项目少维护一份没有必要重复的状态。**

