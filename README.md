# 把第三方 Anthropic API 接进 Claude Code

> **实测环境**
>
> - 实测日期：2026-11-13
> - Claude Code：2.1.280
> - Gateway 类型：Anthropic Messages-compatible 第三方 API Gateway
> - 实测服务：Poke
> - 已通过：最小 `claude -p`、真实 Bash 工具回合
>
> 这里只记录当时真实跑通的环境。Claude Code 和 Gateway 都会更新，版本变化后建议重新跑第 2、3 篇的两个探针。

事情起因其实特别普通。

我原本有两条路：一边用 Claude Code，一边直接调第三方 Anthropic API。工具早就放在自己的后端共用了，可再往上一层看，API 还要自己维护请求循环，Claude Code 又有自己的 agent loop、session 和 `-p`。

于是冒出了一个很朴素的问题：

**既然 Claude Code 本来就能换 API 地址，那第三方 API 能不能直接塞到 Claude Code 底下？**

试了一下——可以。普通 `-p` 跑通了，真实工具回合也完整跑通了，于是有了这份教程。

它主要讲四件事：

1. **怎么把第三方 Anthropic-compatible API 临时接进 Claude Code**，先不碰你原来的长期配置；
2. **怎么用 `claude -p` 和真实工具调用一步一步验收**，别只看“能不能回一句话”；
3. **怎么让 API 和订阅共用同一套 Claude Code runtime**，后端原生工具继续只保留一份；
4. 以及接入时容易撞到的 **Base URL、鉴权、缓存、provider 切换和费用显示**。

这不是什么新框架，也没有神奇黑科技。**这套做法很普通。** 只是中文资料里很少有人把“最小测试 → 工具回合 → 接进项目 → 长期运行”完整写成一条可以照着走的路，所以干脆整理下来。

第一次写教程，希望它至少做到一件事：**不用先懂完整架构，也能从第一步开始亲手验证。**

> **提醒：** Claude Code、Anthropic 条款和第三方 Gateway 都可能变化。这里记录的是技术接法和实测方法，实际使用请以你当时看到的官方条款和服务商规则为准。

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
| [1. 开始之前：准备好三样东西](docs/01-准备.md) | Claude Code、Gateway、Key 和 Base URL |
| [2. 第一关：最小 `-p` 测试](docs/02-最小-p测试.md) | 验证 Claude Code → Gateway → 模型 |
| [3. 第二关：工具调用测试](docs/03-工具调用测试.md) | 真正跑一次完整工具回合 |
| [4. 接进自己的项目](docs/04-接进自己的项目.md) | 保留后端工具，只统一 runtime |
| [5. Provider 切换与长期运行](docs/05-切换与长期运行.md) | resident、session、后台 `-p` |
| [6. 兼容性与常见坑](docs/06-兼容性与常见坑.md) | streaming、thinking、cache、鉴权 |
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
