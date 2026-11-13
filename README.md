# 把第三方 Anthropic API 接进 Claude Code

事情起因其实特别普通。

我原本有两条路：一边用 Claude Code，一边直接调第三方 Anthropic API。工具早就放在自己的后端共用了，但往上看，API 还要自己跑一套请求循环，Claude Code 又有自己的 agent loop、session 和 `-p`。

然后就冒出了一个很朴素的问题：

**既然 Claude Code 本来就能换 API 地址，那第三方 API 能不能直接塞到 Claude Code 底下？**

试了一下——能。至少在实际测试过的 Gateway 上，普通 `-p` 能跑，工具回合也能完整走完。

所以有了这个仓库。

它主要讲四件事：

1. **怎么把第三方 Anthropic-compatible API 临时接进 Claude Code**，先不碰你原来的配置；
2. **怎么用 `claude -p` 和真实工具调用一步一步验收**，别只看“能回一句话”；
3. **怎么把 API 和订阅放在同一套 Claude Code runtime 下面**，工具继续复用自己的后端；
4. 以及实际接入时很容易撞到的 **Base URL、鉴权、缓存、resident、费用显示这些坑**。

这不是一个框架，也没有什么神奇黑科技。说到底就是把 Claude Code 已经提供的能力接好，再认真验证第三方 Gateway 到底兼不兼容。

**这套做法很普通。** 我只是发现中文资料里很少有人把“最小测试 → 工具回合 → 接进项目 → 长期运行”完整写成一条能照着走的路，所以干脆整理下来。第一次写教程，尽量让每一步都能亲手验证，不要求读者先看懂一堆架构图。

> **先提醒一句：** 第三方 Gateway、Claude Code 和订阅相关规则都可能变化。实际使用时请以你当时看到的 Anthropic 条款和服务商规则为准。这个仓库只记录技术接法和实测方法。

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

**工具不用重写。**  
**API 也不用再单独养一套 agent runtime。**

## 读之前

你最好已经有：

- 一个能正常运行的 **Claude Code**；
- 一个第三方 **Anthropic-compatible API Gateway**；
- 一枚测试 Key 和明确的模型名；
- 如果要接进自己的项目，最好已经有自己的后端工具 / MCP / 数据库。

如果你只想验证“某家 API 能不能给 Claude Code 用”，看到第 3 篇就够了。

## 从这里开始

| 章 | 这一篇做什么 |
|---|---|
| [1. 开始之前：准备好三样东西](docs/01-准备.md) | 确认 Claude Code、Gateway、Key 和 Base URL |
| [2. 第一关：最小 `-p` 测试](docs/02-最小-p测试.md) | 只验证 Claude Code → Gateway → 模型 |
| [3. 第二关：工具调用测试](docs/03-工具调用测试.md) | 真正跑一次 tool use → tool result → 最终回复 |
| [4. 接进自己的项目](docs/04-接进自己的项目.md) | 保留后端原生工具，只统一 Claude Code runtime |
| [5. Provider 切换与长期运行](docs/05-切换与长期运行.md) | resident、session、后台 `-p`、切换时怎么处理 |
| [6. 兼容性与常见坑](docs/06-兼容性与常见坑.md) | streaming、thinking、cache、鉴权、费用显示 |
| [7. 自测清单](docs/07-自测清单.md) | 上线前从头到尾打一次勾 |

## 一句话版本

给 Claude Code 子进程一套明确的：

```text
ANTHROPIC_BASE_URL
ANTHROPIC_AUTH_TOKEN
ANTHROPIC_MODEL
```

先用最小 `claude -p` 验证文本，再跑一次真实工具回合。两关都通过以后，再考虑把它接进自己的长期 runtime。

如果你的工具本来就在后端，继续让它们待在那里；**这次要统一的是 Claude Code 上面的 agent loop、session 和上下文入口，底下只换 provider。**

## 当前实测到什么程度

目前真实跑过的结果包括：

- **第三方 Gateway → Claude Code `-p`：成功**
- **第三方 Gateway → Claude Code → Bash 工具回合：成功**
- 另一家 Gateway 的普通 `-p`：成功
- 余额不足的 Gateway：请求能到服务端，但这只能证明链路可达，不能算完整兼容

所以正文里会一直区分：

**“能请求到”** 和 **“能完整跑 Claude Code agent loop”**。

## 关于费用

Claude Code 输出里的 `costUSD` 可能按它掌握的列表价估算。

第三方 Gateway 自己可能有不同的价格、折扣、货币和计费方式，所以**最终请看服务商自己的实际账单**，别直接把 Claude Code 显示的数字当成实扣。

## Attribution

**GPT 全权代笔。**

内容来自真实接入、真实 `-p` 和工具回合测试，再把过程整理成人能一步一步照着做的版本。
