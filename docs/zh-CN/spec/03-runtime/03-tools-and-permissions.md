# 03. 工具和权限

> **翻译说明：** 本页是与 [英文源规格](/spec/03-runtime/03-tools-and-permissions) 一一对应的机器辅助翻译。代码、协议字段和标识符保持原文；如翻译与英文源事实有歧义，以英文版本为准。


> 应用的决定：D003、D004、D005、D006、D013、D015、D093、D114、D115、D181、D186、
> D189、D190、D195（ADR 0057）、D315、ADR 0087

## 0. 冻结政策总结

| 主题 | 决定 |
|---|---|
| 默认模式 | Agent |
| Agent 工具 | 读取/Glob/Grep/写入/编辑/Bash |
| Plan 工具 | 读取 / Glob / Grep / BrowserPreview / Bash / SubmitPlan |
| Goal 工具 | 读取/Glob/Grep/BrowserPreview/Bash/SubmitGoal |
| Plan 和 Goal 硬拒绝 | 编写/编辑/所有插件工具/未知工具/其他类型的提交工具 |
| 权限超时 | 120秒→拒绝 |
| 允许会话范围 | 工具名称 |
| 重击风格 | 非交互式；具有流输出的选定主机目录外壳 |
| Edit 契约 | 行锚定操作 + 整文件 `tag`；不再有 `old_string`/`new_string`（ADR 0087） |
| 询问工具 | 交互式多问题工具；无有效期期限；跳过的答案变成空输出字段 |

## 1. Goal

让代理完成工作，但默认情况下仍处于控制之下。

## 2. MVP内置工具

| 工具 | 风险 | 描述 |
|---|---|---|
| `Read` | 低 | 读取工作区中的文件；返回带行号的内容和 `[path#TAG]` 头 |
| `new_context` | 低 | 在下一个回合边界处启动一个新的上下文窗口；不接受任何参数并且不改变环境状态 |
| `Glob` | 低 | 按模式列出文件 |
| `Grep` | 低 | 内容搜索；本机有 `rg` 时优先用，否则进程内搜索；为每个文件生成 `tag` |
| `BrowserPreview` | 低 | 在随应用打包的浏览器插件中打开与工作区相关的预览（若 `pi.browser` 被禁用则失败） |
| `EnterPlanMode` | 低 | 主机验证后，将相同的 Agent 从 Agent 移动到 Plan |
| `SubmitPlan` | 低 | 在新的 `.pi/plan/*.md` 工件中保留精确的 Markdown 字节并请求批准 |
| `EnterGoalMode` | 低 | 主机验证后，将相同的 Agent 从 Agent 移动到 Goal |
| `SubmitGoal` | 低 | 在新的 `.pi/goal/*.md` 工件中保留精确的 Markdown 字节并请求批准 |
| `Write` | 高 | Create/overwrite 文件；返回写入后的 `tag` |
| `Edit` | 高 | 通过针对已校验 `tag` 的行锚定操作修改文件（[18](18-line-anchored-edit-contract.md)） |
| `Bash` | 高 | 执行命令 |
| `asktool` | 低 | 询问一个或多个用户问题并将提交的答案作为工具输出返回 |

> 名称可以在实现过程中进行微调，但语义保持一致。

### 2. 1 延期辅助工具（D185、ADR 0048）

按照 pi 的编码代理默认值，第一个 Agent 请求仅激活
`Read`、`Bash`、`Edit` 和 `Write`； `Glob` 和 `Grep` 按需加载。
Plan 和 Goal 保留其 read/inspection 核心。运行时还注册功能
无需预先发送其完整模式：

- Agent 模式下的 `Glob` 和 `Grep`
- `BrowserPreview`
- `PluginCheck`、`PluginScaffold` 和 `PluginPack`
- 插件声明的代理工具
- `Skill` 当启用的插件贡献技能时

这些工具出现在有界的 `# On-demand tools` 目录中，具有紧凑的结构
描述。该模型使用确切的名称调用本地 `ToolSearch` 工具或
能力查询；匹配的模式在下一个模型回合中可用。
sidecar 在每个新用户提示开始时重置此延迟集。
主机权限、workspace/scratch 遏制、超时和审核规则
加载工具时不会改变。 `ToolSearch` 本身从不执行工作区
操作并且永远不会绕过 host-core 策略。

## 3. 常用工具约束

每个非交互式执行工具都必须具有：

1. JSON 架构/类型框参数定义
2、超时
3.工作空间路径验证
4.输出截断策略
5. 跟踪ID
6. 结构化结果

`asktool` 是交互式异常：它有一个类型化的请求事件，等待
渲染器响应没有过期，并返回有界结构化工具
结果。停止回合可以解决跳过的未决问题。

## 4. 路径规则

本机文件和搜索工具强制执行不同的路径形状（D208、ADR 0069）：

- `Read.path` 是一个已存在的常规文件。传入目录会返回 `INVALID_ARGUMENT`
  并附一条结构化的 `Glob` 建议，而不是一个笼统的执行失败。
- `Glob.path` 是目录搜索根。
- `Grep.path` 可以是一个文件或一棵目录树。被直接点名的文件会被单独搜索，
  不去遍历它的同级；`include` 仍然过滤它的基本名，各项输出预算也保持不变。
  Grep 优先使用 PATH（以及 Unix 登录 PATH）上用户安装的 `rg`，当 `rg` 缺失
  或失败时回退到进程内搜索器（D315）。面向模型的契约不变。

Agent 模式下 `Glob`/`Grep` 依 D185 保持延迟激活。每一条新的用户提示都会重置
它们的激活状态，因此目录发现要在那条提示里通过 `ToolSearch` 激活 `Glob`，
而不是猜一个文件名，或对着目录调用 `Read`。

运行时为每个规范参数名接受一个别名，并在主机看到调用之前把它折叠掉（D273）：

| 工具 | 规范名 | 接受的别名 |
|---|---|---|
| `Read` / `Write` / `Edit` / `BrowserPreview` | `path` | `file_path` |
| `Glob` / `Grep` | `pattern` | `query` |

两种拼写在 schema 中都是可选的，运行时要求恰好提供其中一个；两个都没给的调用以
`INVALID_ARGUMENT` 失败。当一次调用同时带上两个时，规范名胜出。`Bash.timeout` 在
schema 中放宽到 100000000，好让毫秒值先通过校验：超过 21,600 秒上限的值按毫秒读取并换算成
秒，再夹到 21,600 秒（D273 / D329）。范围内的值（包括 600 和 1800）按秒读取。

- 对于持久的 `sessionId`，`workspaceRoot` 由该会话已持久化的项目绑定解析而来。
  没有路径的临时会话则把它的 `workspaceRoot` 绑定到
  `<data_dir>/scratch/<sessionId>`。两种根都不会在执行时从可变的活动侧边栏
  标签页读取。
- 默认情况下，所有文件路径均相对于已解析的 `workspaceRoot`
- 标准化后，它们必须仍然驻留在工作空间内，除非
  调用收到显式外部路径许可
- `..` 转义和符号链接转义是外部路径请求，不是隐式的
  访问
- 符号链接会在权限批准之后、执行之前再解析一次，因此批准无法跳过规范化这一步
- 例外（D114）：会话 scratch 目录内的绝对路径，是 `Read`/`Write`/`Edit` 的
  第二个合法根——见 §4b。两个根都运行相同的词法 + 符号链接包含防御。
  `Read`/`Glob`/`Grep`/`Write`/`Edit` 只能通过下面的权限策略，去寻址这两个根
  之外的显式路径；被拒绝或未获批准的请求返回 `TOOL_DENIED`。

### 4a。显式外部路径权限

一个解析后落在会话工作区根与 scratch 根之外的显式 `path` 参数，是一项独立的
能力。主机会在通常的低风险自动放行判定之前先检查它：

- `auto` 不出卡片即允许该外部路径；
- `ask` 与 `accept-edits` 发出普通的权限卡；
- `allow-once` 只执行当前这一次调用，而 `allow-session` 沿用既有的按工具计的
  会话授予作用域；
- 拒绝、超时或取消，都绝不执行该操作；
- 相对的 `..` 逃逸与符号链接逃逸，适用与绝对路径相同的规则；
- 成功的外部 `Read`/`Write`/`Edit` 结果携带 `root: "external"` 和一个绝对的
  规范路径；外部的 `Glob`/`Grep` 匹配项也是绝对路径，因此这次访问在转录中
  仍然可见。

该例外只适用于那个显式的路径参数。它不会扩大工作区根、Bash 的工作目录，
或任何隐式的目录遍历。

## 4b.会话临时目录 (D114)

Temporary/intermediate 代理生成的文件（一次性脚本，下载
数据、草稿）不得弄脏用户的项目或其 git 状态。每个
session 在工作区之外获取一个临时目录：

```text
<data_dir>/scratch/<sessionId>/
```

粘贴进输入框的操作系统剪贴板文件与图片，会先由 Electron main 具体化到
`<data_dir>/scratch/<sessionId>/pasted/` 之下，然后它们的路径与元数据才被捕获为
临时的输入框引用。派发时，main 会拿来源对照会话的 scratch/项目根做校验。图片
还会被复制进内容寻址的 `<data_dir>/attachments/<sha256>` 存储；`input:
["text", "image"]` 的已知 pi-ai 模型会收到符合条件的图片块，而非视觉/未知模型
以及超大图片则收到 `@` 回退路径。它们与其他 scratch 数据使用同一套会话生命
周期，并且不会以二进制内容的形式进入工作区、artifacts 或持久化的提示。

- **寻址。** 在项目会话中，模型只能用绝对路径寻址 scratch；该路径会在系统提示
  中公布，相对的工具路径解析到项目工作区，`Bash` 会导出 `PI_SCRATCH_DIR`。在
  临时会话中，同一个 scratch 目录就是该会话的工作区根，因此相对的
  Read/Glob/Grep/Write/Edit/Bash 路径在那里可以直接工作，无需继承一个项目。
- **包含。** `resolve_tool_path` 先尝试工作区根，再尝试 scratch 根，对两者施加
  完全相同的双层防御（词法 `..` 归一化 + 规范化祖先的符号链接检查）。种在
  scratch 里的符号链接无法抵达工作区或任何其他地方。
- **权限。** `path` 在词法上位于会话 scratch 根之内的 `Write`/`Edit` 自动放行，
  不出权限卡——它们碰不到项目。词法检查只是跳过提示；执行仍然走完整的解析器，
  因此它不是一条逃逸路径。Plan 与 Goal 不暴露 Write/Edit，所以 scratch 的自动
  放行规则无法让这两个工具在其中可用。当权限模式允许时，契约模式下的 Bash 调用
  仍然可以创建或改动 scratch 数据。
- **Artifacts。** 成功的 scratch 写入不会记录进 `artifacts` 表；由 artifact
  驱动的文件标签页只代表工作区交付物，而 Files 界面仍然可以浏览当前活动的
  工作区。工具结果携带 `root: "workspace" | "scratch"`，把这个判定和 UI 渲染
  都显式化。
- **工具覆盖。** `Read`/`Write`/`Edit` 使用工作区根与 scratch 根；`Glob`/`Grep`
  默认使用工作区根，也可以搜索一个被显式限定的 scratch 目录，或一个被显式批准
  的外部目录。模型应当使用有界的原生搜索工具，而不是用 shell 遍历目录。
  `BrowserPreview` 在 v1 中仍然是工作区相对的。它的主进程处理器从发起的持久
  会话解析根，渲染器事件携带 `sessionId`；被选中的前台工作区绝不会被用于一次
  后台预览。
- **生命周期。** 在一个会话的第一次 `Write`/`Edit`/`Bash` 或输入框剪贴板粘贴时
  惰性创建。随 `session.delete` 删除。启动扫描会移除那些会话已不存在的 scratch
  目录，以及超过 7 天未被触碰的目录（崩溃/强制退出的兜底；不需要定时任务）。
- 项目切换不会重定向或取消后台会话的工具；会话 A 与 B 分别沙箱在项目 A 与 B 中。
- 临时/无路径会话只把它自己的 scratch 目录当作工作区根，即使另一个项目可见或
  刚刚活跃过。它绝不继承那个项目。Plan 与 Goal 仍然要求一个已持久化的项目根，
  因此这条绑定不会扩大契约模式的执行范围。高风险工具只在该临时会话的 scratch
  根之内运作。
- 无法解析到持久会话的旧调用，只能在兼容窗口期内使用被选中的主机工作区。
- 会话的 lookup/storage 错误会让工具请求失败；它绝不能被当成一个缺失的旧会话，
  也不能被重定向到被选中的工作区。

## 4c。消息拥有的审核快照和回滚

`Write` 和 `Edit` 是结构化审核边界。对于一个成功的
工作区根突变，host-core 在执行前捕获前一个文件
并向工具结果添加有界审查证据：

```ts
type ReviewChange = {
  version: 1;
  snapshotId: string;
  messageId: string;
  path: string;
  operation: "write" | "edit" | "delete";
  status: "added" | "modified" | "deleted";
  state: "active" | "rolledBack";
  additions: number;
  deletions: number;
  hunks: DiffHunk[];
  binary?: boolean;
  truncated?: boolean;
  reversible: boolean;
};
```

- 渲染器保留并显示此记录以及工具消息；它
  不会从 Git、`HEAD` 或当前脏树重新计算 Review。
- 临时根、失败、拒绝和无法解析的写入没有 `review`
  记录。二进制或超大内容可能会省略大块并且是不可逆的。
- 回滚由主机拥有并受哈希保护。它恢复以前捕获的
  字节，或删除新创建的文件，仅当当前内容仍然存在时
  等于后工具哈希。稍后的编辑将返回 `conflict` 而不进行触摸
  文件。完成的回滚会使该路径的会话快照条目失效，
  因此模型无法继续针对回滚已替换的 tag 进行编辑。
- 携带 `MV DEST` 的 `Edit` 在一次工具调用下记录两条条目——源删除
  和目标创建——回滚要么同时恢复两者，要么都不恢复。`REM`
  记录为一次删除，其回滚恢复已捕获的字节。哈希保护使用
  完整摘要，而不是 16 位 `tag`。
- 查看工作区之外的实时快照文件，并随其一起删除
会议;孤立会话目录在主机启动时被清除。

## 4d。突变排序和编辑恢复

`Write` 和 `Edit` 在每个会话中序列化。 Read/search 工具可能
并行继续，不同的会话可能会变异不同的根
同时，但一个会话永远不会有两个正在进行的突变。主持人持有
在消耗全局突变槽之前每个会话突变允许，所以
排队的突变在等待早期编辑时无法保留容量。

`Edit` 命名位置并且只提供新内容；它从不匹配已有文本。
每次调用都携带由最后显示该内容的工具生成的整文件 `tag`，
当 tag 无法哈希出实时文件、或锚点引用了本会话从未显示过的
行时，主机拒绝该调用。完整契约——tag 计算、会话快照存储、
操作语法、块解析、寄存器与漂移恢复——见
[18-line-anchored-edit-contract](18-line-anchored-edit-contract.md)；本节只保留
排序与循环保护规则。代理突变工作流程是：

1. 当交付物位于已公布的工作区之内时，直接用 `Edit` 或 `Write` 编辑它。
2. 如果专用的 worktree 位于那个根之外，就用 Bash 在该 worktree 中执行一次
   受保护的编辑，并验证产生的 diff。
3. 编辑或补丁检查失败之后，对当前目标执行一次全新的 `Read`，并重新生成一次
   改动。一旦某条路径用完它的恢复额度
   （18-line-anchored-edit-contract §9.3），该提示符中针对该路径的下一次失败
   `Edit`——或第二个失败的 shell patch 命令（`apply_patch`、
   `git apply` 或 `patch`）——返回终止工具结果，因此代理
   报告确切的不匹配后停止。不要手动编辑旧的统一差异
   大块标头或继续修复循环。
4. 保持一条路径的突变是连续的，即使 read/search 调用是
   并行发行。

其 reveal 完整的 `EDIT_LINES_UNSEEN` 拒绝不受第 3 步重新读取的约束：
错误本身已经显示了缺失的行并将其并入会话来源集，因此原样重试
同一个 `tag` 即可应用。那次重试同时也是 `EDIT_LINES_UNSEEN` 在该路径上唯一的
宽限，因此第二次确实会计入保护限额。

序列化同时保护快照存储，生产者与 `Edit` 都会修改它：没有按会话的
突变许可，一次并发记录可能落在校验与写入之间。

sidecar 的工具计时线包括 `mutationFailureKind` 和
`mutationFailureAttempt` 用于失败的同路径 `Edit` 调用和已识别的 shell
修补命令，对按 §9.3 被宽限的失败使用 `mutationFailureGrace=true`，并在耗尽额度
的那次失败上使用 `terminate=true`。最后一项是
通过 pi-agent-core 的仅运行时终止提示；它不会改变
耐用的工具结果形状。由于该提示会结束代理循环，runtime 还会用
`MUTATION_RETRY_BUDGET_EXHAUSTED` 敲定该 assistant 行，而不是让本轮无声完成。

## 5. Bash 规则

主机执行基线：

- 需要一个项目绑定的会话工作区
- 默认 cwd = 原始会话的 `workspaceRoot`
- 默认需要确认
- 设置强制 60 秒超时；接受 1 秒至 21,600 秒覆盖（D329）
- 分别流式传输 stdout 和 stderr，然后返回有界的最终输出
- 截断大输出而不混合两个流
- 没有交互式 TTY (MVP)
- 以非零值退出的命令返回 `ok: false`、`isError: true` 和
  `errorCode: TOOL_FAILED`，同时保留其 `exitCode`、stdout 和 stderr
  在 `content` 中，以便代理可以诊断命令而无需盲目重试。

Shell 目录 (D190) 公开稳定 ID `windows-powershell`、`cmd`、
平台支持的 `git-bash` 和 `bash`。楼主坚持
`defaultCommandShell`；如果那个持续的选择后来变得不可用，
有效的目录选择有意回退到第一个可用的目录
平台外壳。一轮固定有效的 shell ID 和方言。 `Bash` 仍然存在
tool/protocol 名称，请求中单独携带固定的 shell ID。
主机核心在生成之前再次解析该条目并拒绝更改的 ID/dialect
与 `COMMAND_SHELL_CHANGED`；设置写入拒绝不可用或
`COMMAND_SHELL_INVALID` 的平台 ID 错误。没有任意可执行路径
或可执行路径哈希被接受作为 shell 标识。

1. `PI_DESKTOP_BASH` env 覆盖（bash 可执行文件的路径）
2. Unix：众所周知的位置（`/bin/bash`、`/usr/bin/bash`、`/usr/local/bin/bash`、Homebrew），然后是 PATH
3. Windows：来自 Git 的 Windows 的 `bash.exe` — 派生自 PATH 上的 `git`，然后是标准安装目录，然后是排除 `System32` 中的 WSL 启动器的 PATH

- Unix 调用 `bash -lc`（登录 shell 为 Finder/Dock 启动保留配置文件路径）； Windows 使用 `CREATE_NO_WINDOW` 调用 `bash -c`
- 在 Unix 上，Bash 工具另外探测用户的登录 shell 以获取其信息
  路径 — `$SHELL`（回退 `/bin/zsh` → `/bin/bash` → `/bin/sh`）
  `-lic 'printf %s "$PATH"'`，5 秒范围内，每个进程缓存 — 并注入它
  进入每个子流程。 `bash -lc` 仅提供 *bash* 配置文件；上
  macOS 默认 shell 是 zsh，因此 nvm/pnpm/Homebrew 初始化于
  否则，`~/.zshrc` / `~/.zprofile` 对代理命令将不可见。
  探测是尽力而为：缺少 shell、非零退出或超时回退
  主机 PATH 不变。 Agent 命令保留 POSIX bash (D181 / ADR 0045)。
- 安装程序中没有捆绑 bash：Windows 的 Git 是 Windows 的先决条件（无论如何，该应用程序都需要 git）
- 解决失败返回稳定的 `SHELL_NOT_FOUND` 并提供安装指导
- Windows PowerShell 和 cmd 使用其本机非交互式调用。
- Git Bash 使用发现的 Git 来执行 Windows 可执行文件。
- Unix Bash 使用经过批准的系统 Bash 条目。
- 用户中止和超时在返回之前终止整个进程树。

初始拒绝名单（可扩展）：

- 直接reading/writing工作空间外的敏感路径
- 未经确认的破坏性操作（由权限层控制的策略）

## 6. 权限模型

### 风险级别

| 风险 | 示例 | 默认政策 |
|---|---|---|
| 低 | 会话根目录内的 Read/Glob/Grep | 自动允许 |
| 中等 | 低风险 network/metadata | 政策确认或允许 |
| 高 | Write/Edit/Bash | 默认确认 |

### 决策类型

- `allow-once`
- `allow-session`
- `deny`

稍后可能会添加：
- `allow-always-for-tool`
- `allow-always-for-command-pattern`

### 权限模式 (D115/D132)

高风险工具调用如何获得批准由**权限模式**控制：

| 模式 | Write/Edit | Bash / 插件工具 |
|---|---|---|
| `ask`（默认） | 确认 | 确认 |
| `accept-edits` | 自动允许 | 确认 |
| `auto` | 自动允许 | 自动允许 |

显式的工作区外部路径是低风险那一行的例外：它只在 `auto` 下自动放行；
`ask` 与 `accept-edits` 都会要求确认。

每次工具调用的解析顺序（host-core 的 `tools.execute`）：

1. 会话已持久化的 `permission_mode`，除非它是 `inherit`
2. 应用设置中的全局 `defaultPermissionMode`（`ask` / `accept-edits` / `auto`）
3. `ask`

规则：

- 会话值存储在 `sessions.permission_mode`（`inherit | ask | accept-edits |
  auto`，默认 `inherit`，schema v5），并通过 `session.configure` 的
  `permissionMode` 设置。
- 对于 Write/Edit 和插件工具，Plan 的硬拒绝胜过任何权限模式。`auto` 无法
  重新启用一个被隐藏或被拒绝的工具。
- 会话根目录之内的低风险工具（`Read`/`Glob`/`Grep`）在每种模式下都自动放行，
  与此前一致。
- `BrowserPreview` 是一项显式的只读 UI 检查能力，在两种操作模式下都可用。
- Plan 保留权限模式选择器。Bash 在 `ask` 与 `accept-edits` 下需要确认，在
  `auto` 下自动放行；因此 Plan 表达的是规划意图，而不是一份严格的只读安全画像。
- `allow-session` 授予在 `ask` 下继续有效，且作用域仍限于该会话；在
  `accept-edits`/`auto` 下则根本用不上它们。
- scratch 目录写入（D114）在每种模式下都保持免提示。
- UI：设置 → 分段式全局默认；输入框在 Agent、Plan 和 Goal 中显示一个按会话的
  chip，它的菜单提供三种有效模式，不设单独的 global-default/inherit 条目。
  chip 与被选中的菜单项显示的是有效模式；选中某一项会存下那个显式的会话覆盖。
  已有的继承式会话会继续走全局设置解析，直到用户自己选定一种模式。
- 强制执行只存在于 host-core；sidecar/模型从不被告知该模式，也无法影响它。

## 7. 权限流程

```text
tool call
 → policy.evaluate()
 → allow? execute
 → need confirm? push to UI
 → deny? return tool error result
```

权限确认超时：
- 120秒后，自动拒绝（D005：失败关闭，不永远挂起）

## 8. 工具结果对模型的可见性

- 成功结果：给予模型
- 失败结果：提供给模型（带有错误信息）
- 用户拒绝：给模型明确的“用户拒绝权限”
- 敏感信息：在 persisting/displaying 之前进行编辑

## 9. 审计

各工具调用记录：

- 会话ID
- 转号
- 工具呼叫ID
- 工具名称
- 参数哈希/预览
- 存在显式路径时的 externalPathPermission 分类
- 决定
- 持续时间
- 成功/错误代码

MVP 可以通过写入 SQLite 或日志文件来启动。

计时以分段记录，而不是作为一个持续时间 (D183)：`prompted`
（是否出示许可卡）、`permissionWaitMs`、`durationMs`（
工具体）、`overheadMs`（主机簿记）和 `totalMs`。拒绝来电携带
具有零工具体的相同字段。参见
[09.日志记录和可观测性](/zh-CN/spec/03-runtime/09-logging-and-observability)
匹配日志行。

## 10. 操作模式矩阵

| 模式 | Read/Glob/Grep | BrowserPreview | Write/Edit | Bash | 插件 |
|---|---|---|---|---|---|
| Agent | 允许 | 允许 | 按权限策略 | 按权限策略 | 按注册的风险策略 |
| Plan | 允许 | 允许 | 拒绝 | `ask`/`accept-edits`：确认；`auto`：允许 | 拒绝 |
| Goal | 允许 | 允许 | 拒绝 | `ask`/`accept-edits`：确认；`auto`：允许 | 拒绝 |

### 注释
- 权限 UI 之前的 Plan 和 Goal 硬否认 Write/Edit/plugin 工具；直接主机
  调用不能绕过矩阵。
- Agent 模式使用权限卡或选定的自动策略
  Write/Edit/Bash 和注册的插件工具。
- 当用户选择“自动”时，Plan 和 Goal Bash 可能会改变工作区或暂存状态；
  用户界面必须使这种权衡可见。
- 仅针对活动会话记住每个 toolName 的允许会话
- 会话授权遵循 `sessionId` 跨项目选项卡开关，并且永远不会
  由另一个会话或临时会话继承

### 10. 1 Plan 和 Goal 控制工具

`new_context` 在每种模式下都可用，无需确认：它只询问
在下一回合边界压缩的运行时间，主机将在其上执行此操作
一旦达到硬预算（参见
[02-代理运行时](/zh-CN/spec/03-runtime/02-agent-runtime) §5.1)。提交
工具仅在其自己的合同模式下可用，并且必须是其辅助批次中唯一的工具调用。它保留了
类型目录下新的独特工件中的确切 Markdown 字节
（`.pi/plan/*.md` 为 `SubmitPlan`，`.pi/goal/*.md` 为 `SubmitGoal`）
在创建一项待批准之前通过 host-core。 `EnterPlanMode` 和
`EnterGoalMode` 仅在 Agent 中可用，并且每个工具调用必须是唯一的
在其批次中。主机验证持久模式、提案类型和
任何转换之前的 active-turn/configuration 边界；可见的工具列表
是指导，而不是安全边界。

### 10. 2 委派和子代理工具范围（D201、ADR 0062）

`Task` 仅在 Agent 模式下可用，且仅当会话至少拥有一个子代理定义时可用。
Plan 与 Goal 是只读的契约协商，因此一个带着 `Bash`、`Edit` 或 `Write` 的委托
会径直冲过它们。

定义声明其委托可以调用的工具，只能取自这七个工作工具：`Read`、`Glob`、
`Grep`、`BrowserPreview`、`Bash`、`Edit` 和 `Write`。什么都不声明的定义得到
`Read`、`Glob`、`Grep`；`tools: "*"` 表示全部七个工作工具。无法识别的名称——
包括已撤销的 `A2A` 与 `Peer` 工具（D326 / ADR 0165）——会被丢弃并留下一条解析
警告。插件工具、`Skill`、`ToolSearch`、`new_context`、模式工具，以及 `Task`
本身，永远不可分配：委托是一个有界的 file/search/shell 工作者，而不是第二个
会话。

委托可用的工具来自其定义，而不是其会话。它无法因为父级拥有某个工具而获得
该工具，会话也不能把修改权限借给只读委托。委托调用由会话运行时构建并通过相同
的 `tools.execute` 路径，因此路径规则（§4）、Bash 规则（§5）、权限模式（§6）、
操作模式矩阵（§10）和审计（§9）保持不变，并针对拥有该调用的会话评估。

**内置定义与用户定义**还可以声明 `permission: inherit | ask | accept-edits |
auto`（ADR 0089，默认 `inherit`）。使用默认的 `inherit`（包括所有内置定义）时，
sidecar 不附加覆盖，委托使用会话的有效权限模式；因此父会话为 `auto` 时，明确的
外部路径也不会再次弹出授权卡。项目定义随仓库一起到来，其 scope 会在解析时被丢弃
并留下警告，克隆仓库永远不会获得权限升级。当一个合格的内置或用户定义显式声明了非 `inherit` 的作用域时，sidecar 会把它
附加到该委托的 `tools.execute` 调用上，host-core 则在那个模式下裁决每一次调用，
而不是按会话的有效权限模式。scope 只是
权限模式覆盖：契约模式的硬拒绝和外部路径门禁（§4.1）仍然生效；显式
`accept-edits` 仅自动允许工作区和 scratch 根内的 `Write`/`Edit`，外部路径及其他
工具仍按其正常审批边界处理。

来自委托的权限请求会携带发起请求的那个委托的名称，因此卡片可以说明是哪个委托
想要这次调用（参见 `04-ux/03-permission-ux.md` §6a）。
会话范围的 `allow-session` 授予仍然是按 `toolName`、按会话计的：某一个委托对
`Bash` 的批准适用于整个会话，包括父级和其他委托。

## 11. 插件工具

插件只能通过 Agent 中的 `agentTools` 提供工具：

1. 舱单声明
2. 用户授予 `agent.tool.register`
3.PluginManager将它们注册到ToolHost中
4.执行经过统一的permission/audit/timeout包装器

无论清单如何，Plan 或 Goal 中均不可见或可执行任何插件工具
风险、声明的权限、会话授予或 `auto`。直接尝试返回
`PLUGIN_DISABLED_IN_PLAN` — `_IN_PLAN` 代码由两个合约共享
模式而不是每种重复 - 并且作为合同模式政策进行审核
否认。对于 Agent，缺失或无效的插件风险默认为 `medium`，并且从不
授予合同模式访问权限。

命名：
- 内部全名：`plugin.<pluginId>.<toolName>`
- 暴露给模型的名称：强制前缀 `plugin_<pluginIdSafe>_<toolName>` (D015) 以避免冲突

## 12. 未来的扩展

- MCP 工具
- 工具组切换
- 命令允许列表/拒绝列表
- 空运行模式
- 预览后应用补丁
