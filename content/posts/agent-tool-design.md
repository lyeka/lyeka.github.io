+++
date = '2026-09-06T00:00:00+08:00'
title = 'Agent Runtime 如何执行 Tool Call 并回填结果'
tags = ['AI', 'Agent', 'Agent Runtime', 'Tool Call', 'Tool Use', 'MCP', 'Claude Code']
+++

# Agent Runtime 如何执行 Tool Call 并回填结果

用户让 Agent 查看仓库状态时，模型不会自己启动 Shell。模型先生成一条结构化的 Tool Call，表达“调用哪个工具、传入什么参数”。Agent Runtime 随后检查这条请求，决定是否允许执行，并在命令结束后把结果写回对话。这里的 **Agent Runtime** 指运行模型循环、调用工具并维护消息的宿主程序。

Tool Call 能否顺利执行，取决于三类对象：

- **Tool Definition** 是给模型看的接口说明，告诉模型工具能做什么、何时使用以及参数怎样填写。
- **Tool Call** 是模型生成的调用请求，包含调用 ID、工具名和参数。
- **Tool Outcome** 是 Runtime 处理完调用后形成的完整结果，包括成功内容、错误状态、结构化数据和宿主需要保存的元数据。Tool Outcome 是这里为了说明机制使用的统称，不对应某个协议的固定字段。

这三类对象由不同组件处理。模型根据 Definition 生成 Call；Runtime 校验、授权、调度并执行 Call；工具结束后，Runtime 再把 Outcome 中适合模型阅读的部分放进下一轮请求。Tool 设计如果只定义函数签名，没有处理权限、执行顺序和结果回填，模型仍然无法可靠地使用它。

下面的实现细节主要来自 PI 和 Claude Code 的公开源码快照。Claude Code 快照只包含可以核对的客户端实现，不包含 Anthropic 未公开的服务端逻辑。版本、代码位置和公开链接统一列在文末资料索引。

## 1. 一次 Tool Call 从用户请求走到下一轮模型

以 Anthropic Messages API 的客户端 Tool 为例，Runtime 第一次请求模型时，会同时发送用户消息和 Bash Tool 的 Definition。下面省略 `model`、`max_tokens` 等无关字段：

~~~json
{
  "tools": [
    {
      "name": "bash",
      "description": "Run a shell command and return its output.",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": { "type": "string" }
        },
        "required": ["command"]
      }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "查看当前仓库状态"
    }
  ]
}
~~~

`input_schema` 是参数的机器可读规则。它规定 `command` 必须是字符串，而且调用时不能缺少这个字段。模型读取 Definition 后，可能返回下面这条 Call：

~~~json
{
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01",
      "name": "bash",
      "input": {
        "command": "git status --short"
      }
    }
  ]
}
~~~

`tool_use` 只表示模型希望调用 Bash。Runtime 仍要查找工具、检查参数和权限，然后才会启动命令。命令结束后，Runtime 保留这条 Assistant Message，并紧接着加入一条带相同调用 ID 的结果消息：

~~~json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01",
      "content": " M src/app.ts"
    }
  ]
}
~~~

模型服务商的 API，也就是下文所称的 **Provider**，会规定角色名和字段形状。不同 Provider 的格式可以不同，但客户端 Tool 都需要完成同一条处理链：

`Tool Definition → 模型生成 Tool Call → Runtime 校验、授权与调度 → Tool 执行 → Runtime 形成 Tool Outcome → 下一轮模型请求`

模型在下一轮读到 `tool_result` 后，才能判断接下来应该读取 `src/app.ts`、修改代码，还是直接回答用户。调用 ID 把 Result 和原 Call 连接起来。即使多条 Call 同时执行，Runtime 也不能依靠数组位置猜测某个结果属于哪次调用。

~~~mermaid
sequenceDiagram
    participant R as Agent Runtime
    participant M as Model
    participant B as Bash Process

    R->>M: 发送用户消息和 Bash Definition
    M-->>R: 返回 tool_use(id, name, input)
    Note over R: 查找工具、校验参数、判断权限、安排顺序
    R->>B: 启动 git status --short
    B-->>R: 返回 stdout、stderr 和退出状态
    Note over R: 整理成 Tool Outcome
    R->>M: 发送历史消息和 tool_result(id, content)
    M-->>R: 继续调用工具或回答用户
~~~

图中只画了由宿主执行的客户端 Tool。Web Search 等服务端 Tool 可能由模型服务商执行，但模型提出调用、某个运行时完成执行、结果再返回模型，这三段责任仍然存在。

## 2. 模型和 Runtime 需要不同的 Tool 信息

### Definition 只放模型需要使用的接口

Tool Definition 的直接读者是模型。一个自定义客户端 Tool 通常包含三个基础字段：

- `name` 提供稳定且唯一的调用名。
- `description` 说明工具能做什么、何时使用以及能力边界。
- `input_schema` 规定参数名称、类型和必填关系。

Provider 还可能支持 Output Schema、Strict Mode、延迟加载和协议 Annotation。Annotation 是 Tool 随接口提供的行为提示。Output Schema 约束输出形状；Strict Mode 要求模型更严格地遵守 Schema；延迟加载允许 Runtime 在真正需要某个 Tool 时再发送完整 Definition。这些字段都在帮助模型选择工具、填写参数或理解结果。执行函数仍由 Runtime 保存和调用。

Runtime 还需要另一组信息。它要知道怎样启动和取消工具、多久算超时、怎样报告进度；权限模块要判断参数会访问什么资源、是否改变状态以及能否并发；结果模块要决定模型、界面、SDK 和日志各自读取什么。SDK 是供其他程序调用 Agent 的开发接口。把这些宿主字段全部发给模型会增加 Context 占用，也不会自动形成安全限制。**Context** 指模型在当前请求中能够读取的全部输入。

PI 按使用者拆分这些字段。底层 `Tool` 主要保存名称、描述和参数 Schema；`AgentTool` 增加参数准备、执行函数和执行模式；Coding Agent 的扩展定义再增加提示片段和 UI Renderer。**UI Renderer** 是把工具结果转换成界面内容的渲染器。Claude Code 的产品型 Tool 接口更集中，同一个 Tool 类型会暴露 `call`、`isReadOnly(input)`、`isConcurrencySafe(input)`、`checkPermissions`、结果映射和 Renderer。两种组织方式都让模型 Definition 保持精简，同时把真正需要执行的规则留在宿主侧。

### 模型描述保持稳定，调用说明随参数变化

同一个 Bash Tool 需要两类说明。模型需要稳定的接口描述，例如“执行 Shell 命令并返回结果”。用户在权限确认框中需要看到这次调用具体会做什么：`git status --short` 可以显示为“读取当前仓库状态”，`rm -rf build` 则应显示为“删除 build 目录”。后一类说明必须读取本次 Input。

PI 的底层 `Tool.description` 保存静态的模型接口描述。Claude Code 把两类文字分开：`prompt(options)` 生成模型看到的描述，`description(input, options)` 根据参数和权限状态生成用户看到的调用说明。Claude Code 第一次调用 `prompt()` 后，会把名称、描述和 Input Schema 等基础字段放入 Session 级 Schema Cache，后续请求可以直接复用。Session 级缓存的生命周期与当前会话相连。

Runtime 可以在会话建立或可用 Tool 列表变化时，根据 Shell 类型、租户能力和已安装 Tool 生成一版模型接口。此后，这版 Definition 应保持稳定。调用说明则在每次执行前重新生成，因为参数和权限状态都会变化。实时业务数据更适合通过普通消息或专门的查询 Tool 提供，不适合反复写进接口描述。

稳定的 Definition 还能提高 Prompt Cache 命中率。**Prompt Cache** 会复用请求开头没有变化的字节，减少重复输入的处理成本。Anthropic 请求先序列化 Tools，再放入 System Prompt 和 Messages。同一个 Tool 如果每轮生成不同描述，变化会破坏后面的缓存前缀；同一段 Transcript 在重放时也会面对不同的接口语义。**Transcript** 是按顺序保存的完整会话记录，包括模型消息、Tool Call 和 Tool Result。

Tool Definition 列表也要保持确定顺序。相同的工具集合如果每轮随机换序，序列化后的请求前缀仍会变化。固定顺序既方便复现一次运行，也能让相同前缀继续使用 Prompt Cache。单个 Definition 保持稳定，并不要求 Runtime 从第一轮开始加载全部 Tool。

### Tool 很多时先搜索，再加载完整 Schema

Tool 数量很少时，Runtime 可以把全部 Definition 一次发给模型。Tool 数量增加后，每个 Description 和 Input Schema 都会持续占用 Context，大量无关接口也会干扰模型选择。支持延迟加载的 Runtime 可以先发送常用 Tool 的完整 Definition，再提供一个 `ToolSearch`。其余 Tool 先作为候选名称存在，完整 Schema 等搜索命中后再进入模型请求。

Claude Code 快照中的 `ToolSearchTool` 使用本地词法搜索。模型把当前意图写成搜索词，Runtime 负责执行确定的匹配和排序。以 `ToolSearch(query: "database query")` 搜索 `query_database` 为例，处理过程如下：

1. Runtime 把查询转成小写，再按空格拆成 `database` 和 `query`。Tool 名称按 CamelCase 和下划线拆分，因此 `QueryDatabase` 与 `query_database` 都会得到 `query`、`database` 两段。
2. Runtime 只给尚未加载的候选 Tool 计分。查询词与名称分段完全相等加 10 分；被某个名称分段包含加 5 分；命中 `searchHint` 加 4 分；名称的规范化完整形式命中、且候选此前尚未得分时加 3 分；Description 按单词边界命中再加 2 分。
3. `query_database` 的两个名称分段分别精确命中，名称得分为 20。如果它的 `searchHint` 是 `database query`，Description 是 `Run a database query and return rows`，两个查询词还会各自得到 4 分和 2 分，总分为 32。
4. Runtime 删除零分候选，按分数倒序排列，默认返回前 5 个。查询中的 `+database` 表示必选词；候选的名称、Description 或 `searchHint` 必须命中所有带 `+` 的词，才会进入计分阶段。

`select:query_database` 会跳过排序，直接按完整名称选择 Tool。关键词查询如果与完整 Tool 名称完全相等，也会优先命中该 Tool。

搜索结果返回 `tool_reference`。它只是一个 Tool 引用，不包含执行器。Runtime 从消息历史中找出这些引用，后续请求再加入命中 Tool 的完整 Schema。模型拿到 Schema 后，才能生成 `query_database(sql, database)` 这样的 Call。

这套搜索没有使用 Embedding、向量数据库或另一个 LLM。Embedding 会把文本转换成可计算的数值向量，LLM 指大语言模型。Tool Search 也不提供同义词扩展、翻译和拼写纠错。模型可以根据中文请求主动提交英文查询 `database query`。如果模型直接提交 `查询数据库`，而候选 Tool 的名称、Description 和 `searchHint` 都只有英文，Runtime 不会自动匹配到 `query_database`。Tool 的命名和搜索元数据会直接影响召回结果。

PI 没有内置通用 Tool Search。它允许 Extension Tool 在运行中激活已经注册的 Tool。PI 的回归测试里，普通扩展 Tool `load_more_tools` 调用 `pi.setActiveTools()`，把 `after_load` 加入 Active Tools；PI 记录这个变化，并在后续模型请求中提供 `after_load` 的 Schema。

具体怎样寻找和选择 Tool 由 Extension 自己实现。测试只写死了 `after_load`，实际项目可以在扩展里加入关键词、规则或语义检索。Loader Tool 因而是一种扩展用法，不是 PI 的内置 Tool 类型。Claude Code 则把通用的加权关键词搜索放进 Runtime。两种做法都能推迟完整 Schema 的加载。Provider 如果不支持这种延迟加载协议，Runtime 需要退回完整预加载；Tool 很少时，完整预加载本身也更简单。

Definition 帮助模型生成结构化 Call。Call 到达宿主后，Runtime 还要根据参数和当前状态决定它能否执行。

## 3. Runtime 把调用请求变成受控执行

### Runtime 在启动 Tool 前逐层检查 Call

模型生成的 Call 是一份外部输入。模型准备修改 `config.yaml` 时，可能先生成：

`Read(path: "config.yaml")`

Runtime 不会直接读取文件。它先在本轮可用 Tool 中查找 `Read`，再确认 `path` 是字符串且必填参数齐全。参数形状通过后，Runtime 检查路径和文件状态，并根据用户权限、沙箱范围与本地 Policy 决定放行、询问还是拒绝。**Policy** 是宿主用来判断哪些调用可以执行的一组规则。如果另一个调用正在修改同一文件，Read 还可能需要等待。

一次 Call 进入执行的顺序是：

`查找 Tool → 校验参数形状 → 检查当前状态 → 判断权限 → 安排执行顺序 → 启动 Tool`

每一步处理的问题不同。Tool 不存在时，Runtime 无法解释调用。模型把 `path` 生成为数组时，参数不能安全地交给执行器。文件已经删除或自上次读取后发生变化时，参数格式虽然正确，当前状态却不再满足操作条件。调用有效但用户没有授权时，Runtime 必须在副作用发生前停止。两次写入命中同一文件时，调用可能都合法，但执行顺序仍要受到控制。

PI 和 Claude Code 都在真实执行前完成参数校验与宿主拦截。工具函数没有最终决定权。Runtime 可以因为输入、状态、权限或调度条件不满足而不调用它。

### Runtime 根据本次参数判断调用特性

生产级 Tool 还要告诉 Runtime 本次 Call 会怎样影响外部状态。

| Runtime 读取的信息 | Runtime 用它决定什么 |
|---|---|
| 本次调用是否只读 | 是否改变可观察状态，能否使用较低的权限与并发门槛 |
| 修改是否难以恢复 | 是否需要更强的确认或恢复措施 |
| 是否访问外部系统 | 是否越过本地边界，是否可能向外发送数据 |
| 与其他调用并发是否安全 | 同时运行时结果能否保持正确 |
| 调用能否中断 | 用户发来新指令时能否立即停止 |
| 调用操作的资源 | 具体涉及哪个文件、进程、事务或远端对象 |

这些判断通常依赖本次 Input，不能只按 Tool 名称写死。`git status --short` 只读取本地仓库；Shell 重定向到 `config.yaml` 会修改文件；删除目录还会造成难以恢复的变化；`curl` 即使只读取远端数据，也会越过本地边界，并可能把 URL、Header 或请求体发送给外部系统。同一个 Bash Tool 会随着 `command` 改变风险。

只读、容易恢复和可以并发是三个独立判断。两个只读查询可能争用同一事务或触发远端限流。两个写操作如果修改不同文件，反而可能安全地同时执行。一次网络读取没有修改本地文件，却仍可能泄露数据。Tool 计算出的特性为 Runtime 提供判断依据，Runtime 仍要结合 Input、当前状态和 Policy 作出准入、确认与调度决定。无法确认安全性时，Runtime 应采用更保守的处理。

Claude Code 根据本次 Input 计算只读性、外部访问和并发安全，再把结果交给权限与调度流程。PI 提供执行前的拦截点，由宿主应用配置权限规则和隔离环境。两种实现都把工具能力声明和单次调用的执行许可分开处理。

### Read、Edit 与 Write 共享文件状态

单次调用通过检查后，多次调用的组合仍可能出错。模型先读取 `config.yaml`，再根据文件内容生成 Edit。如果用户、格式化器或另一个进程已经修改了文件，Edit 的参数仍可能符合 Schema，但它依据的是旧内容。

Claude Code 的 Read 会记录模型实际读到的内容和文件版本。Edit 或覆盖已有文件的 Write 执行前，Runtime 确认模型已经完整读取过目标文件。文件在读取后发生变化时，写入会被拒绝，模型必须重新读取并重新生成修改。Runtime 在真正写盘前还会再次核对文件状态，缩小“检查时未变化、落盘时已经变化”的时间窗口。

这套检查把 `Read → Edit/Write` 变成 Runtime 维护的状态关系。它阻止的是参数合法、依据却已经过期的修改。

PI 的 Edit 在执行时读取当前文件，并检查模型指定的旧文本是否仍然存在。旧文本不存在时，Edit 会拒绝修改，不会猜测新的位置。PI 还让修改同一文件的 Edit 和 Write 进入同一条队列。前一个写操作结束后，后一个才能开始。修改不同文件的调用可以并行；两个不同路径最终指向同一文件时，Runtime 仍按同一资源处理。

Claude Code 的读后写检查保证模型基于自己看过的版本修改。PI 的旧文本检查和同文件队列保证替换目标仍然成立，并阻止多个写操作交错覆盖。Runtime 既要识别单次调用的特性，也要维护 Tool 之间共享的资源状态。

### 并发调度先处理依赖和资源冲突

一次模型响应可以包含多个 Call。Runtime 面对的关系主要有三种。

用户要求比较两份配置时，模型可以同时生成 `Read(config.yaml)` 和 `Read(schema.yaml)`。两次读取互不修改文件，也不需要对方的结果来生成参数。Runtime 可以同时启动它们，完成顺序不会改变结果。

`Edit(config.yaml)` 与 `Write(config.yaml)` 操作同一资源，必须排队，否则后完成的写入可能覆盖前一个结果。`Edit(config.yaml)` 与 `Write(schema.yaml)` 操作不同文件。能够按文件隔离资源的 Runtime 可以并行执行；把所有写操作都视为独占的 Runtime 会选择串行。后者放弃一部分吞吐量，换来更简单的安全边界。

修改配置后运行测试存在结果依赖。测试只有在新内容落盘后才有意义。模型可以先调用 `Edit(config.yaml)`，收到成功 Outcome 后，再在下一轮生成测试命令。如果两个 Call 已经出现在同一响应中，Runtime 只有采用保留 Call 顺序的串行策略，才能保证测试晚于修改执行。通用并发执行器不能只凭 Tool 名称推断这条业务依赖。

Runtime 可以按以下顺序安排调用：

`先看结果依赖 → 再看资源冲突 → 最后检查 Tool 的并发声明和外部限制`

需要前一个结果才能生成参数的调用应当分轮。已经带有完整参数的兄弟调用，再检查它们是否争用文件、进程或事务。没有依赖和资源冲突时，Tool 的并发声明、外部限流和连接数才决定它们能否同时启动。

Claude Code 收到完整 Call 后，先按 Schema 解析 Input，再让对应 Tool 根据参数判断并发安全。Read 明确允许并发。Edit 和 Write 没有声明并发安全，按默认值视为不安全。Bash 会检查整条 `command`，只有能够识别为只读操作时才返回安全。`git status --short` 可以通过；写重定向、无法解析的复杂命令和 `npm test` 都会落到不安全一侧。Schema 校验失败或判断过程报错时，Runtime 同样按不安全处理。

并发安全只回答调用能否与其他 Call 同时执行。Read 仍可能因路径权限被拒绝。Edit 被判为不可并发，也不表示每次都需要用户确认；它只需要在执行期间独占运行。

Claude Code 在等待完整响应后再执行的路径中，会按 Call 顺序切分批次。例如模型依次生成：

`Read(config.yaml) → Read(schema.yaml) → Edit(config.yaml) → Read(other.yaml)`

前两个 Read 连续且都安全，会一起启动。Edit 单独形成一个批次，等两个 Read 结束后执行。最后一个 Read 不能越过前面的 Edit，只能等 Edit 完成。两个修改不同文件的 Edit 也会被归入独占调用，因此不会获得额外并发。这套规则无须计算两个 Input 是否指向同一文件，代价是放弃一部分本来安全的并行。

PI 先取得完整的 Assistant Message，再从中提取全部 Call。它默认让批次并行，也允许宿主把全局模式改成串行。只要批次中有一个 Tool 声明必须串行，整批 Call 就按原顺序执行。全局允许并行且没有 Tool 要求串行时，PI 先逐个完成参数准备和执行前拦截，再同时启动通过检查的调用。

PI 的文件 Tool 在批次调度下面还有一层资源队列。默认并行批次如果同时包含 `Edit(config.yaml)`、`Write(config.yaml)` 和 `Edit(schema.yaml)`，前两个调用争用同一文件，只能依次写入；第三个调用使用另一条队列，可以同时修改 `schema.yaml`。Claude Code 主要依据本次 Input 的安全属性决定独占或并发。PI 先用全局设置和 Tool 声明决定整批模式，再由具体 Tool 处理同一资源的冲突。

### 流式响应允许完整的 Call 提前执行

流式响应会分片返回 Tool 名称和参数 JSON。Runtime 不会执行一段尚未生成完整的参数。它必须等一条 Tool Call 完整结束并通过解析，才可能启动对应 Tool。此时整条 Assistant Response 仍可继续生成，后面也可能出现新的 Call。

Claude Code 可以按下面的到达顺序执行：

1. `Read(config.yaml)` 完整到达。Runtime 将它判为并发安全并立即启动。
2. `Read(schema.yaml)` 随后完整到达。当前运行的也是安全调用，第二个 Read 可以同时启动。
3. `Edit(config.yaml)` 最后到达。它不能与当前调用重叠，必须等待两个 Read 结束。

如果 Edit 先到达并开始执行，后到的 Read 也要等待。非安全调用在执行期间独占 Runtime。调度器无须预知后面还会生成什么；每条新 Call 到达时，它只需比较这次调用的属性和当前执行状态。

这套规则要求“并发安全”的标记足够严格。两个被标记为安全的调用如果实际争用同一事务或共享同一个外部限额，调度器在冲突发生后无法补救。更细的 Runtime 可以在调用到达时为文件、进程或事务申请资源锁。后续 Call 命中已占用资源就等待，操作其他资源仍可运行。

资源锁只能识别共享对象，无法推断“测试必须看见刚才的配置修改”这类业务关系。模型应把这种依赖分成两轮，或者由 Runtime 等完整响应结束后统一调度。

Claude Code 在单条 Call 完整后便可提前执行，并用“安全调用相互并发、非安全调用独占”处理尚未到达的兄弟调用。PI 的核心循环先取得完整 Assistant Message，因此没有同样的未知后续 Call 窗口。PI 仍会在批次执行阶段使用串行模式和同文件队列控制冲突。

流式提前执行也扩大了故障边界。响应随后中断时，已经完成的外部动作不会随消息丢弃而撤销。Runtime 不能自动重放同一个 Call，除非操作本身幂等，也就是重复执行不会新增副作用；执行端也可以根据稳定的幂等键识别并丢弃重复请求。

PI 会在并发批次结束后按原 Call 顺序生成 Result。Claude Code 的流式执行器允许安全调用的 Result 按完成情况返回，非安全调用继续充当顺序屏障。无论结果采用哪种顺序，Runtime 都依靠 Call ID 维持 Call 与 Result 的配对。

### 调用启动后仍要处理取消和失败

Tool 启动后，Runtime 还要决定什么时候可以停止。用户发来新指令时，读取或搜索通常可以安全取消。文件写入已经进入关键阶段时，强行停止可能只写入一半。Runtime 更适合等原子操作结束，再处理新指令。可中断性描述的是这条执行边界，不只取决于 Tool 是否接收 `AbortSignal`，也就是代码里的取消信号。

并发批次中的失败也可能影响兄弟调用。几条 Bash 命令可能形成隐含的操作链，前一条失败后继续执行其他命令会扩大错误。几次独立文件读取没有这种依赖，一次读取失败不必取消其他调用。Claude Code 因而会在 Bash 调用报错时取消同批仍在运行的兄弟调用，普通读取失败不会触发相同处理。

PI 的文件修改队列还处理取消后的竞态。写操作收到取消信号后，队列不会立刻放行同一文件的下一次修改。它会等底层文件操作真正结束。否则，已经宣布取消的旧写入可能晚于新写入落盘，并覆盖新结果。

Runtime 最终会拒绝、等待、取消或完成一条 Call。工具完成后产生的对象，还要转换成模型、UI、SDK 和日志各自能使用的结果。

## 4. Runtime 把 Tool Outcome 交给不同消费者

### 成功结果在本地消息中保留两种表示

模型生成 `Bash(command: "npm test")` 后，Bash Tool 可以得到 stdout（标准输出）、stderr（标准错误）、中断状态和后台任务信息。成功输出过大时，内部结果还可能记录完整输出文件的位置和原始大小。Anthropic API 接受的 `tool_result` 只包含 Call ID、模型需要阅读的内容和可选错误标记。内部返回对象和 Provider Message 因而承担不同任务。

下面是根据 Claude Code 公开快照的字段关系精简后的示意对象。它表示命令执行成功，但输出达到 820KB。对象里的 Preview 是截取给模型阅读的输出片段：

```json
{
  "message": {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_01",
        "content": "<persisted-output>\nOutput too large (820KB). Full output saved to: /tmp/tool-results/toolu_01.txt\n\nPreview (first 2KB):\nPASS src/config.test.ts\n...\n</persisted-output>"
      }
    ]
  },
  "toolUseResult": {
    "stdout": "PASS src/config.test.ts\n...",
    "stderr": "",
    "interrupted": false,
    "persistedOutputPath": "/tmp/tool-results/toolu_01.txt",
    "persistedOutputSize": 839680
  }
}
```

外层对象属于 Claude Code，本地 Runtime 会完整保存它。嵌套的 `message` 才准备交给 Provider。下一轮请求构造 API 参数时，Claude Code 只读取 `message.role` 和 `message.content`。外层的 `toolUseResult`、本地 UUID 和 `mcpMeta` 不会进入模型 Context。消息转换代码负责这个隔离，Tool 作者无须在每次调用后手工删除字段。

模型必须收到带原 Call ID 的 `tool_result`，才能在下一轮使用执行结果。外层 `toolUseResult` 即使保存了结构化状态，也不能代替模型历史中的结果消息。

UI 会读取完整的本地消息。成功结果到达界面后，Claude Code 取出 `toolUseResult`，先按当前 Tool 的 Output Schema 校验，再交给成功 Renderer。Bash Renderer 可以直接使用 `stdout`、`stderr` 和执行期间保存的 Progress Message，不必解析写给模型的 `<persisted-output>` 文本。界面也不会原样显示这段专供模型读取的包装。

Claude Code 的 SDK 可以把外层对象显式序列化为 `tool_use_result`，让调用程序取得完整 Tool 输出。**Telemetry** 指 Runtime 为运行分析记录的测量数据。它不会整段复制模型消息或 UI 对象，而会选择 Tool 名称、耗时、结果大小、权限决策和经过控制的参数。失败事件改为记录错误分类或错误内容、耗时与决策信息。Telemetry 还要执行自己的脱敏和访问控制，适合界面显示的内容不一定适合写入日志。

失败路径与成功路径略有不同。UI 先检查 `tool_result.is_error`。它为 `true` 时，错误 Renderer 通常直接接收同一条 `tool_result.content`。这段错误文本既帮助模型恢复，也帮助用户理解问题，但模型和界面会采用不同的显示方式。

PI 使用另一种对象结构完成相同分工。`AgentToolResult.content` 保存模型可见的文字或图片，`details` 保存截断范围、完整输出路径等宿主信息，Renderer 直接读取 `details`。

### 每个失败都要形成与 Call 配对的 Result

`npm test` 以退出码 `1` 结束时，Claude Code 的 Bash Tool 会抛出包含退出码和命令输出的 `ShellError`。Executor 在单次 Tool 边界捕获异常，整理错误文本，并创建一条正常的 User Message。进入下一轮模型历史的主要内容类似下面这样：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01",
      "is_error": true,
      "content": "Exit code 1\nFAIL src/config.test.ts\nAssertionError: expected 3, received 2"
    }
  ]
}
```

`tool_use_id` 指明失败属于哪条 Call，`is_error` 让模型和 UI 能够直接识别错误，`content` 保存恢复所需的信息。多条 Tool 并发完成时，Result 仍通过 ID 找到原 Call。已经生成的 Call 如果没有配对 Result，模型无法判断它仍在执行、被 Runtime 遗漏，还是已经失败，Transcript 也会停在不完整状态。

Claude Code 会在四个阶段生成 Error Result：

1. Tool 名称不存在。Executor 找不到实现，不会启动任何外部动作。
2. Input 没有通过 Schema。错误内容会指出具体字段问题，执行器仍未启动。
3. 权限链拒绝调用。Runtime 在副作用发生前停止，并把拒绝原因写进结果。
4. Tool 启动后抛出异常。Executor 格式化执行错误；Bash 非零退出属于这条路径，文本按退出码、stderr 和 stdout 组织。

错误进入本地消息后，Claude Code 还会在外层保存形如 `Error: ...` 的 `toolUseResult`，供 Transcript 或 SDK 使用。错误 UI 依据 `is_error` 把 `tool_result.content` 交给 Tool 的错误 Renderer。成功 UI 使用内部结果对象，失败 UI 使用模型可见的错误内容，两条路径读取的数据并不完全相同。

一条有用的错误信息要说明失败发生在哪个阶段：调用尚未执行、已经执行但返回失败，还是响应丢失后执行结果未知。它还要说明外部动作是否发生，并给出具体的 Tool、字段、路径或退出码。现场输出应控制大小，结尾再告诉模型可以修改参数、换 Tool、检查状态还是只能上报。`is_error` 只标记状态，不能代替这些行动信息。Claude Code 为部分查找和校验错误添加的 `<tool_use_error>` 只是内容标记，并非所有 Tool 协议都要采用的统一格式。

Tool 不存在、参数错误、权限拒绝和普通执行异常都可以作为 Error Outcome 返回模型。Transcript 损坏、Provider 请求无法继续、宿主进程级故障或用户明确终止整个 Run 时，错误才会越过单次 Tool 边界。写操作超时或远端连接中断会留下未知状态：Runtime 没有收到成功响应，但外部副作用可能已经发生。执行端没有稳定幂等键或状态检查能力时，Runtime 不能直接重试。

### 大结果按内容类型选择 Preview

Runtime 应先在数据产生处缩小结果。测试 Tool 可以只运行失败用例；数据库查询可以选择字段并设置 Limit；读取和搜索可以使用范围、分页或游标。结果仍然超限时，Preview 要适应内容类型。日志通常需要末尾错误，文件通常从开头建立背景，搜索要保留命中附近，JSON 更适合分页或整体外置。统一执行 `slice(0, N)` 会丢掉不同结果中真正重要的部分。

Claude Code 的成功 Bash 路径会在原始输出已经落盘时，把文件复制到 Tool Result 存储位置，并在内部对象中保留 `persistedOutputPath` 和 `persistedOutputSize`。模型映射读取这些字段，生成文件位置、原始大小和 Preview。成功 UI 继续读取 `toolUseResult.stdout`，不会显示专供模型使用的持久化包装。其他 Tool 的映射结果超过单结果阈值时，通用处理器也可以把全文保存到文件，再向模型提供 Preview 和读取方法。公开快照还提供可选的消息级总量预算，防止多个兄弟 Result 单独不大、合在一起却占满 Context。

Bash 执行失败走另一条路径。非零退出触发 `ShellError` 后，Executor 使用 `formatError` 生成错误文本。公开快照对超过 10000 字符的内容保留开头 5000 字符和结尾 5000 字符，并在中间写明删去了多少字符。这条错误路径没有先为模型生成可继续读取的完整输出引用。因此，Claude Code 的成功结果和执行异常使用不同的大结果处理方式。

PI 的 Bash 输出捕获会保留日志尾部。内容被截断时，PI 把完整输出写入临时文件，并在模型内容中写明显示了多少行以及全文路径；`details` 还保存截断信息供宿主使用。

外置结果要让模型知道 Preview 是否完整、保留了哪一段以及原文有多大。Runtime 还要提供当前模型真正能够调用的续读方式。摘要适合帮助模型定位内容，但不能成为大结果的唯一记录。

本地 Runtime 到这里已经完成结果转换。执行器如果位于另一个进程或服务中，远端返回还要先经过协议转换，才能进入相同的本地消息处理流程。

## 5. MCP 把远端 Result 送到 Host

**Model Context Protocol（MCP）** 是 Host 与外部 Tool Server 交换工具清单、调用参数和执行结果的协议。Host 运行模型循环、权限系统和 UI；Host 内的 MCP Client 负责连接；MCP Server 保存远端 Tool Definition，并对数据库、文件系统或外部 API 执行真实操作。

以远端 `query_database` 为例，MCP 负责把 Definition、Call 和 Result 送过进程或语言边界。Result 到达 Client 后，Host 仍要把它转换成本地 Tool 返回，再决定模型、UI 和 SDK 分别读取什么。

### 远端成功结果先变成本地 Tool 返回

连接建立后，Client 通过 `tools/list` 取得 `query_database` 的名称、描述、Input Schema 和 Annotation。这里的 Annotation 会提示 Tool 是否只读、是否具有破坏性或是否可能访问外部系统。Claude Code 为远端 Tool 名称加入 Server 标识，再把它包装成本地 Tool，并将 Definition 提供给模型。

模型生成 `query_database(sql, database)` 后，这条 Call 先经过本地权限和调度。Host 放行后，Client 才会发送 `tools/call`。

数据库 Server 可以返回三类信息：

- `content` 保存查询文本等普通内容。
- `structuredContent` 保存结构化的行和列。
- `_meta` 保存请求标识等只供 Client 使用的协议元数据。

Claude Code 先按 MCP Result Schema 校验远端返回。结果含有 `structuredContent` 时，公开快照会把它序列化为本地 Tool 可读的 `data`；结果只有普通 `content` 时，Client 会把文本、图片或资源内容转换成本地可处理的形式。原始 `structuredContent` 和 `_meta` 另存到 `mcpMeta`。

本地 Tool 随后返回 `{ data, mcpMeta }`。Runtime 把 `data` 映射到嵌套的 `message.content`，形成模型读取的 `tool_result`。同一份 `data` 也保存在外层 `toolUseResult`，供 MCP Renderer 和 SDK 使用。`mcpMeta` 留在外层，不会进入 Provider Message。SDK 可以显式组装 `content`、`structuredContent` 和 `_meta`，模型只读取 Host 已经整理并限制大小的内容。

MCP 负责把远端 Result 交给 Host。Host 的本地消息结构和序列化规则决定哪些字段进入模型。模型与 UI 在 MCP Tool 上可能从相同的 `data` 开始，只是 UI 可以使用专门 Renderer，SDK 还能读取 `mcpMeta`。

大结果也由 Host 处理。MCP 返回在进入本地 Tool 以前会经过大小检查。超限的非图片内容可以保存到文件，再把 `data` 替换为文件位置、原始大小和读取说明。模型和 MCP Renderer 此后拿到的都可能是这份说明，SDK 仍可能通过 `mcpMeta` 保留结构化内容。Host 的转换允许丢弃或缩短部分内容，MCP 不保证原始响应会完整复制给每个消费者。

### MCP 的两类失败都转成 Error Outcome

MCP 使用两条路径表示失败。协议消息格式错误、未知方法等问题使用 JSON-RPC Error。Tool 已经被正确调用，但参数值、下游 API 或业务规则失败时，Server 可以返回正常的 Call Tool Result，并设置 `isError: true`。两类错误到达 Claude Code 后都会经过本地 Executor，但 Runtime 仍要保留它们各自的来源，方便排查和判断能否重试。

Server 返回 `isError: true` 时，Claude Code 从 Result 中提取第一段可读文本，创建一个 MCP Tool 异常，并把远端 `_meta` 挂到异常上。通用 Executor 捕获异常后，生成带原 Call ID、`is_error: true` 和错误文本的本地 `tool_result`。外层 `toolUseResult` 保存 `Error: ...`，`mcpMeta` 继续供 SDK 使用。模型和错误 UI 读取本地错误内容，不直接处理 MCP 原始 Result。

JSON-RPC 通信异常、超时和意外返回格式也会被 Executor 转换为 Error Outcome，因此模型最后看到的结构可能相同。Telemetry 和调试信息仍要区分 Server 明确返回业务失败、连接中断和 Host 无法解析响应。否则，Runtime 会丢失排障和重试所需的信息。

Claude Code 在 MCP SDK 的超时之外还设置了本地调用期限。它只在明确检测到 Session 失效时重新连接并重试一次，普通远端错误不会被统一重放。

### Host 与 Server 各自执行安全检查

MCP 标准化了 Tool Definition 的发现、Call 的传输和 Result 的协议表示。Host 仍要决定向模型暴露哪些远端 Tool、是否请求用户确认、怎样调度、多少结果进入 Context、怎样生成本地错误，以及 UI 和日志显示什么。Server 则要再次校验参数，并在真实数据源前实施访问控制、限流和输出清理。Host 允许一条 Call，只表示本地 Policy 同意发出请求，不能代替 Server 的数据库权限。

Annotation 也只提供判断线索。Claude Code 会把 `readOnlyHint`、`destructiveHint` 和 `openWorldHint` 映射到本地的只读、破坏性和外部访问属性，`readOnlyHint` 还会影响并发分类。MCP 规范要求 Host 不要直接信任来自不可信 Server 的 Annotation。远端声明不会自动产生 Allow 决策，Call 仍要经过本地权限链。

沙箱边界由部署方式决定。通过标准输入输出通信的本地 stdio Server 可以由 Host 隔离。远端 Server 使用的文件系统、网络和凭据只能由服务端控制，Host 的本地沙箱无法约束它们。

Tool 需要跨进程、跨语言或供多个 Host 复用时，MCP 可以省去重复设计发现和调用协议。少量进程内函数直接使用本地接口更简单。采用 MCP 只改变 Host 与执行器之间的连接方式。Runtime 仍负责模型消息、权限、调度、大结果、错误恢复和运行记录。

沿一次调用回看，模型生成 Call，Runtime 决定它能否执行并把结果写回历史。Tool Search 控制哪些 Definition 进入请求，并发调度决定 Call 何时启动，本地消息决定 Outcome 的哪些字段交给模型、UI、SDK 和 Telemetry，MCP 则让 Definition、Call 和 Result 跨越进程与语言边界。

## 资料索引

### PI

- [PI 源码，commit `46bb9a2c`](https://github.com/earendil-works/pi/tree/46bb9a2c3bdb296b0d2179f7309ec6b79a7f3106)
- Tool 批次执行：`packages/agent/src/agent-loop.ts`，`executeToolCalls()`、`executeToolCallsSequential()`、`executeToolCallsParallel()`
- 同文件写入队列：`packages/agent/src/harness/tools/file-mutation-queue.ts`，`withFileMutationQueue()`
- 动态激活 Tool：`packages/coding-agent/src/core/agent-session.ts`，`setActiveToolsByName()`；回归测试见 `packages/coding-agent/test/suite/regressions/6162-extension-active-tools-next-turn.test.ts`
- Tool Result 分层：`packages/agent/src/types.ts` 中的 `AgentToolResult`，以及 Coding Agent Tool Renderer

### Claude Code 公开源码快照

- 快照 commit：`09f43552c76cb8856c4a5414f9aa9c9cda6ee035`
- 快照来自 2026 年 3 月公开暴露的 Source Map。Anthropic 没有把它作为完整源码正式发布，因此这里只引用可以在客户端快照中核对的行为
- Tool 接口与 Schema Cache：`src/Tool.ts`
- Tool Search：`src/tools/ToolSearchTool/ToolSearchTool.ts`，`searchToolsWithKeywords()`；`src/utils/toolSearch.ts`，`extractDiscoveredToolNames()`
- 流式并发：`src/services/tools/StreamingToolExecutor.ts`
- Result 与 Error Outcome：`src/services/tools/toolExecution.ts`，`addToolResult()`
- Provider Message 转换：`src/services/api/claude.ts`，`userMessageToMessageParam()`
- Bash 并发与大结果：`src/tools/BashTool/BashTool.tsx`，`isConcurrencySafe()`、`isReadOnly()` 及结果映射
- MCP 适配：`src/services/mcp/client.ts` 中的 Tool Adapter 和 Result 处理逻辑

### 协议与官方文档

- [Anthropic：How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)
- [Anthropic：Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
- [Anthropic：Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)
- [Anthropic：Tool use with prompt caching](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching)
- [Anthropic：Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
- [Model Context Protocol 2025-11-25：Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
