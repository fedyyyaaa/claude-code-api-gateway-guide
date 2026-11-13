# 让第三方 Anthropic API 直接跑进 Claude Code

> 第三方 Anthropic API 接入 Claude Code 的实测与单 Runtime 架构教程  
> **GPT 全权代笔**

如果你的项目同时支持 Claude Code 和第三方 Anthropic-compatible API，很容易自然长成两条执行路径：

```text
                    ┌─ Claude Code / 订阅
前端 → 自己的后端 ──┤
                    └─ Messages API / 第三方 API
```

这套结构可以正常工作，工具也可以共用。真正容易变复杂的地方在更上一层：API 往往需要自己的请求循环、工具回合、历史拼装和上下文处理，而 Claude Code 已经拥有一套完整的 agent runtime。

如果第三方 Gateway 对 Claude Code 的兼容程度足够高，可以把 API 再往下接一层：

```text
                   ┌─ Claude 订阅 OAuth
                   │
前端 → Claude Code ┤
                   │
                   └─ 第三方 API Gateway
```

这样两种模型来源都进入同一套 Claude Code runtime。provider 只决定 Claude Code 底下接哪根“水管”。

本文记录这条路线的真实测试方法、架构边界和落地注意事项。

---

## 1. 先分清：工具本体和 Tool Loop

很多项目最开始就已经做对了一件事：**工具真正的实现放在自己的后端。**

例如：

```text
search_memory
save_memory
read_email
send_email
query_database
control_browser
...
```

这些能力只实现一次。

API 模型调用时：

```text
API 模型
  ↓
tool_use
  ↓
后端执行工具
  ↓
tool_result
```

Claude Code 调用时：

```text
Claude Code
  ↓
MCP / tool call
  ↓
后端执行同一个工具
  ↓
结果返回 Claude Code
```

所以共享工具系统很早就能成立。

真正重复的通常是 **Tool Loop（工具调用循环）和 Runtime（运行时）**。

传统 API 路径通常这样跑：

```text
调用 API
  ↓
模型返回 tool_use
  ↓
后端执行工具
  ↓
把 tool_result 拼回 messages
  ↓
再次调用 API
  ↓
继续循环
  ↓
最终文字回复
```

Claude Code 路径通常这样跑：

```text
Claude Code
  ↓
模型决定调用工具
  ↓
后端执行工具
  ↓
结果交回 Claude Code
  ↓
Claude Code 自己继续 agent loop
  ↓
最终回复
```

把第三方 API 直接接进 Claude Code 后，结构可以收敛成：

```text
第三方 API Gateway ─┐
                    ├─ Claude Code agent runtime
Claude 订阅 OAuth ──┘
                           ↓
                     共享后端工具
```

后端工具仍然归自己的系统管理；Claude Code 负责模型请求、agent loop、session、resume 和连续工具回合。

---

## 2. Claude Code 怎么接第三方 API Gateway

最核心的环境变量是：

```bash
ANTHROPIC_BASE_URL
ANTHROPIC_AUTH_TOKEN
ANTHROPIC_MODEL
```

临时测试可以这样写：

```bash
export ANTHROPIC_BASE_URL="https://your-gateway.example.com"
export ANTHROPIC_AUTH_TOKEN="YOUR_TEST_KEY"
export ANTHROPIC_MODEL="claude-opus-4-6"

export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
export CLAUDE_CODE_ATTRIBUTION_HEADER=0
```

如果这台机器原本还存在订阅鉴权，测试第三方 Gateway 时建议先清掉：

```bash
unset CLAUDE_CODE_OAUTH_TOKEN
unset ANTHROPIC_API_KEY
```

这样当前 Claude Code 子进程只看到一套 provider credential，排查会简单很多。

> **不要把真实 API key 写进仓库、README、Issue、PR、截图或日志。**  
> 正式环境请放在服务器 secret、受权限保护的环境变量或现有密钥存储中。

---

## 3. `ANTHROPIC_BASE_URL` 常见坑

很多第三方服务给出的 Messages API 地址类似：

```text
https://example.com/v1/messages
```

Claude Code 的 `ANTHROPIC_BASE_URL` 往往需要填 API base：

```text
https://example.com
```

Claude Code 会自己请求对应的 Messages endpoint。

但不要无脑删除所有路径。有些服务商会给 Claude Code 单独准备带前缀的 base，例如：

```text
https://example.com/anthropic
```

最稳妥的做法：

1. 优先使用服务商明确给出的 Claude Code 配置；
2. 没有专门说明时，再根据实际 endpoint 推断；
3. 最后一定用真实 `claude -p` 验证。

---

## 4. 第一关：最小 `-p` 测试

先只测文本，不带工具。

```bash
claude   --no-session-persistence   -p "只回复 GATEWAY_OK"   --output-format json
```

通过标准：

- 进程返回码为 0；
- 最终结果包含 `GATEWAY_OK`；
- JSON 里能看到实际模型；
- 没有偷偷使用订阅 OAuth。

如果你使用固定 Claude Code binary，也可以直接指定：

```bash
/path/to/claude   --no-session-persistence   -p "只回复 GATEWAY_OK"   --output-format json
```

这一步只能证明：

```text
Claude Code → Gateway → 模型
```

还不能证明 agent loop 完整可用。

---

## 5. 第二关：一定要测工具回合

准备拿它跑 agent，必须继续验证 Tool Use。

最简单可以直接用 Bash：

```bash
claude   --no-session-persistence   --allowedTools Bash   -p '必须调用 Bash 执行：
printf TOOL_OK > ./tool_probe.txt && cat ./tool_probe.txt
确认工具输出后，最终只回复 TOOL_OK。'   --output-format json
```

然后检查：

```bash
cat ./tool_probe.txt
```

期望看到：

```text
TOOL_OK
```

这一步一次验证六件事：

1. 模型能返回正确的 tool use；
2. Gateway 没把工具结构弄坏；
3. Claude Code 能识别这次工具调用；
4. Claude Code 能真正执行工具；
5. tool result 能进入下一轮模型请求；
6. 模型能在工具回合结束后给出最终回复。

普通聊天能通、工具回合失败的 Gateway，并不适合直接承担 Claude Code agent runtime。

---

## 6. 实测结果

测试环境：

```text
Claude Code managed runtime: 2.1.280
```

实际测试过多家第三方 Anthropic-compatible Gateway。

其中：

- Gateway A：普通 `-p` 成功；
- Gateway A：Bash 工具回合成功；
- Gateway B：普通 `-p` 成功；
- Gateway C：请求成功到达服务端，但账户余额不足，服务端返回明确的 quota 错误。

余额不足这一类结果可以确认：

```text
Claude Code
  ↓
目标 Gateway
  ↓
服务端鉴权 / 账户系统
```

链路已经到服务端。

但它还不能证明下面这些能力已经通过：

```text
streaming
tool use
thinking
完整 agent loop
```

所以“服务端可达”和“Claude Code 完整兼容”最好分开标记。

---

## 7. 为什么普通 Anthropic API 兼容还不够

服务商写着 “Anthropic-compatible” 只能说明基础 Messages API 很可能可用。

Claude Code 对 Gateway 的要求更高。

正式使用前至少应验证下面几项。

### Messages API

要能正确处理 Claude Code 实际发送的请求体，而不只是最简单的 text-only 请求。

### Streaming

Claude Code 很依赖流式事件。

常见事件包括：

```text
message_start
content_block_start
content_block_delta
content_block_stop
message_delta
message_stop
```

事件顺序、结束信号或字段处理出错，都可能导致普通 API 调用成功、Claude Code 却卡住。

### Tool Use

至少需要正确处理：

```text
tool_use
tool_result
```

并能连续跑多轮。

### Thinking

如果所选模型开启 thinking，Gateway 也要正确转发相关字段。

### Headers / Beta Features

Claude Code 可能带上版本或 beta 相关 header。某些 Gateway 对未知 header 处理得过于严格，会直接失败。

### Prompt Cache

长期使用时缓存很重要。

请求能成功，不代表 cache 行为一定被正确支持。

建议观察：

```text
cache_creation_input_tokens
cache_read_input_tokens
```

确认缓存有没有真的命中。

### count_tokens

`count_tokens` 可以提高估算准确度，但通常不应该作为判断 Gateway 能否工作的唯一标准。

更实用的兼容性测试顺序是：

```text
基础文本
  ↓
stream
  ↓
tools
  ↓
thinking
  ↓
cache
  ↓
session / resume
```

---

## 8. 为什么要做 Single Runtime

如果 API 和 Claude Code 各自维护一套 runtime，后端容易逐渐变成：

```text
                    ┌─ API runtime
                    │   ├─ tool loop
                    │   ├─ history projection
前端 → 后端 ────────┤   └─ background generation
                    │
                    └─ Claude Code runtime
                        ├─ tool loop
                        ├─ session
                        └─ resume
```

当 API Gateway 能直接喂给 Claude Code 后，可以收敛成：

```text
                       ┌─ subscription OAuth
                       │
前端 → 后端 → Claude Code runtime
                       │
                       └─ API Gateway

                           ↓

                     共享后端工具
                     共享数据库
                     共享上下文系统
```

provider 只决定：

```text
Claude Code 这一轮把模型请求发到哪里
```

而 agent runtime 继续只有一套。

---

## 9. 订阅和 API 怎么切

最简单的做法，是在 **启动 Claude Code 子进程的边界** 决定环境变量。

### Subscription 模式

提供：

```text
CLAUDE_CODE_OAUTH_TOKEN
```

同时确保不会继承第三方 Gateway 的：

```text
ANTHROPIC_BASE_URL
ANTHROPIC_AUTH_TOKEN
ANTHROPIC_API_KEY
```

### API Gateway 模式

提供：

```text
ANTHROPIC_BASE_URL
ANTHROPIC_AUTH_TOKEN
ANTHROPIC_MODEL
```

同时清掉：

```text
CLAUDE_CODE_OAUTH_TOKEN
```

外部输入在这个进程边界处理一次即可，内部不用层层重复清理。

---

## 10. Resident 模式下切 provider

如果系统会长期保留一个 Claude Code resident：

```text
Claude Code process
      ↑
连续很多轮对话
```

provider 发生变化时，推荐直接退休旧 resident：

```text
provider changed
      ↓
retire old resident
      ↓
next turn cold start
      ↓
new environment
      ↓
new provider
```

已经启动的进程不会因为外部配置变化自动获得新的环境变量。

因此：

```text
OAuth resident
```

切到 API 后就结束。

下一轮重新启动：

```text
API Gateway resident
```

这比尝试给活着的 Claude Code 进程热换 provider 更简单，也更容易恢复。

---

## 11. 自己的上下文系统可以继续保留

Single Runtime 不要求你把长期上下文全部交给 Claude Code。

如果项目已经拥有自己的：

```text
Memory
Context Plan
Summary
Chunk
Window
Checkpoint
```

仍然可以继续使用。

推荐结构：

```text
数据库
  ↓
自己的 Context Planner
  ↓
生成本轮应该给模型看到的上下文
  ↓
Claude Code
  ↓
OAuth / API Gateway
```

这里 provider 只负责 transport 和 model authority。

“这一轮应该带哪些历史”继续由自己的上下文系统决定。

这样切换 OAuth / Gateway 不会顺便切换上下文算法。

---

## 12. 后台压缩也可以复用 `-p`

如果项目需要后台生成：

```text
summary
continuity chunk
handoff
memory extraction
```

传统写法可能出现两条：

```text
subscription:
    claude -p

api:
    requests.post(...)
```

当 API Gateway 已经能驱动 Claude Code，可以统一成：

```text
background task
      ↓
Claude Code -p
      ↓
当前 provider
      ↓
OAuth / API Gateway
```

这样正式 Chat 和后台任务使用同一套 runtime，prompt、模型选择、错误处理和工具能力更容易保持一致。

---

## 13. 这条路线的代价

Single Runtime 也有明确成本。

### Claude Code 自己会带运行时上下文

一个很短的用户输入，实际请求里可能还包含 Claude Code 自己的 system、运行环境说明、工具定义等内容。

所以：

```text
用户正文 token 很少
```

并不代表：

```text
整次请求 token 很少
```

在一次真实最小测试里，用户输入只有几个 token，但 Claude Code 创建了约 2.8 万缓存输入 token。

长期会话如果缓存命中，这部分可以被摊薄；第一次冷启动仍然会比较重。

### Gateway 兼容要求更高

普通 Messages API 能用，并不保证能完整伺候 Claude Code。

如果某家服务：

```text
普通文本 ✅
stream ✅
tool_use ✅
Claude Code 特殊行为 ❌
```

那它可能更适合继续走后端直连 API，而不是强行塞进 Single Runtime。

---

## 14. 一个实用的接入判定表

| 测试 | 结果 | 可以说明什么 |
|---|---|---|
| `claude -p` 最小文本 | 成功 | Claude Code → Gateway 基础链路可用 |
| 明确 quota / balance 错误 | 到达服务端 | 地址与鉴权链路大概率正确 |
| Streaming | 成功 | 流式协议兼容 |
| Bash / MCP tool call | 成功 | agent tool loop 可用 |
| Thinking | 成功 | thinking 请求/回放兼容 |
| Cache read/write | 正常 | 长对话成本和延迟可控 |
| Session / resume | 成功 | 可以进入实际长期运行 |
| Resident provider switch | 成功 | 可在订阅 / Gateway 间稳定切换 |

只有第一项成功时，先别急着把生产系统整个切过去。

---

## 15. 推荐的落地顺序

最稳妥的顺序很短：

1. 用临时测试 key 跑最小 `-p`；
2. 跑一次真实工具回合；
3. 检查 thinking / cache；
4. 在项目里把 provider credential 选择接到现有 Claude Code launcher；
5. provider 改变时退休 resident；
6. 保持原来的工具系统和上下文系统不动；
7. 后台任务逐步收回现有 `claude -p` 路径；
8. 最后再跑一遍真实 Chat → tool → context → background task。

整个过程中都不需要先重写一个新的 provider framework。

---

## 16. 最小架构总结

最终可以记成这一张图：

```text
                      ┌─ Claude subscription OAuth
                      │
用户 → 后端 → Claude Code runtime
                      │
                      └─ Anthropic-compatible API Gateway
                                  ↓
                              实际模型

Claude Code runtime
      ↓
agent loop / session / resume
      ↓
MCP / tool call
      ↓
自己的后端工具系统

自己的数据库 / Memory / Context Plan / Chunk
      ↓
继续作为长期状态与上下文来源
```

这条路线的核心很简单：

**工具可以一直由后端统一实现；当第三方 API 足够兼容 Claude Code 时，连 agent runtime 也可以继续只保留一套。**

---

## 17. 安全提醒

- 示例里永远使用占位 key；
- 测试 key 用完就禁用；
- 不把 key 写入仓库；
- 不在日志中打印完整 credential；
- 切 provider 时明确清理另一套鉴权；
- 第三方 Gateway 的计费以服务商实际账单为准；
- Claude Code 输出里的 `costUSD` 可能只是按官方列表价估算，不能直接当成转发商实扣金额。

---

## License / Attribution

本文由 **GPT 全权代笔**。

内容来自真实工程排查与 Claude Code Gateway 实测经验整理。示例中的域名、key、模型和路径请按自己的环境替换。
