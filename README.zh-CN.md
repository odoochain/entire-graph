![entire-graph theme](docs/images/gh-repo-cover.png "entire-graph 封面图")

# Entire Graph 中文说明

编码智能体（coding agent）的时间，大量花在「动手改之前」——还在找代码在哪的时候。
Entire Graph 是 Entire CLI 的一个插件，它为智能体提供**单个 Git 仓库的预计算地图**：
带排序的代码检索，外加定义、调用者、类型、路由与变更影响范围，每一项都带
`file:line` 位置。内置分析器在本地用 tree-sitter 解析仓库，**不发起任何网络请求、
不调用模型、不查询 API key**——联网只发生在安装插件这一步。

每个仓库只需配置一次。之后，你的界面就是编码智能体本身：用自然语言提出一个代码
问题，智能体执行图查询、读取图指向的代码，并给出带引用的回答。下面是一段真实
抓取的示例。

> 本文是 [README.md](README.md) 的中文翻译，内容以英文版为准。

## 基准测试

在一项八系统对比的 LoCoMo 评测中（1,540 个问题，共享 reader 与 judge，每个对比
组的检索预算均为 200 条），Entire Graph 的检索引擎排名第一，且构建索引时**不消耗
任何模型 token**。评测时间为 2026-08-14，对应首次发布于
[v0.4.0](https://github.com/entireio/entire-graph/releases/tag/v0.4.0) 的检索路径
（[#104](https://github.com/entireio/entire-graph/pull/104)）。

| 系统 | LoCoMo | 建索引 token | 测试版本 |
| --- | --- | --- | --- |
| **entire-graph** | **94.74** | **0** | [v0.4.0](https://github.com/entireio/entire-graph/releases/tag/v0.4.0)（[#104](https://github.com/entireio/entire-graph/pull/104)） |
| [mem0](https://github.com/mem0ai/mem0) | 93.83 | 50.85M | commit [`4debc58`](https://github.com/mem0ai/mem0/commit/4debc58a83377b18be81ae1e5969a300736b2fac) |
| [cognee](https://github.com/topoteretes/cognee) | 92.86 | 12.35M | commit [`38eece5`](https://github.com/topoteretes/cognee/commit/38eece5bbb0cb9f5706fed908abd16dba0f5505e) |
| [bm25](https://github.com/dorianbrown/rank_bm25)（词法基线） | 91.88 | 0 | [0.2.2](https://github.com/dorianbrown/rank_bm25/releases/tag/0.2.2) |
| [codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp)（cmm） | 91.30 | 0 | [v0.9.0](https://github.com/DeusData/codebase-memory-mcp/releases#release-v0.9.0) |
| [graphify](https://github.com/Graphify-Labs/graphify) | 87.34 | 0 | [v0.9.43](https://github.com/Graphify-Labs/graphify/releases/tag/v0.9.43) |
| [letta](https://github.com/letta-ai/letta) | 84.68 | 不可推算 | [0.16.8](https://github.com/letta-ai/letta/releases/tag/0.16.8) |
| [supermemory](https://github.com/supermemoryai/supermemory) | 82.08 | 托管服务 | [server-v0.0.7-rc.2](https://github.com/supermemoryai/supermemory/releases/tag/server-v0.0.7-rc.2) |

完整方法、分类结果、撤回说明与复现步骤见
[benchmarks](docs/benchmarks.md)。

## 安装

Entire Graph 要求 **Entire CLI 0.10.0 或更高版本**，且 `PATH` 上有 **Git 2.36 或更高
版本**。Git 2.36 引入了 Entire Graph 所依赖的单会话对象协议：在读取对象内容之前先
判断对象类型。以下命令来自 [Entire CLI 安装指南](https://docs.entire.io/installation)
中的官方命令，该指南同时覆盖 Windows 及其他安装渠道。

macOS：

```sh
brew tap entireio/tap
brew trust entireio/tap
brew install --cask entire
```

Linux：

```sh
curl -fsSL https://entire.io/install.sh | bash
```

然后从插件索引**安装插件**并确认版本：

```sh
entire plugin install graph
entire graph version
```

`entire graph version` 打印出发布标签，说明当前使用的是带版本号的正式构建。
本地安装辅助脚本会输出源码检出目录的 Git 描述信息；未经版本化的原始构建会打印
`dev`（见 [operations](docs/operations.md)）。

## 为你的智能体启用

启用是按仓库生效的：

```sh
entire graph init-agents --repo .
```

该命令会创建或更新以下文件：

- `.entire/graph-agent.md`：智能体操作指南。每次成功执行都会**完整重新生成**，
  手工修改不会保留。
- `AGENTS.md` 与 `CLAUDE.md`：若不存在则创建；若已存在，则在
  `<!-- entire-graph:begin -->` 与 `<!-- entire-graph:end -->` 标记之间添加或替换
  一个受管区块。标记之外的文本会被保留。

请检查这三个文件，然后一起提交，以便这些指引对整个团队生效。

最后，**在该仓库中开启一个全新的智能体会话**。启用过程中处于打开状态的会话并没有
读到新的指引。

## 提出你的第一个问题

用你自己的话向智能体提一个关于仓库的问题。下面这段是真实抓取的会话记录，输入全部
被固定：已安装的 Entire Graph 发布版、`gorilla/mux` 在 commit `db9d1d0` 上的全新
克隆（此前没有任何智能体指引文件）、激活文件已提交、客户端为 Claude Code 2.1.233
（模型 Claude Fable 5）。提示词是：

> 不要修改任何文件，找出这个仓库里路由匹配是在哪里实现的、谁调用了它，以及如果它的
> 行为发生变化，最可能受影响的是什么。请引用相关源码。

该会话的工具调用按顺序如下（长命令为适应宽度做了换行，最后一条有截断）：

```text
Bash: entire graph search --repo . --profile full --query "route matching
      implementation for incoming request paths"
Bash: entire graph impact --repo . --symbol Route.Match
Bash: entire graph impact --repo . --symbol Router.Match
Bash: sed -n 145,230p mux.go
Bash: sed -n 40,120p route.go
Bash: grep -n "func.*Match\|matcher\b" route.go regexp.go | head -40
Bash: sed -n 185,225p regexp.go; grep -n "\.Match(" mux.go route.go middleware.go ...
```

第一次调用就是图检索。这正是激活指引在起作用：Claude Code 在会话启动时加载
`CLAUDE.md` 并解析其中对指南的引用，因此智能体在任何 grep 之前先用了图。检索返回
带排序的 JSON 证据；该查询的首个命中是 `route.go:237` 的 `Route.addMatcher`，紧随
其后的是 `regexp.go:41` 的 `newRouteRegexp`。随后智能体询问了影响范围。以下是它
收到的 `impact` 输出开头（除一行为适应宽度换行外，逐字原样）：

```text
Index: cache-hit (49ms) | Query: 0ms | Total: 50ms
Impact: Router.Match (mux.go:151) def=151 span=151-182 [method in Router]
Blast radius: 1 caller (1 direct, 0 transitive), 0 callees, 3 type consumers,
  1 data flow, 7 co-change files, 29 siblings.
Callers (1 direct, 0 transitive; who breaks if behavior changes):
- Router.ServeHTTP (mux.go:203, def :188)
```

这份抓取早于 ADR 0004 的安全修正。当前**默认的工作树查询会报告 `cache-miss`**；
基于提交树的查询在存在匹配条目时才可能报告 `cache-hit`。

只有在图查询之后，智能体才开始读源码，而且只围绕图给出的位置读取很窄的行区间。
它的回答沿着 `Router.Match`（`mux.go:151-182`）、`Route.Match`（`route.go:47-114`）
和 `routeRegexp.Match`（`regexp.go:189-209`）追踪了匹配流程，并指出一处改动会波及：
处理器分发与 404/405 的选择、经由 `setMatch` 的路由变量、由 `newRouteRegexp` 构建的
URL 反解，以及经过 `getAllMethodsForRoute` 的 CORS 中间件路径。它还暴露了一个图的
局限，并用源码做了核实：`impact --symbol Route.Match` 报告零个调用者，而源码检查
能找到两处直接调用点。

最后这一点正是你应当预期的工作关系：**图的输出是供智能体对照源码核查的证据，不是
神谕。** [支撑记录](docs/evidence/2026-08-16-mux-agent-session.md) 包含抓取条件、
相关的智能体与工具事件、完整的图命令输出，以及逐字的最终回答。

配置的每一层都有自己的成功信号。安装：`entire version` 与 `entire graph version`
都成功。激活：三个文件存在且标记完整。采纳：在新会话中，第一个「定位代码」的调用
是 `entire graph search`。如果智能体一上来就做宽泛 grep 或整文件探索，说明指南可能
没有加载或没有被遵循。请检查激活文件与客户端的指引视图，见
[agent activation](docs/agents.md)。落地性：回答引用的是智能体真正打开过的文件与
行号。

## 可以问什么

提示词就是界面。命令是智能体在底层执行的东西；你也可以直接调用它们来做人工检查、
调试或自动化。见[命令参考](docs/commands.md)。

| 目标 | 示例提示词 | 图命令 |
| --- | --- | --- |
| 找到实现 | 请求路由是在哪里实现的？ | `search` |
| 读取某个定义 | 给我看 `ResolveRoute` 的定义。 | `def` |
| 追踪调用者或被调者 | 谁调用了 `ResolveRoute`？ | `neighbors` |
| 评估影响范围 | 修改 `ResolveRoute` 会影响什么？ | `impact` |
| 评审一个分支 | 总结从 `main` 到 `HEAD` 的语义变更。 | `diff` |
| 导出完整图 | 把仓库的图导出为 NDJSON。 | `snapshot` |

## 工作树与缓存

交互式查询族（`search`、`def`、`explain`、`neighbors`、`impact`）默认读取**工作树**，
因此智能体能看到未提交的改动。加上 `--head` 则改为查询已提交树。批量流式命令
（`snapshot`、`symbols`、`edges`）与基于引用分析（`diff`、`commit`）默认使用已提交
状态。

查询可以写入派生的本地缓存，但**永不修改仓库文件**。默认的工作树查询总是重新构建，
既不加载也不写入缓存条目。`--head` 查询可以复用一个以已提交树和查询选项为键的
快照；修改 `.graphignore` 会选中另一个已提交树条目。

缓存状态在会报告它的输出格式中可见：默认 `search` JSON 带 `stats.index_cache_hit`
字段，`impact`/`neighbors` 的文本输出以 `Index: cache-hit` / `cache-miss` 行开头。
默认工作树查询报告 miss；匹配的 `--head` 查询可能 hit。`search --format text` 不报告
缓存状态。

`entire graph index` 只为已提交树（`--head`）查询预热，且默认 profile 为 `full`，
而裸 `search` 默认为 `fast`。因此一次默认的 `index` 运行**不会**预热默认的工作树
路径。查询族内部还有一个例外：`def` 与 `explain` 只有在设置了 `--cache-dir` 或
`ENTIRE_PLUGIN_DATA_DIR` 时才使用缓存，这一点与其他查询命令不同。缓存位置、键输入
与预热方法见 [operations](docs/operations.md#cache)。

## 局限

静态分析是启发式的。经由接口、反射、动态分发以及生成代码或运行时装配代码的调用，
可能被漏掉或无法解析。上面抓取的会话就展示了这样一个例子。**依赖计数是用于检查的
参考，不是编译期事实。** 解析器无法处理的文件会以机器可读的部分失败（partial
failure）形式暴露，而不是静默缺失。

[语言覆盖](docs/language-support.md)分为两层：36 种具备语义解析的语言，以及仅有
清单（inventory-only）的文件类型——后者只有文件与符号结构，没有调用或类型分析。
用 `entire graph capabilities --json` 查看当前构建的实际情况。

Entire Graph 是面向「你的智能体正在工作的那个仓库」的代码智能：带排序的检索、关系
与变更影响，全部以源码为根基。它**不**存储用户或对话记忆，不在后台常驻运行，也不
暴露自己的 MCP server。完整的数据流与写入面描述（包括哪些东西会执行调用方提供的
命令）见 [trust and security](docs/trust-and-security.md) 文档。

## 文档

- [全部文档](docs/README.md)
- [命令参考](docs/commands.md)
- [智能体激活与验证](docs/agents.md)
- [检索结果与排序](docs/search.md)
- [运维：安装、缓存、故障排查](docs/operations.md)
- [信任与安全](docs/trust-and-security.md)
- [语言支持](docs/language-support.md)
- [基准测试方法学与证据](docs/benchmarks.md)

欢迎在 [GitHub Issues](https://github.com/entireio/entire-graph/issues) 报告问题，
或提交 pull request。谢谢！❤️

## 许可证

Entire Graph 基于 [MIT License](LICENSE) 发布。
