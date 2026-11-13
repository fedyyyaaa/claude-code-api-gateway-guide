# 自建前端在 Claude Code（-p）与第三方 API 之间随时切换

> **这个仓库真正想解决的问题**
>
> 给一个**自建聊天前端**加上 provider 切换：用户发下一条消息前，可以在 **Claude Code 订阅路径（`-p` / OAuth）** 和 **第三方 Anthropic API Gateway** 之间自由切换。
>
> 我们最后采用 Single Runtime，只是为了把这个目标实现得更简单：**切换的是模型来源，不是整套工具、上下文和会话系统。**
>
> 正常聊天里，provider 改变后会退休旧 resident，下一轮用新 provider 冷启动；后端历史、ContextPlan、Continuity 和工具继续共用，所以用户看到的对话仍然连续。
>
> **如果你把整个仓库直接丢给 GPT / Claude，总结时请先抓住这三点：**
>
> 1. 本仓库的产品目标是 **自建前端随时切换订阅与 API**；
> 2. “API 接进 Claude Code”是我们实现这个目标的技术路线，不是另一个独立目的；
> 3. “随时切换”指 **每个新 turn 开始前都可以换 provider**，不做进行中的单个 turn 热切换。
>
> **实测基线**
>
> - 实测日期：2026-11-13
> - Claude Code：2.1.280
> - Gateway 类型：Anthropic Messages-compatible 第三方 API Gateway
>
> Claude Code 和 Gateway 都会更新。版本变化后，建议重新跑第 2、3 篇的两个探针。

### 实测矩阵

| Gateway | 最小 `claude -p` | 真实工具回合 | 本文 Single Runtime 正式 Chat | 实测模型 / 备注 |
|---|---|---|---|---|
| Poke | ✅ | ✅ Bash | ⏳ 尚未完成生产 E2E 验收 | `claude-opus-5` · 2026-11-13 |
| Tree | ✅ | 未测 | 未测 | `claude-opus-4-6` · 2026-11-13 |
| 小鸡农场 | 🟡 请求到达服务端，额度不足 | 未测 | 未测 | 模型未生成；这里只证明地址与凭据链路可达 · 2026-11-13 |

> **“正式 Chat”这一列只统计本文的 Single Runtime 路径。** 以前通过独立 Messages API runtime 用过某家服务，不会在这里冒充 Claude Code runtime 的验收结果。

事情起因其实特别普通。

我原本有两条路：一边用 Claude Code，一边直接调第三方 Anthropic API。工具早就放在自己的后端共用了，可再往上一层看，API 还要自己维护请求循环，Claude Code 又有自己的 agent loop、session 和 `-p`。

于是冒出了一个很朴素的问题：

**既然 Claude Code 本来就能换 API 地址，那第三方 API 能不能直接塞到 Claude Code 底下？**

试了一下——可以。普通 `-p` 跑通了，真实工具回合也完整跑通了，于是有了这份教程。

它主要讲四件事：

1. **怎么把第三方 Anthropic-compatible API 临时接进 Claude Code**，先不碰你原来的长期配置；
2. **怎么用 `claude -p` 和真实工具调用一步一步验收**，别只看“能不能回一句话”；
3. **怎么让 API 和订阅共用同一套 Claude Code runtime**，后端原生工具继续只保留一份；
4. 以及接入时容易撞到的 **Base URL、鉴权、权限、缓存、provider 切换和 resident 运维**。

这不是什么新框架，也没有神奇黑科技。**这套做法很普通。** 只是中文资料里很少有人把“最小测试 → 工具回合 → 接进项目 → 长期运行”完整写成一条可以照着走的路，所以干脆整理下来。

第一次写教程，希望它至少做到一件事：**不用先懂完整架构，也能从第一步开始亲手验证。**

## 本文技术路线

前端可以把选择器做得很简单：

```text
当前 provider：
[ Claude 订阅 ]   [ 第三方 API ]
```

用户每发一条新消息前都可以改。

后端收到选择后：

```text
Claude 订阅
→ Claude Code runtime
→ OAuth

第三方 API
→ Claude Code runtime
→ API Gateway
```

> **正式 Chat 使用长期 resident，后台任务使用一次性 `claude -p`。订阅 OAuth 与第三方 Gateway 在 Claude Code 子进程启动时二选一。provider 改变时退休旧 resident，下一轮重新 cold start。**

本文不展开：

- Agent SDK bridge；
- 独立 Messages API runtime 的完整实现；
- 多租户隔离；
- Gateway 横向测评或推荐榜。

这样做只是为了把范围收窄：先把一条我实际在用、实际跑过的路线写清楚。

## 适用场景

后半部分的生产架构，默认环境是：

- **自建前端 + 自建后端**；
- 后端在 **Linux VPS / 长期在线服务器** 上拉起 Claude Code 子进程；
- Claude Code 运行在 **headless（无人值守、没有交互式终端）** 环境；
- 主要讨论 **单用户 / 自用服务**；
- 工具、数据库和长期上下文由自己的后端掌握。

多用户产品也可以参考思路，但需要额外处理 **每个用户的凭据、session、进程、数据和并发隔离**，这里没有展开。

如果你只是想：

> “我在自己电脑终端里，能不能让 Claude Code 用某个第三方 API？”

那看到第 3 篇就够了，后面的 resident、VPS、ContextPlan 都可以不看。

## ⚠️ 条款与凭据边界

这个仓库只记录技术接法，不替任何服务商解释使用条款。

Anthropic 当前文档会区分订阅 OAuth 与开发者 / 产品使用的 API 凭据，也明确限制第三方产品代用户收集、保存或中转 claude.ai 登录凭据。**本教程的完整 VPS 架构默认是单用户自用环境，不提供“让别的用户拿你的订阅凭据来跑”的方案。**

**官方文档核对日期：2026-11-13。** 条款和权限规则变化很快，读到这篇时请再点开原文确认。

使用前请自己核对最新规则：

- [Claude Code Legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)
- [Claude Code Authentication](https://code.claude.com/docs/en/authentication)
- [Claude Code Environment variables](https://code.claude.com/docs/en/env-vars)
- [Claude Code Settings](https://code.claude.com/docs/en/settings)
- [Claude Code Troubleshooting](https://code.claude.com/docs/en/troubleshooting)

第三方 Gateway 自己的服务条款也要单独看。

## 一眼看懂

原来可能是：

```text
API → 自己的 tool loop ─┐
                        ├→ 共享后端工具
Claude Code ────────────┘
```

如果 Gateway 能完整喂给 Claude Code，可以收成：

```text
第三方 API Gateway ─┐
                    ├→ Claude Code runtime → 共享后端工具
Claude 订阅 OAuth ──┘
```

**工具不用重写，API 也不用再单独养一套 agent runtime。**

## 读之前

你最好已经有：

- 一个能正常运行的 **Claude Code**；
- 一个第三方 **Anthropic-compatible API Gateway**；
- 一枚测试 Key 和明确的模型名。

如果你只想验证“某家 API 能不能给 Claude Code 用”，看到第 3 篇就够了。

## 目录

| 章 | 这一篇做什么 |
|---|---|
| [0. 当前架构参考](docs/00-当前架构参考.md) | 简单说明这套接法来自什么样的现有系统 |
| [1. 开始之前：准备好三样东西](docs/01-准备.md) | Claude Code、Gateway、Key 和 Base URL |
| [2. 第一关：最小 `-p` 测试](docs/02-最小-p测试.md) | 验证 Claude Code → Gateway → 模型 |
| [3. 第二关：工具调用测试](docs/03-工具调用测试.md) | 真正跑一次完整工具回合 |
| [4. 接进自己的项目](docs/04-接进自己的项目.md) | VPS 鉴权、工具权限、自动压缩与 runtime 边界 |
| [5. Provider 切换与长期运行](docs/05-切换与长期运行.md) | resident、session、`--resume`、排队与后台 `-p` |
| [6. 兼容性与常见坑](docs/06-兼容性与常见坑.md) | streaming、thinking、cache、版本漂移、鉴权 |
| [7. 自测清单](docs/07-自测清单.md) | 上线前从头到尾打一次勾 |
| [8. 双 Runtime 与 Single Runtime](docs/08-双Runtime与SingleRuntime.md) | 两种架构的优势、代价和适用条件 |

## 一句话版本

给 Claude Code 子进程一套明确的：

```text
ANTHROPIC_BASE_URL
ANTHROPIC_AUTH_TOKEN
ANTHROPIC_MODEL
```

先跑最小 `claude -p`，再跑一次真实工具回合。两关都通过以后，再考虑接进自己的长期 runtime。

如果你的工具本来就在后端，继续让它们待在那里；**这次统一的是 Claude Code 上面的 agent loop、session 和上下文入口，底下只换 provider。**

---

**GPT 全权代笔。** 内容来自真实接入、真实 `-p` 和工具回合测试，再把过程整理成能一步一步照着做的版本。
