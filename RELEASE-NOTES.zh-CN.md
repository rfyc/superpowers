# Superpowers 发布说明

## v6.3.0 (2026-08-12)

### 宿主环境支持

- **Devin CLI**：`devin plugins install obra/superpowers` 现在可以正常工作，技能会在会话启动时自动触发。（#1995）
- **Hermes Agent**：从 git 克隆安装；技能通过 Hermes 的原生加载器注册，引导程序在第一个回合加载。（#1922、#2025）
- **Grok Build CLI** 已添加到安装文档中。（#1919）

### Brainstorming

- **仪式感现在随任务规模伸缩。** 请求被分类为 spike、bounded 或 architectural；小任务跳过双文档仪式。每条路径在实现前仍会停下来等待你的批准。（#2063）

### Subagent-Driven Development

- **控制器不再因计划冲突而停滞。** 非灾难性冲突和歧义会得到已记录在案的裁决，工作继续进行；只有破坏性或不可逆的操作仍会停下来等待人类。有一个捐赠的会话在一个控制器本可以自行决定的问题上被阻塞了近九个小时。（#2077）
- **派发前的冲突扫描会把它的检查记录到账本（ledger）中**，而不是仅仅断言计划是干净的。（#2080）
- **小的同形状任务会批量为一次派发**，大幅降低微任务计划的 subagent 成本；批量审查会验证 brief 中的每个文件都进入了 diff。（#2078）
- **实现者和审查者不得派生自己的 subagent**，这曾导致重复审查。（#2059）
- **计划携带 `Spec:` 指针**，SDD 在设置时读取该规范，因此计划冲突会对照设计来解决，而不是靠猜测。（#2086）
- 审查者会重新阅读他们觉得难以辨认的证据，而不是重新运行测试套件（#2089），熔断器（circuit-breaker）裁决现在会出现在 Finish 报告中。

### Codex

- Subagent 等待改为事件驱动而非高频率轮询，派发会显式固定模型和推理强度，多代理参考也已对照 Codex 源码更正。（#2060、#2061、#2062）

### Finishing a Development Branch

- **移除 worktree 不再破坏未跟踪文件。** 当 `git worktree remove` 因为树中存在未提交的工作而拒绝时，该技能会停下来、点名这些文件并提出询问——而不是动用 `--force`。（#2016、#1223、#2024）

### 修复

- writing-skills 中的 `render-graphs.js` 在 Windows 上正常工作。
- 更正了 Windows 上的 Copilot CLI 后台化指导。（#1929、#2006）
- `bump-version.sh` 现在覆盖 Hermes 清单。

### 文档

- README：新增目录，并重新组织了"开始使用"部分。

## v6.2.0 (2026-07-23)

### Subagent-Driven Development

SDD 跟踪进度和收尾审查发现的方式发生了两项结构性变化，两者都是针对活跃的评估活动开发的。

- **工作区现在是计划作用域的（plan-scoped）。** `.superpowers/sdd/` 没有计划身份，也没有生命周期终点：同一个工作树中的后续计划可能把前一个计划的账本当作自己的进度来读（在真实世界中观察到过，伴有多次污染轮次和临时变通方法）。`sdd-workspace` 现在要求提供计划文件，并解析出按计划划分的目录 `.superpowers/sdd/<plan-basename>/`；`task-brief` 和 `review-package` 写入它们计划的目录（`review-package` 将计划文件作为其第一个参数）；账本在第一行注明其计划名称；一旦最终审查干净，工作区就会被删除——git 历史才是持久记录。基线评估显示控制器已经会拒绝外部账本，但要付出每次恢复时 6-13 次跨计划 git 取证工具调用的代价；计划作用域使答案变成结构性的。（25/25 基线和 GREEN 评估运行记录在 `docs/specs/` 和 `docs/plans/` 中。）
- **审查-修复循环现在恢复实现者。** 生命周期重组赋予修复轮次"恢复实现者"语义而非全新派发，新增了一个范围限定的重新审查提示（`re-review-prompt.md`），使重新审查者检查修复而不是重读整个任务，并安装了一个五轮熔断器，触发时由控制器裁决。SKILL.md 按生命周期重组，其 Red Flags 转换为标准合理化表格形式。

### 技能

一次覆盖整个分支的压缩行动：针对已经调用过技能的读者而写的回顾部分、社会认同和卖点宣传散文都被删除，每一个承重的论证都被折叠进合理化表格行，或移到其使用点。每次删减都用 subagent 探针进行了微测试，唯一一次可测出行为劣化的删减被返工而不是发布。

- **`testing-anti-patterns.md` 现在是 `writing-good-tests.md`。** TDD 参考文档被重建为一份正面清单——六条规则都以 GOOD 示例开头——并吸收了一项可证伪性纪律：命名会让测试失败的产线改动，独立于被测代码推导期望值，再加上一个收尾的变异检查。它按名称堵住了两个漏洞：字符串存在陷阱（对脚本、技能和提示进行 grep 式测试伪造了可证伪性——可观察的是行为，绝不是文本）和变更检测器陷阱（常量断言可以失败却不保护任何东西），每个都在门控函数中设置了硬停止。平凡代码和人类散文不值得写测试；触发器从"添加 mock"拓宽到任何测试编写。
- **TDD 的"Why Order Matters"反驳以合理化行的形式存活下来。** 在"直接写，测试放后面"的压力下，直接删除该部分可测地降低了先测试行为（对照组 8/10 → 处理组 5/10，在 Claude 和 Codex 上都得到佐证），所以每条散文式反驳现在都放在它的 Common Rationalizations 行中——部分没有了，但论据会在代理陷入合理化时触发。
- **`finishing-a-development-branch` 不再主动提供丢弃你的工作。** 完成菜单起源于扔掉分支是常规操作的年代；"Discard this work"与"Merge"并列，等于在宣传销毁已完成的、通过测试的工作。丢弃保留为仅限显式请求的路径，并沿用相同的打字确认仪式。同一次修订还让 PR 创建变得与 forge 无关（使用你的 forge 的 CLI 或推送时打印的 URL，而不是一份受祝福的工具清单），并修复了一个真实 bug：worktree 路径在清理已经改变目录之后被重新计算，因此溯源检查从不匹配，清理静默地变成无操作。
- **整个库移除了回顾和说服性散文。** `brainstorming`、`systematic-debugging`、`dispatching-parallel-agents`、`verification-before-completion`、`executing-plans`、`subagent-driven-development`、`requesting-code-review`、`receiving-code-review`、`using-git-worktrees`、`writing-plans` 和 `writing-skills` 都删除了它们的 Bottom Line / Key Principles / Real-World Impact / Advantages 部分；`using-git-worktrees` 和 `finishing-a-development-branch` 把它们的守卫部分转换为标准的 Excuse/Reality 合理化表格。

### Windows

- **SessionStart 钩子现在通过 Git Bash 派发。** 钩子的命令字符串以带引号的路径开头，这破坏了 Claude Code 可能交给它的两个 shell：PowerShell 把带引号的字符串解析为表达式并死于解析错误（#1751），而 cmd.exe 的引号剥离规则在配置文件路径包含 `(` 之类的元字符时截断命令（#1918）——无论哪种情况，引导程序都静默地从未加载。钩子现在声明 `shell: "bash"`，Claude Code ≥ 2.1.81 会将其解析为直接使用 Git for Windows，并且在缺少 Git Bash 时呈现一条可操作的安装提示。旧版 Claude Code 会忽略这个未知键，行为与以前一样。已在 Linux、带有恶意路径下 Git Bash 的 Windows 11 以及没有 Git Bash 的 Windows 11 上进行了端到端验证。

### 宿主环境支持

- **恢复 Gemini CLI 支持。** v6.1.0 中的移除（据称 Google 已 EOL 了 Gemini CLI）为时过早；安装文档和 `gemini-tools.md` 工具映射参考已恢复，而永久移除需要经过适当的评估。（#1959）

### 修复

- **`find-polluter.sh` 现在真的能找到测试文件了。** `find .` 会输出 `./` 前缀路径，因此文档中描述的 `-path "src/**/*.test.ts"` 模式匹配不到任何东西——而对空输入执行 `wc -l` 后又报告"Found 1"。修复了前缀不匹配（#2008、#2011），外加两个后续问题：调用方提供的 `./` 前缀模式不再双重加前缀而变成永不匹配的形式，`**/` 也被折叠匹配，因此直接在基础目录下的测试（`src/top.test.ts` 对比 `src/**/*.test.ts`）不会被静默跳过。该脚本获得了一个确定性的测试套件。
- **Codex 包脚本在 macOS 之外也能工作。** 确定性元数据 tar 标志是 bsdtar 专属的拼写，暂存文件模式依赖两个 umask 相互抵消，而测试的时间戳断言以美国时区解析了 bsdtar 的列布局。GNU tar 现在获得等效标志，产生字节相同的头，模式被固定为规范形式，测试通过 `tarfile` 断言 mtime。
- **SDD 的技能测试不再抖动。** 该文件的最坏情况超过了运行器的单文件上限（提高到 900 秒），而断言辅助函数对自由形式的模型散文进行区分大小写的匹配；现在匹配不区分大小写，`assert_order` 在失败时转储输出，使下一次抖动可诊断。
- **v6.1.0 参考修剪后的文档和测试清理。** 指向已删除的 `claude-code-tools.md`/`copilot-tools.md` 的死链被替换为当前架构（#1969），Antigravity 参考中悬空的 `#subagent-support` 锚点被移除（#2010），Antigravity/Pi 映射测试现在只断言存活的宿主环境特定映射——作用域限定到表格，因此它被删除时测试会再次失败。

## v6.1.1 (2026-07-02)

### Codex

- **Codex 不再重新注册 Claude SessionStart 钩子。** v6.1.0 移除了 Codex 钩子配置及其清单中的 `hooks` 指针，本意是阻止 Codex 安装 SessionStart 钩子——但因为没有 `hooks` 字段，Codex 回退到自动发现 `hooks/hooks.json`，即市场随仓库根目录一起发布的 Claude Code SessionStart 钩子，并连同其安装时的信任提示一起重新注册了它。Codex 清单现在声明一个显式的空 hooks 对象（`hooks: {}`），Codex 会将其解读为"无钩子"，而不是走自动发现回退。字段缺失、`[]` 和空的内联列表都会塌缩回回退路径，所以这个值必须是恰好 `{}`。
- **移除了孤立的 Codex session-start 死代码。** `hooks/session-start-codex` 在 Codex 钩子配置被删除后没有调用者，所以它和它冗余的测试用例都不见了。`docs/porting-to-a-new-harness.md` 中完整的 shell 钩子示例从 Codex——现在是原生技能发现、无 session-start 钩子——移到 Cursor（一个活跃的 shell 钩子宿主环境），`docs/windows/polyglot-hooks.md` 中过时的 `hooks-codex.json` 指针也被更正。Codex 插件类别也修复为"Developer Tools"。

### 打包

- **用于构建 Codex portal 包的新 `package-codex-plugin.sh`。** 一个维护者脚本会生成确定性的 Codex "portal" 归档——默认 `.zip`，按需 `tar.gz`——它规范化条目时间戳、保留可执行模式、验证每个打包的技能都附带其 OpenAI 元数据、包含 app 和 composer 图标，并拒绝在脏工作树上运行。打包的清单保留源码中的 `hooks: {}` 对象，因此通过 portal 安装的插件会避开同样的 SessionStart 自动发现，而且该脚本可以从保存的元数据源重建字节相同的归档。由一个新的测试套件覆盖。

## v6.1.0 (2026-06-30)

### 更低的每会话 token 成本

`using-superpowers` 引导程序被注入到每个会话中，因此它的体积是持续付出的成本。本版本修剪了它及其指向的每个宿主环境参考，没有丢弃塑造行为的内容。

- **压缩了 `using-superpowers` 引导程序。** 用它所编码的散文替换了 graphviz 技能流程图，把独立的 Instruction-Priority 部分折叠进 User Instructions，删除了按平台的"How to Access Skills"演练，并把 Platform Adaptation 指针裁剪到仍然附带参考文件的宿主环境。完整的 Red Flags 合理化表格和用户指令优先级规则不变。
- **修剪了每宿主环境的工具映射参考。** 冗长的动作到工具表格复述了现代代理已经遵循的指导。每个参考文件都被裁剪到仍然有分量的宿主环境特定说明——subagent 派发、任务跟踪、指令文件路径——而已经没有任何宿主环境特定内容可留的 `claude-code-tools.md` 和 `copilot-tools.md` 被删除。

### Codex

- **Codex 可以从市场安装。** Codex 市场源期望在市场根目录有一个 `.agents/plugins/marketplace.json`；仓库只发布了 Claude 市场文件，所以 Codex 能指名这个市场，却找不到可安装的插件条目。一个仓库本地的 Codex 市场清单现在指向同一个仓库根目录，因此插件可从 Codex 安装。
- **Codex 不再附带 SessionStart 钩子。** Codex 自己就能可靠地触发技能，引导钩子让 UX 变得更糟而不是更好。Codex 钩子配置（`hooks-codex.json`）及其清单注册被移除。

### 宿主环境支持

- **移除 Gemini CLI 支持。** Google 于 2026-06-18 EOL 了 Gemini CLI；该扩展无法再安装或更新。Gemini 从安装文档、支持 subagent 的平台列表和评估宿主环境描述中消失，其工具映射参考也被删除。

## v6.0.3 (2026-06-18)

### Subagent-Driven Development

- **SDD 临时文件移出了 `.git/`。** Claude Code 把 `.git/` 视为受保护路径并拒绝代理在那里写入，所以一个把报告写进 `.git/sdd/` 的实现者 subagent 会在运行中途被阻止。任务 brief、实现者报告、审查 diff 和进度账本现在位于工作树中一个自忽略的 `.superpowers/sdd/` 目录——被排除在 `git status` 和提交之外，并由共享的 `sdd-workspace` 辅助程序按 worktree 解析。一个注意事项：因为工作区是被 git 忽略的工作树临时文件，`git clean -fdx` 会删除进度账本；如果发生这种情况，从 `git log` 恢复。（#1780）

## v6.0.2 (2026-06-16)

### 安装修复

- **我们不再附带 `evals` 子模块。** 它破坏了一些用户的插件安装，所以评估宿主环境现在位于它自己的仓库中，与已发布的插件分开。（#1778、#1774）

## v6.0.1 (2026-06-16)

### Codex 修复

- **头脑风暴伴侣中的版本显示** —— 打包的 Codex 插件不带根 `package.json` 发布，所以可视化伴侣报告它的版本为"unknown"。当 `package.json` 不存在时，`readSuperpowersVersion()` 现在回退到 `.codex-plugin/plugin.json`。
- **更干净的 Codex 插件同步** —— 同步到 Codex 的脚本现在排除 `.gitmodules` 和 `.pre-commit-config.yaml`，使仓库元数据不进入打包的 Codex 插件。

## v6.0.0 (2026-06-16)

Superpowers 6.0 是一个大版本。头条是重写了 `subagent-driven-development` 审查每个任务的方式——更便宜、更严格、更难钻空子。

虽然这些数字不会在每个宿主环境和每种工作负载上都成立，但在我们的评估中，Claude Code 和 Codex 产生类似的高质量结果，速度大约快一倍，同时花费的 token 少了近 50%。

它还新增了三个宿主环境（Kimi Code、Pi 和 Antigravity），为 brainstorming 可视化伴侣提供了更好的安全模型，并重写了大量技能的工具调用，使其明显更与厂商无关。

### 可见的变化

- **两个每任务审查者提示合而为一。** `spec-reviewer-prompt.md` 和 `code-quality-reviewer-prompt.md` 已不存在，被单个 `task-reviewer-prompt.md` 取代。如果你直接派发旧文件，请切换到新的。
- **遗留的全局 worktree 目录已不存在。** `using-git-worktrees` 和 `finishing-a-development-branch` 不再使用 `~/.config/superpowers/worktrees/`。worktree 现在落在项目内——如果你已有 `.worktrees/` 或 `worktrees/` 就用它，否则新建一个 `.worktrees/`——除非你另有说明。

### 新宿主环境支持

Superpowers 现在可以在另外三个宿主环境上运行。每个都附带自己的引导程序、工具映射参考和测试，并且每个都在 README 中获得自己的安装部分。

- **Kimi Code** —— 一个插件清单、安装文档和清单测试；可从 Kimi 的市场或直接从这个仓库安装。（初始清单由 @qer 提供）
- **Pi** —— 一个会话启动扩展，注册技能并注入 `using-superpowers` 引导程序。Pi 拥有原生技能，因此不需要兼容性垫片。
- **Antigravity（`agy`）** —— 直接安装插件并从第一条消息开始引导；已针对标准的"make a react todo list"验收测试进行了端到端验证。

### Subagent-Driven Development

在真实项目上长期的成本与质量实验重塑了控制器审查每个任务的方式。旧流程每个任务运行两个审查者，并依赖控制器的判断来选择模型和严重程度，结果两者都既昂贵又容易被钻空子。新流程每个任务运行一个审查者，把工作作为文件而不是粘贴的文本移交，并把几项判断职责从控制器手里拿走。

- **每个任务一个审查者，两份裁决。** 单个 `task-reviewer-prompt.md` 读取一次任务的 diff，同时返回规格合规裁决和质量裁决，因此一次修复循环就能清掉两者。一个新的"无法从 diff 验证"裁决会标记位于未改动代码中的需求，留给控制器自行检查。（#1538、#1543）
- **结尾一次广泛审查。** 运行结束时，在最强的模型上对整个分支做一次审查，而不是逐任务重新审查一切。
- **计划获得预检读取。** 在第一个任务之前，控制器检查计划的内部冲突——以及计划要求的、审查者会标记为缺陷的任何东西——并一次性全部提出，而不是在运行中途绊上去。
- **diff 和任务文本以文件形式传递。** 粘贴的 diff 会永久停放在最昂贵的上下文中，而没有 diff 的审查者必须手工重建它——这是单个最大的审查者成本。两个新脚本 `task-brief` 和 `review-package` 把任务文本和审查 diff 写入文件供 subagent 读取。
- **每次派发都声明其模型。** 一旦由他们自己选择，控制器就干脆不命名模型了——而无名模型会悄悄继承会话中最贵的那个，所以一次运行把它的全部 26 个审查者都放到了最高档。模板现在要求提供模型，并附有在允许时触达更便宜档位的指导。
- **控制器不能告诉审查者忽略什么。** 真实运行抓到过控制器教唆审查者跳过某个发现或将其称为"至多 Minor"，而这个缺陷就这样发布了。压制发现和预先评定严重程度现在被明令禁止，计划本身规定的缺陷会被上报给你来决定，而不是挥手放过。
- **审查者是只读的，并怀疑各种理由。** 审查不再触碰工作树或分支——曾经有审查者运行 `git checkout`，导致后来的提交成为孤儿——而实现者的"我故意不抽出来"不再能说服审查者放弃一个真实发现。
- **更强的证据和报告。** 审查者用文件和行号支持每个答案，实现者的报告移到文件中，并在适用 TDD 时携带红/绿证据，进度账本让失去上下文的控制器能够恢复而不是重做已完成的工作。（#994）

### Writing Plans

计划现在携带控制器和审查者过去每次派发都要重新推导的结构。

- **一个 Global Constraints 块** 列出约束每个任务的规则——版本下限、依赖限制、命名和文案、精确值——逐字复制，这样它们确实能到达下游的实现者和审查者手中。
- **一个每任务 Interfaces 块** 精确地命名每个任务消费和产出的内容，因此只看到自己任务的实现者仍然知道邻居的契约。
- **正确规模化指导** 让任务保持在足以赢得自己的测试循环和一次审查者通行的规模，把设置、配置和文档折叠进需要它们的任务。在测试中，用这种方式编写的计划只需要一轮修复，而对照组需要两到四轮——而且对照组还带出了一个真实 bug。

### Brainstorming 可视化伴侣

可视化伴侣是一个小型的 web 服务器，代理在对话旁边打开它。它完全没有认证，所以在共享或远程机器上，任何能触达端口的人都能读取你的头脑风暴——或注入代理视为你输入的的事件。本版本赋予它一个真正的安全模型，并让它能熬过重启和断连。

- **一个每会话密钥现在守护一切。** 代理的 URL 携带一次性密钥，浏览器把它藏进标签页作用域的 cookie 里，每个请求和 WebSocket 连接都必须出示它。这既关上了零散本地标签页的门，也关上了可路由远程主机的门，包括来源白名单无法捕捉的 DNS 重绑定情况。（关闭 #1014）
- **文件服务器留在它的沙箱里。** 它拒绝符号链接、点文件和任何爬出内容目录的路径，忽略 macOS 资源分叉文件，并发送通常的 no-store 和 deny-framing 头。持有会话密钥的文件以仅所有者权限写入。
- **伴侣只在有帮助时才被提供。** 技能在某个问题以展示方式呈现比以讲述方式更好读的第一次，以它自己的消息提出伴侣，并让一次拒绝就此定案。接受会在你的浏览器中打开第一个屏幕。（关闭 #755）
- **它能熬过重启和不稳定连接。** 给定一个项目目录，服务器在重启之间保持相同的端口和密钥，所以打开的标签页只需重新连接。页面自行重连，显示一个实时的状态胶囊，并在服务器宕机时浮起一个"paused"覆盖层。
- **更长的空闲寿命，更安全地关闭。** 空闲超时从 30 分钟改为 4 小时，`stop-server.sh` 现在在发信号前确认它拥有正确的进程，因此它永远不会在重启后杀掉一个无关的 `node`。（#1703）
- **Windows 启动加固** —— 整合了 shell 检测，Windows 现在依赖空闲超时来关闭，因为 Node 无法跨 MSYS2 跟踪 POSIX 进程所有权。

### 现有宿主环境更新

- **Codex** 现在通过自己的 SessionStart 钩子引导，而不是共享接线，Codex App 获得了一个安装部分和更完整的工具文档（web 搜索、`AGENTS.md`、个人技能）。（#1540）
- **OpenCode** 在其插件、安装文档和 README 中获得基于动作的工具映射，外加一个引导缓存测试。
- **Cursor** 的清单删除了它的 `agents` 和 `commands` 条目，因为这些目录已不存在。

### 一套技能，每个宿主环境

技能过去讲的是 Claude Code 的方言——"use the Task tool"、"put it in CLAUDE.md"。本版本把这种词汇改写为你实际在做的事情（"dispatch a subagent"、"your instructions file"），并新增一个每宿主环境参考，把每个动作映射到正确的工具，并对照每个运行时检查过。提到"Claude"的散文现在改说"your agent"。

- **每个宿主环境一份工具参考**，位于 `skills/using-superpowers/references/`，涵盖 Claude Code、Codex、Copilot、Gemini、Pi 和 Antigravity。
- **`finishing-a-development-branch` 变得与 forge 无关** —— 它不再硬编码 `gh pr create`，所以代理用他们拥有的任何 forge 工具推送。（#1609）
- **一次重命名：** "Claude Search Optimization"现在是"Skill Discovery Optimization"，因为这个技术不是 Claude 特有的。

### Writing Skills

面向技能作者的两项新增。

- **Match the Form to the Failure** —— 一张用于选择正确指导形式的小表格。平铺的"别做 X"适用于纪律滑坡，但当问题在于输出的*形态*时就适得其反，此时一个实操示例做得更好。这张表格，加上对现有合理化部分的更紧范围，引导作者选择真正有帮助的形式。
- **Micro-Test Wording** —— 一种在定稿措辞之前检查它的廉价方法：对照无指导对照组抽样若干次，并人工通读每个结果，把运行间的差异视为警告信号。

### 测试

技能行为测试从 `tests/` 移入一个基于"drill"的新 `evals/` 子模块，它运行真实的 Claude Code、Codex 和 Gemini 会话并用 LLM 评判。几个树内 bash 套件在更严格的 drill 场景覆盖它们后被退役；没有对应版本的少数几个被保留。从这里开始，`tests/` 存放插件代码测试，`evals/` 存放技能行为测试，`docs/testing.md` 解释了这种划分。新后端触达 Antigravity、Pi 和更多模型，新的 shell-lint 和 pre-commit 检查守护宿主环境。（#1541）

### 缺陷修复

- **systematic-debugging 不再把每个会话都逼进扩展思考。** 有一个条目恰好含有 Claude Code 扫描的关键字，在每个加载了该技能的会话上都悄悄触发了开关。一个连字符打破了该关键字；文本仍然可读。（#1283，由 @Nick Galatis 提供）
- **Windows SessionStart 钩子不再在每个会话打印一条写入错误** —— 每个 `printf` 现在都通过 `cat` 路由以吸收损坏的管道，除此之外输出不变。（#1612，由 @silvertakana 报告）
- **Windows 前台模式** 跟踪正确的进程，并在 MSYS2 上清除其所有者 PID。（由 @nestorluiscamachopaz 提供）
- **`using-superpowers` 引导程序** 不再把"debugging"列为一个不存在的技能。（由 @mhat 报告）
- **TDD 技能** 链接到测试反模式参考。（#1532、#1529；链接修复 #1474 由 @Stable Genius 提供）
- **`using-git-worktrees`** 修复了它的步骤编号，并删除了过时的 Cursor 引用。（#1522，以及 @fuleinist 提供）
- **Codex 审查技能** 用一个内部玩笑换来了朴实的指导。（#1531）

### 文档与贡献者指南

- **一份把 Superpowers 移植到新宿主环境的指南**（`docs/porting-to-a-new-harness.md`）列出了每个集成所需的三个组成部分，以及决定成败的一条规则：在会话启动时加载引导程序。
- **每个 PR 和 issue 现在披露它是如何制作的** —— 模型、宿主环境、版本和已安装的插件，或一段说明它是手工编写的文字。我们会根据贡献是什么产出的来权衡它。PR 也以 `dev` 而非 `main` 为目标。PR 模板、全部三个 issue 模板以及一个新的平台支持模板都承载了这一要求。

### 贡献者

感谢 @mattvanhorn、@nawfal、@Nick Galatis、@silvertakana、@nestorluiscamachopaz、@qer、@mhat、@Stable Genius、@fuleinist、@dev_Hakaze、@robotsnh、Rahul 和 @arittr。

## v5.1.0 (2026-04-30)

### 移除

- **移除遗留的斜杠命令** —— `/brainstorm`、`/execute-plan` 和 `/write-plan` 已不存在。它们是已废弃的存根，除了告诉用户调用相应技能外什么都不做。请直接调用 `superpowers:brainstorming`、`superpowers:executing-plans` 和 `superpowers:writing-plans`。（#1188）
- **移除 `superpowers:code-reviewer` 命名代理** —— 该代理是这个插件唯一的命名代理，只被恰好两个技能使用，而仓库中所有其他审查者/实现者 subagent 都是派发 `general-purpose`，并在其技能旁附一个提示模板。代理的人设和清单已合并进 `skills/requesting-code-review/code-reviewer.md`，成为一个自包含的 Task 派发模板。任何派发 `Task (superpowers:code-reviewer)` 的人应改用 `Task (general-purpose)` 加提示模板。（PR #1299）
- **从技能中移除集成部分** —— 这些是在代理拥有原生技能系统之前的遗留物，对引导没有帮助。

### Worktree 技能重写

`using-git-worktrees` 和 `finishing-a-development-branch` 现在会检测代理是否已经在隔离的 worktree 内运行，并优先使用宿主环境的原生 worktree 控制，然后才回退到 `git worktree`。行为经过了 TDD 验证，并在五个宿主环境中跨平台检查过。（PRI-974，PR #1121）

- **环境检测** —— 两个技能在做任何事之前都会检查 `GIT_DIR != GIT_COMMON`；如果已经在链接的 worktree 中，则完全跳过创建。一个 submodule 守卫可防止误检测。
- **创建 worktree 前征得同意** —— `using-git-worktrees` 不再隐式创建 worktree；技能会先询问用户。修复 #991（subagent-driven-development 曾在未经同意的情况下自动创建 worktree）。
- **原生工具偏好（步骤 1a）** —— 当宿主环境暴露自己的 worktree 工具（例如 Codex）时，技能会优先使用它。用户表达过的偏好会被尊重。
- **基于溯源的清理** —— `finishing-a-development-branch` 只清理 `.worktrees/` 内（由 superpowers 创建）的 worktree；其他任何位置都保持不动。修复 #940（选项 2 曾错误地清理 worktree）、#999（先合并后移除的顺序）和 #238（在 `git worktree remove` 前 `cd` 到仓库根目录）。
- **游离 HEAD 处理** —— 当没有分支可合并时，完成菜单塌缩为两个选项。
- **技能示例中的硬编码 `/Users/jesse` 路径** 被替换为通用占位符。（#858，PR #1122）

### 面向 AI 代理的贡献者指南

`CLAUDE.md`（符号链接到 `AGENTS.md`）顶部有两个新部分，直接对 AI 代理说话。对针对此仓库的最后 100 个已关闭 PR 的审计显示，94% 的拒绝率是由 AI 生成的垃圾驱动的：不读 PR 模板的代理、打开重复项、捏造问题描述，或把 fork 特定或领域特定的改动推到上游。

- **提交前清单** —— 阅读 PR 模板，搜索已有 PR，验证真实问题的存在，确认改动属于核心，并在提交前向人类伙伴展示完整的 diff。
- **我们不会接受什么** —— 第三方依赖、对技能内容做"合规性"改写、项目特定配置、批量 PR、投机性修复、领域特定技能、fork 特定改动、捏造内容，以及捆绑的不相关改动。
- **新宿主环境 PR 需要会话记录** —— 过去大多数新宿主环境集成都是复制技能文件或用 `npx skills` 包装，而不是在会话启动时加载 `using-superpowers` 引导程序。现在要求验收测试（"Let's make a react todo list"必须在干净会话中自动触发 `brainstorming`）和一份完整的会话记录。

### Codex 插件镜像工具

新的 `sync-to-codex-plugin` 脚本把 superpowers 镜像到 OpenAI Codex 插件市场，即 `prime-radiant-inc/openai-codex-plugins`。路径/用户无关，任何团队成员都能运行它。（PR #1165）

- 每次运行都把一个新 fork 克隆到临时目录，内联重新生成覆盖层，并打开一个 PR；从脚本自身的位置自动检测上游，并预检 `rsync`/`git`/`gh auth`/`python3`。
- `--bootstrap` 标志用于首次设置；`EXCLUDES` 模式锚定到源根目录；排除 `assets/`。
- 镜像 `CODE_OF_CONDUCT.md`；丢弃 `agents/openai.yaml` 覆盖层。
- 在镜像的 `plugin.json` 中播种 `interface.defaultPrompt`。（PR #1180 由 @arittr 提供）
- Codex 插件文件提交到源仓库，因此同步脚本使用规范版本；Codex 市场元数据被保留。

### OpenCode

- **引导内容在模块级别缓存** —— `getBootstrapContent()` 在代理的每一步都调用 `fs.existsSync` + `fs.readFileSync` + frontmatter 正则（`experimental.chat.messages.transform` 钩子在 OpenCode 的代理循环中每步触发）。现在只读取一次，在会话生命周期内缓存，并用一个 null 哨兵处理缺失文件的情况。15 个回归测试覆盖缓存行为、fs 调用计数、注入守卫、缺失文件哨兵和缓存重置。（修复 #1202）
- **集成测试已现代化。**
- **README 中澄清了安装注意事项。**

### 代码审查整合

`requesting-code-review` 现在是自包含的：人设、清单和派发模板位于 `skills/requesting-code-review/code-reviewer.md`，技能直接派发 `Task (general-purpose)`。（PR #1299）

- **单一事实来源** —— 之前同时存在于 `agents/code-reviewer.md` 和技能占位模板中（并各自漂移）的人设/清单现在是一个文件。
- **`subagent-driven-development` 如法炮制** —— 它的 `code-quality-reviewer-prompt.md` 现在派发 `Task (general-purpose)` 而不是命名代理。
- **新增行为测试** —— `tests/claude-code/test-requesting-code-review.sh` 在一个微小的项目中植入真实 bug（SQL 注入、明文密码处理、凭据记录），并断言派发的审查者以 Critical/Important 严重程度标记每一个植入的问题，且拒绝批准该 diff。

> 注意：`tests/claude-code/test-requesting-code-review.sh` 和 `tests/claude-code/test-document-review-system.sh`（本文档后面提到）于 2026-05-06 被提升为 drill 场景，并从 `tests/` 移除。参见 `evals/scenarios/code-review-catches-planted-bugs.yaml` 和 `evals/scenarios/spec-reviewer-catches-planted-flaws.yaml`。上面和下面的引用作为本节所描述工作的带日期工件被保留。
- **Codex 和 Copilot 变通文档被裁剪** —— `references/codex-tools.md` 和 `references/copilot-tools.md` 中的"Named agent dispatch"部分记录了如何把命名代理拍平成通用派发。既然没有命名代理发布了，这个变通就不必要了；两个部分都被删除。

### Subagent-Driven Development

- **不再每 3 个任务暂停一次** —— `requesting-code-review` 中的"每批（3 个任务）后审查"节奏（最初用于 `executing-plans`）泄漏到了 `subagent-driven-development`。它被替换为"每个任务或自然的检查点"，外加一条显式的持续执行指令。
- **SDD 集成测试现在真的运行它的断言** —— 三个相互独立的 bug 曾导致该测试在打印任何验证结果之前静默退出：工作目录路径中未解析的 `..` 段、`set -euo pipefail` 与 `find | sort | head -1` 的交互（生产者上的 SIGPIPE 杀掉了脚本），以及 `claude -p` 调用上缺失 `--plugin-dir`，导致测试加载已安装的插件而不是工作树。三个都已修复；六个验证测试现在真的对一次真实的端到端 SDD 运行进行断言。

### Cursor

- **Windows SessionStart 钩子** 改为通过 `run-hook.cmd` 路由，而不是直接调用无扩展名的 `session-start` 脚本。修复了 Windows 在编辑器中打开文件而不是运行它的问题。还从 `hooks-cursor.json` 中移除了一个意外的 UTF-8 BOM。

### Gemini CLI

- **Subagent 派发映射** —— Gemini 的 `Task` 派发现在映射到 `@agent-name` / `@generalist`，并记录了独立任务的并行 subagent 派发。

### 技能

- **跨技能内容的术语清理。**

### 文档与安装

- **README 中新增 Factory Droid 安装说明。**
- **README 中的快速入门安装链接。**（PR #1293 由 @arittr 提供）
- **Codex 插件安装指导已更新。**（PR #1288 由 @arittr 提供）
- **Codex `wait` 映射更正为** 工具参考中的 `wait_agent`。
- **安装顺序重新组织；** Codex 安装说明已清理。
- **移除残余的 `CHANGELOG.md`**，改用 `RELEASE-NOTES.md` 作为唯一来源。（PR #1163 由 @shaanmajid 提供）
- **Discord 邀请链接已修复；** 社区部分新增了发布公告链接和详细的 Discord 描述。

### 社区

- @shaanmajid —— 移除残余的 `CHANGELOG.md`（PR #1163）
- @arittr —— README 快速入门安装链接（#1293）、Codex 插件安装指导（#1288）、`sync-to-codex-plugin` 的 `interface.defaultPrompt` 播种（#1180）

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 支持

- **SessionStart 上下文注入** —— Copilot CLI v1.0.11 在 sessionStart 钩子输出中增加了对 `additionalContext` 的支持。会话启动钩子现在检测 `COPILOT_CLI` 环境变量，并输出 SDK 标准的 `{ "additionalContext": "..." }` 格式，让 Copilot CLI 用户在会话启动时获得完整的 superpowers 引导程序。（最初修复由 @culinablaz 在 PR #910 中提供）
- **工具映射** —— 新增 `references/copilot-tools.md`，包含完整的 Claude Code 到 Copilot CLI 工具等价表
- **技能和 README 更新** —— 把 Copilot CLI 添加到 `using-superpowers` 技能的平台说明和 README 安装部分

### OpenCode 修复

- **技能路径一致性** —— 引导文本不再宣传一个与运行时路径不匹配、会误导人的 `configDir/skills/superpowers/` 路径。代理应使用原生 `skill` 工具，而不是按路径导航到文件。测试现在使用从单一事实来源推导出的一致路径。（#847、#916）
- **引导程序作为用户消息** —— 把引导注入从 `experimental.chat.system.transform` 移到 `experimental.chat.messages.transform`，前置到第一条用户消息而不是添加一条系统消息。避免系统消息每回合重复造成的 token 膨胀（#750），并修复了与 Qwen 及其他在多个系统消息上出问题的模型的兼容性（#894）。

## v5.0.6 (2026-03-24)

### 内联自审取代 subagent 审查循环

subagent 审查循环（派发一个新代理来审查计划/规格）使执行时间翻倍（约 25 分钟开销），却没有可测地改善计划质量。对 5 个版本各 5 次试验的回归测试显示，无论审查循环是否运行，质量分数都相同。

- **brainstorming** —— 用内联的 Spec 自审清单替换了 Spec 审查循环（subagent 派发 + 3 次迭代上限）：占位符扫描、内部一致性、范围检查、歧义检查
- **writing-plans** —— 用内联的自审清单替换了 Plan 审查循环（subagent 派发 + 3 次迭代上限）：规格覆盖、占位符扫描、类型一致性
- **writing-plans** —— 新增显式的"No Placeholders"部分，定义计划的失败情形（TBD、含糊描述、未定义引用、"similar to Task N"）
- 自审在约 30 秒内捕获每次运行的 3-5 个真实 bug，而不是约 25 分钟，缺陷率与 subagent 方法相当

### Brainstorm 服务器

- **会话目录已重组** —— brainstorm 服务器会话目录现在包含两个对等的子目录：`content/`（提供给浏览器的 HTML 文件）和 `state/`（事件、服务器信息、pid、日志）。此前，服务器状态和用户交互数据与所提供的内容存储在一起，使其可以通过 HTTP 访问。`screen_dir` 和 `state_dir` 路径都包含在服务器启动的 JSON 中。（由吉田仁报告）

### 缺陷修复

- **所有者 PID 生命周期修复** —— brainstorm 服务器的所有者 PID 监控有两个 bug，导致 60 秒内误关闭：(1) 跨用户 PID（Tailscale SSH 等）的 EPERM 被当作"进程已死"，(2) 在 WSL 上，祖父 PID 解析为一个短命的子进程，它在第一次生命周期检查之前就退出了。修复方法：把 EPERM 当作"存活"，并在启动时验证所有者 PID——如果它已经死了，就禁用监控，服务器依赖 30 分钟空闲超时。这也移除了 `start-server.sh` 中 Windows/MSYS2 特定的豁免，因为服务器现在以通用方式处理它。（#879）
- **writing-skills** —— 更正了一个错误声明：SKILL.md frontmatter 只支持"两个字段"；现在改为"两个必填字段"，并为所有受支持的字段链接到 agentskills.io 规范（PR #882 由 @arittr 提供）

### Codex App 兼容性

- **codex-tools** —— 新增命名代理派发映射，记录如何把 Claude Code 的命名代理类型翻译成带 worker 角色的 Codex `spawn_agent`（PR #647 由 @arittr 提供）
- **codex-tools** —— 为 worktree 感知技能新增环境检测和 Codex App 完成部分（由 @arittr 提供）
- **设计规格** —— 新增 Codex App 兼容性设计规格（PRI-823），涵盖只读环境检测、worktree 安全技能行为和沙箱回退模式（由 @arittr 提供）

## v5.0.5 (2026-03-17)

### 缺陷修复

- **Brainstorm 服务器 ESM 修复** —— 把 `server.js` 重命名为 `server.cjs`，使 brainstorm 服务器能在 Node.js 22+ 上正确启动，因为根 `package.json` 的 `"type": "module"` 曾导致 `require()` 失败。（PR #784 由 @sarbojitrana 提供，修复 #774、#780、#783）
- **Windows 上的 Brainstorm 所有者 PID** —— 在 Windows/MSYS2 上跳过 PID 生命周期监控，因为那里的 PID 命名空间对 Node.js 不可见，防止服务器 60 秒后自行终止。（#770，文档来自 PR #768 由 @lucasyhzlu-debug 提供）
- **stop-server.sh 可靠性** —— 在报告成功之前验证服务器进程确实已死。SIGTERM + 2 秒等待 + SIGKILL 回退。（#723）

### 变更

- **执行交接** —— 在计划编写后恢复用户在 subagent 驱动与内联执行之间的选择。推荐 subagent 驱动，但不再是强制的。

## v5.0.4 (2026-03-16)

### 审查循环优化

通过消除不必要的审查遍次并收紧审查者焦点，大幅降低 token 使用量并加快规格和计划审查。

- **单次全计划审查** —— 计划审查者现在一次性审查完整计划，而不是逐块审查。移除了所有与块相关的概念（`## Chunk N:` 标题、1000 行块限制、逐块派发）。
- **提高了阻塞问题的门槛** —— 规格和计划审查者提示现在都包含一个"Calibration"部分：只标记会在实现过程中造成真实问题的内容。小问题、风格偏好和格式吹毛求疵不应阻塞批准。
- **减少了最大审查迭代次数** —— 规格和计划审查循环都从 5 次减到 3 次。如果审查者校准正确，3 轮绰绰有余。
- **精简审查者清单** —— 规格审查者从 7 类减到 5 类；计划审查者从 7 类减到 4 类。移除了注重格式的检查（任务语法、块大小），转而注重实质（可构建性、规格一致性）。

### OpenCode

- **一行插件安装** —— OpenCode 插件现在通过 `config` 钩子自动注册技能目录。不需要符号链接或 `skills.paths` 配置。安装只是在 `opencode.json` 中添加一行。（PR #753）
- **新增 `package.json`**，使 OpenCode 可以把 superpowers 作为 npm 包从 git 安装。

### 缺陷修复

- **验证服务器确实已停止** —— `stop-server.sh` 现在在报告成功前确认进程已死。SIGTERM + 2 秒等待 + SIGKILL 回退。如果进程存活则报告失败。（PR #751）
- **通用代理语言** —— brainstorm 伴侣等待页面现在说"the agent"而不是"Claude"。

## v5.0.3 (2026-03-15)

### Cursor 支持

- **Cursor 钩子** —— 新增 `hooks/hooks-cursor.json`，使用 Cursor 的 camelCase 格式（`sessionStart`、`version: 1`），并更新 `.cursor-plugin/plugin.json` 以引用它。修复 `session-start` 中的平台检测，先检查 `CURSOR_PLUGIN_ROOT`（Cursor 也可能设置 `CLAUDE_PLUGIN_ROOT`）。（基于 PR #709）

### 缺陷修复

- **停止在 `--resume` 时触发 SessionStart 钩子** —— 启动钩子会在恢复的会话上重新注入上下文，而这些会话的对话历史中已经有该上下文。钩子现在只在 `startup`、`clear` 和 `compact` 时触发。
- **Bash 5.3+ 钩子挂起** —— 在 `hooks/session-start` 中用 `printf` 替换 heredoc（`cat <<EOF`）。修复 macOS 上 Homebrew bash 5.3+ 中由 bash 在 heredoc 中大型变量展开的回归引起的无限挂起。（#572、#571）
- **POSIX 安全的钩子脚本** —— 在 `hooks/session-start` 中用 `$0` 替换 `${BASH_SOURCE[0]:-$0}`。修复 Ubuntu/Debian 上 `/bin/sh` 为 dash 时的"Bad substitution"错误。（#553）
- **可移植 shebang** —— 在所有 shell 脚本中用 `#!/usr/bin/env bash` 替换 `#!/bin/bash`。修复 NixOS、FreeBSD 以及使用 Homebrew bash 的 macOS 上 `/bin/bash` 过时或缺失时的执行问题。（#700）
- **Windows 上的 Brainstorm 服务器** —— 自动检测 Windows/Git Bash（`OSTYPE=msys*`、`MSYSTEM`）并切换到前台模式，修复由 `nohup`/`disown` 进程回收引起的静默服务器失败。（#737）
- **Codex 文档修复** —— 在 Codex 文档中用 `multi_agent` 替换已弃用的 `collab` 标志。（PR #749）

## v5.0.2 (2026-03-11)

### 零依赖 Brainstorm 服务器

**移除了所有打包的 node_modules —— server.js 现在是完全自包含的**

- 用零依赖的 Node.js 服务器替换 Express/Chokidar/WebSocket 依赖，使用内置的 `http`、`fs` 和 `crypto` 模块
- 移除了约 1,200 行打包的 `node_modules/`、`package.json` 和 `package-lock.json`
- 自定义 WebSocket 协议实现（RFC 6455 分帧、ping/pong、正确的关闭握手）
- 原生 `fs.watch()` 文件监视取代 Chokidar
- 完整测试套件：HTTP 服务、WebSocket 协议、文件监视和集成测试

### Brainstorm 服务器可靠性

- **空闲 30 分钟后自动退出** —— 当没有客户端连接时服务器关闭，防止孤儿进程
- **所有者进程跟踪** —— 服务器监视父宿主环境 PID，并在所属会话死亡时退出
- **存活检查** —— 技能在重用现有实例前验证服务器有响应
- **编码修复** —— 在提供的 HTML 页面上正确设置 `<meta charset="utf-8">`

### Subagent 上下文隔离

- 所有委派技能（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）现在都包含上下文隔离原则
- Subagent 只接收它们需要的上下文，防止上下文窗口污染

## v5.0.1 (2026-03-10)

### Agentskills 合规性

**Brainstorm-server 移入技能目录**

- 按照 [agentskills.io](https://agentskills.io) 规范把 `lib/brainstorm-server/` 移到 `skills/brainstorming/scripts/`
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 引用都被替换为相对的 `scripts/` 路径
- 技能现在完全跨平台可移植——定位脚本不需要平台特定的环境变量
- 移除了 `lib/` 目录（它是最后剩余的内容）

### 新功能

**Gemini CLI 扩展**

- 通过仓库根目录的 `gemini-extension.json` 和 `GEMINI.md` 提供原生 Gemini CLI 扩展支持
- `GEMINI.md` 在会话启动时 `@import` `using-superpowers` 技能和工具映射表
- Gemini CLI 工具映射参考（`skills/using-superpowers/references/gemini-tools.md`）—— 把 Claude Code 工具名（Read、Write、Edit、Bash 等）翻译成 Gemini CLI 对应物（read_file、write_file、replace 等）
- 记录 Gemini CLI 的局限：不支持 subagent，技能回退到 `executing-plans`
- 扩展根位于仓库根目录以实现跨平台兼容（避免 Windows 符号链接问题）
- 安装说明添加到 README

### 改进

**多平台 brainstorm 服务器启动**

- visual-companion.md 中按平台的启动说明：Claude Code（默认模式）、Codex（通过 `CODEX_CI` 自动前台）、Gemini CLI（带 `is_background` 的 `--foreground`），以及其他环境的回退
- 服务器现在把启动 JSON 写入 `$SCREEN_DIR/.server-info`，这样即使 stdout 被后台执行隐藏，代理也能找到 URL 和端口

**Brainstorm 服务器依赖打包**

- `node_modules` 被打包进仓库，使 brainstorm 服务器在全新插件安装时立即可用，无需在运行时要求 `npm`
- 从打包依赖中移除 `fsevents`（仅 macOS 的原生二进制；没有它 chokidar 也能优雅回退）
- 如果 `node_modules` 缺失，通过 `npm install` 回退自动安装

**OpenCode 工具映射修复**

- `TodoWrite` → `todowrite`（之前错误地映射到 `update_plan`）；已对照 OpenCode 源码验证

### 缺陷修复

**Windows/Linux：单引号破坏 SessionStart 钩子**（#577、#529、#644，PR #585）

- hooks.json 中 `${CLAUDE_PLUGIN_ROOT}` 周围使用单引号在 Windows 上失败（cmd.exe 不把单引号识别为路径分隔符），在 Linux 上也失败（单引号阻止变量展开）
- 修复：用转义双引号替换单引号 —— 在 macOS bash、Windows cmd.exe、Windows Git Bash 和 Linux 上都能工作，路径含或不含空格均可
- 已在 Windows 11（NT 10.0.26200.0）上使用 Claude Code 2.1.72 和 Git for Windows 验证

**跳过了 Brainstorming 规格审查循环**（#677）

- 规格审查循环（派发 spec-document-reviewer subagent，迭代直到批准）存在于散文的"After the Design"部分，但缺少于清单和流程图
- 因为代理对照图和清单比对照散文更可靠，规格审查步骤被完全跳过
- 在清单中添加了步骤 7（规格审查循环），并在 dot 图中添加相应节点
- 使用 `claude --plugin-dir` 和 `claude-session-driver` 测试：worker 现在能正确派发审查者

**Cursor 安装命令**（PR #676）

- 修复 README 中的 Cursor 安装命令：`/plugin-add` → `/add-plugin`（经 Cursor 2.5 发布公告确认）

**brainstorming 中的用户审查门**（#565）

- 在规格完成与 writing-plans 交接之间新增显式的用户审查步骤
- 在实现规划开始之前，用户必须批准规格
- 清单、流程图和散文都已用新门更新

**会话启动钩子每个平台只发出一次上下文**

- 钩子现在检测它是在 Claude Code 还是其他平台中运行
- 对 Claude Code 输出 `hookSpecificOutput`，对其他平台输出 `additional_context` —— 防止双重上下文注入

**token 分析脚本中的 lint 修复**

- `tests/claude-code/analyze-token-usage.py` 中 `except:` → `except Exception:`

### 维护

**移除死代码**

- 删除 `lib/skills-core.js` 及其测试（`tests/opencode/test-skills-core.js`）—— 自 2026 年 2 月以来未使用
- 从 `tests/opencode/test-plugin-loading.sh` 中移除 skills-core 存在性检查

### 社区

- @karuturi —— Claude Code 官方市场安装说明（PR #610）
- @mvanhorn —— 会话启动钩子双输出修复、OpenCode 工具映射修复
- @daniel-graham —— 裸 except 的 lint 修复
- PR #585 作者 —— Windows/Linux 钩子引号修复

---

## v5.0.0 (2026-03-09)

### 破坏性变更

**规格和计划目录已重组**

- 规格（brainstorming 输出）现在保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- 计划（writing-plans 输出）现在保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 用户对规格/计划位置的偏好会覆盖这些默认值
- 所有内部技能引用、测试文件和示例路径都已更新以匹配
- 迁移：如有需要，把现有文件从 `docs/plans/` 移到新位置

**subagent 驱动开发在有能力的主机上为强制**

Writing-plans 不再提供 subagent 驱动与 executing-plans 之间的选择。在具有 subagent 支持的宿主环境（Claude Code、Codex）上，subagent-driven-development 是必需的。Executing-plans 保留给没有 subagent 能力的宿主环境，现在它会告诉用户 Superpowers 在支持 subagent 的平台上效果更好。

**Executing-plans 不再批处理**

移除了"执行 3 个任务然后停下来审查"的模式。计划现在持续执行，只在遇到阻塞项时停止。

**斜杠命令已弃用**

`/brainstorm`、`/write-plan` 和 `/execute-plan` 现在显示弃用通知，把用户指向相应的技能。这些命令将在下一个大版本中移除。

### 新功能

**可视化头脑风暴伴侣**

面向 brainstorming 会话的可选浏览器伴侣。当一个主题能从可视化中受益时，brainstorming 技能会主动提出在浏览器窗口中展示线框图、图表、对比和其他内容，与终端对话并列。

- `lib/brainstorm-server/` —— WebSocket 服务器，带浏览器辅助库、会话管理脚本和深/浅主题的框架模板（"Superpowers Brainstorming"，含 GitHub 链接）
- `skills/brainstorming/visual-companion.md` —— 针对服务器工作流、屏幕编写和反馈收集的渐进式披露指南
- Brainstorming 技能在其流程中添加了一个可视化伴侣决策点：在探索项目上下文之后，技能会评估即将到来的问题是否涉及可视化内容，并在它自己的消息中提供伴侣
- 逐问题决策：即使接受后，每个问题都会被评估是浏览器还是终端更合适
- `tests/brainstorm-server/` 中的集成测试

**文档审查系统**

使用 subagent 派发对规格和计划文档进行自动化审查循环：

- `skills/brainstorming/spec-document-reviewer-prompt.md` —— 审查者检查完整性、一致性、架构和 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md` —— 审查者检查规格一致性、任务分解、文件结构和文件大小
- Brainstorming 在编写设计文档后派发规格审查者
- Writing-plans 在每个部分之后包含基于块的计划审查循环
- 审查循环重复进行，直到批准或在 5 次迭代后升级
- `tests/claude-code/test-document-review-system.sh` 中的端到端测试
- `docs/superpowers/` 中的设计规格和实现计划

**贯穿技能管线的架构指导**

为 brainstorming、writing-plans 和 subagent-driven-development 新增了面向隔离的设计和文件大小感知指导：

- **Brainstorming** —— 新部分："Design for isolation and clarity"（清晰的边界、定义良好的接口、可独立测试的单元）和"Working in existing codebases"（遵循现有模式，只做有目标的改进）
- **Writing-plans** —— 新的"File Structure"部分：在定义任务之前先画出文件和职责。新的"Scope Check"兜底：捕获本应在 brainstorming 期间分解的多子系统规格
- **SDD 实现者** —— 新的"Code Organization"部分（遵循计划的文件结构，报告对不断增长文件的担忧）和"When You're in Over Your Head"升级指导
- **SDD 代码质量审查者** —— 现在检查架构、单元分解、计划符合性和文件增长
- **规格/计划审查者** —— 架构和文件大小被添加到审查标准中
- **范围评估** —— Brainstorming 现在会评估一个项目对单个规格来说是否过大。多子系统请求会被及早标记，并分解为子项目，每个都有自己的规格 → 计划 → 实现循环

**subagent 驱动开发改进**

- **模型选择** —— 按任务类型选择模型能力的指导：机械实现用便宜模型，集成用标准模型，架构和审查用高能力模型
- **实现者状态协议** —— Subagent 现在报告 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。控制器适当地处理每种状态：用更多上下文重新派发、升级模型能力、拆分任务或升级给人类

### 改进

**指令优先级层级**

为 using-superpowers 添加了显式的优先级排序：

1. 用户的显式指令（CLAUDE.md、AGENTS.md、直接请求）—— 最高优先级
2. Superpowers 技能 —— 覆盖默认的系统行为
3. 默认系统提示 —— 最低优先级

如果 CLAUDE.md 或 AGENTS.md 说"不要用 TDD"，而某个技能说"始终用 TDD"，用户的指令胜出。

**SUBAGENT-STOP 门**

为 using-superpowers 添加了 `<SUBAGENT-STOP>` 块。为特定任务派发的 Subagent 现在跳过技能检查，而不是激活 1% 规则并调用完整的技能工作流。

**多平台改进**

- Codex 工具映射移到渐进式披露参考文件（`references/codex-tools.md`）
- 添加了 Platform Adaptation 指针，使非 Claude Code 平台可以找到工具等价物
- 计划标题现在称呼"agentic workers"而不是特指"Claude"
- Collab 功能要求在 `docs/README.codex.md` 中记录

**Writing-plans 模板更新**

- 计划步骤现在使用复选框语法（`- [ ] **Step N:**`）进行进度跟踪
- 计划标题同时引用 subagent-driven-development 和 executing-plans，并带平台感知路由

---

## v4.3.1 (2026-02-21)

### 新增

**Cursor 支持**

Superpowers 现在可以与 Cursor 的插件系统协作。包含一个 `.cursor-plugin/plugin.json` 清单和 README 中 Cursor 特定的安装说明。SessionStart 钩子输出现在在现有的 `hookSpecificOutput.additionalContext` 之外增加一个 `additional_context` 字段，用于 Cursor 钩子兼容性。

### 修复

**Windows：恢复多语言包装器以保证可靠的钩子执行（#518、#504、#491、#487、#466、#440）**

Claude Code 在 Windows 上的 `.sh` 自动检测会把 `bash` 前置到钩子命令，破坏执行。修复：

- 把 `session-start.sh` 重命名为 `session-start`（无扩展名），使自动检测不干扰
- 恢复 `run-hook.cmd` 多语言包装器，带多位置 bash 发现（标准 Git for Windows 路径，然后 PATH 回退）
- 找不到 bash 时静默退出而不是报错
- 在 Unix 上，包装器通过 `exec bash` 直接运行脚本
- 使用 POSIX 安全的 `dirname "$0"` 路径解析（在 dash/sh 上可用，而不仅是 bash）

这修复了 Windows 上带空格路径、缺少 WSL、MSYS 上 `set -euo pipefail` 的脆弱性以及反斜杠混乱导致的 SessionStart 失败。

## v4.3.0 (2026-02-12)

这个修复应大幅提高 superpowers 技能的合规性，并应降低 Claude 无意中进入其原生计划模式的机会。

### 变更

**Brainstorming 技能现在强制执行它的工作流，而不是描述它**

模型会跳过设计阶段，直接跳到前端设计等实现技能，或者把整个 brainstorming 过程塌缩成一个文本块。该技能现在使用硬门、强制清单和 graphviz 流程图来强制合规：

- `<HARD-GATE>`：在设计被呈现并由用户批准之前，不使用任何实现技能、代码或脚手架
- 显式清单（6 项），必须创建为任务并按顺序完成
- 以 `writing-plans` 作为唯一有效终态的 Graphviz 流程图
- 对"这个太简单了不需要设计"的反模式提醒 —— 这正是模型用来跳过流程的合理化说法
- 基于部分复杂度而非项目复杂度来设定设计部分的大小

**Using-superpowers 工作流图拦截 EnterPlanMode**

为技能流程图添加了 `EnterPlanMode` 拦截。当模型即将进入 Claude 的原生计划模式时，它检查 brainstorming 是否已经发生，并改走 brainstorming 技能路由。计划模式永远不会进入。

### 修复

**SessionStart 钩子现在同步运行**

把 hooks.json 中的 `async: true` 改为 `async: false`。异步时，钩子可能在模型的第一个回合之前无法完成，这意味着 using-superpowers 指令在第一条消息时不在上下文中。

## v4.2.0 (2026-02-05)

### 破坏性变更

**Codex：用原生技能发现替换引导 CLI**

`superpowers-codex` 引导 CLI、Windows `.cmd` 包装器以及相关的引导内容文件已被移除。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生技能发现，因此旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只是克隆 + 符号链接（记录在 INSTALL.md 中）。不需要 Node.js 依赖。旧的 `~/.codex/skills/` 路径已弃用。

### 修复

**Windows：修复 Claude Code 2.1.x 钩子执行（#331）**

Claude Code 2.1.x 改变了钩子在 Windows 上的执行方式：它现在会自动检测命令中的 `.sh` 文件并前置 `bash`。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 试图把 `.cmd` 文件当作 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 来为 shell 脚本强制 LF 换行（修复 Windows 检出时的 CRLF 问题）。

**Windows：SessionStart 钩子异步运行以防止终端冻结（#404、#413、#414、#419）**

同步的 SessionStart 钩子阻塞了 TUI 在 Windows 上进入原始模式，冻结所有键盘输入。异步运行钩子可以在仍注入 superpowers 上下文的同时防止冻结。

**Windows：修复 O(n^2) 的 `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环在 bash 中由于子串复制开销是 O(n^2)。在 Windows Git Bash 上这需要 60 多秒。替换为 bash 参数替换（`${s//old/new}`），它把每个模式作为单次 C 级遍次运行 —— 在 macOS 上快 7 倍，在 Windows 上快得多。

**Codex：修复 Windows/PowerShell 调用（#285、#243）**

- Windows 不尊重 shebang，因此直接调用无扩展名的 `superpowers-codex` 脚本会触发"打开方式"对话框。所有调用现在都前缀 `node`。
- 修复 Windows 上的 `~/` 路径展开 —— 作为参数传给 `node` 时 PowerShell 不展开 `~`。改为 `$HOME`，它在 bash 和 PowerShell 中都能正确展开。

**Codex：修复安装程序中的路径解析**

使用 `fileURLToPath()` 而不是手动 URL pathname 解析，以便在所有平台上正确处理带空格和特殊字符的路径。

**Codex：修复 writing-skills 中的过时技能路径**

把 `~/.codex/skills/` 引用（已弃用）更新为 `~/.agents/skills/` 以用于原生发现。

### 改进

**worktree 隔离现在在实现前是必需的**

把 `using-git-worktrees` 添加为 `subagent-driven-development` 和 `executing-plans` 的必需技能。实现工作流现在明确要求在开始工作前设置隔离的 worktree，防止意外直接在 main 上工作。

**主分支保护放宽为要求显式同意**

技能现在允许在主分支上工作，但需要显式用户同意，而不是完全禁止。更灵活，同时仍确保用户了解后果。

**简化安装验证**

从验证步骤中移除了 `/help` 命令检查和特定的斜杠命令列表。技能主要通过描述你想做什么来调用，而不是运行特定命令。

**Codex：澄清引导中的 subagent 工具映射**

改进关于 Codex 工具如何映射到 Claude Code 等价物以用于 subagent 工作流的文档。

### 测试

- 为 subagent-driven-development 添加 worktree 要求测试
- 添加主分支红旗警告测试
- 修复技能识别测试断言中的大小写敏感性

---

## v4.1.1 (2026-01-23)

### 修复

**OpenCode：按官方文档标准化为 `plugins/` 目录（#343）**

OpenCode 的官方文档使用 `~/.config/opencode/plugins/`（复数）。我们的文档之前使用 `plugin/`（单数）。虽然 OpenCode 接受两种形式，我们还是标准化为官方约定以避免混淆。

变更：
- 在仓库结构中把 `.opencode/plugin/` 重命名为 `.opencode/plugins/`
- 更新所有平台上的所有安装文档（INSTALL.md、README.opencode.md）
- 更新测试脚本以匹配

**OpenCode：修复符号链接说明（#339、#342）**

- 在 `ln -s` 前添加显式的 `rm`（修复重新安装时"file already exists"错误）
- 添加 INSTALL.md 中缺失的技能符号链接步骤
- 从已弃用的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### 破坏性变更

**OpenCode：切换到原生技能系统**

OpenCode 版 Superpowers 现在使用 OpenCode 的原生 `skill` 工具，而不是自定义的 `use_skill`/`find_skills` 工具。这是更干净的集成，可与 OpenCode 的内置技能发现协作。

**需要迁移：** 技能必须符号链接到 `~/.config/opencode/skills/superpowers/`（见更新的安装文档）。

### 修复

**OpenCode：修复会话启动时的代理重置（#226）**

之前使用 `session.prompt({ noReply: true })` 的引导注入方法导致 OpenCode 在第一条消息时把所选代理重置为"build"。现在使用 `experimental.chat.system.transform` 钩子，它直接修改系统提示，没有副作用。

**OpenCode：修复 Windows 安装（#232）**

- 移除对 `skills-core.js` 的依赖（消除文件被复制而非符号链接时损坏的相对导入）
- 为 cmd.exe、PowerShell 和 Git Bash 添加全面的 Windows 安装文档
- 记录每个平台正确的符号链接与 junction 用法

**Claude Code：修复 Claude Code 2.1.x 的 Windows 钩子执行**

Claude Code 2.1.x 改变了钩子在 Windows 上的执行方式：它现在会自动检测命令中的 `.sh` 文件并前置 `bash`。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 试图把 .cmd 文件当作 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 来为 shell 脚本强制 LF 换行（修复 Windows 检出时的 CRLF 问题）。

---

## v4.0.3 (2025-12-26)

### 改进

**强化 using-superpowers 技能以应对显式技能请求**

解决了一个失败模式：即使用户显式按名称请求某个技能（例如"subagent-driven-development, please"），Claude 也会跳过调用它。Claude 会认为"我知道那是什么意思"，然后直接开始工作，而不是加载技能。

变更：
- 把"The Rule"从"Check for skills"改为"Invoke relevant or requested skills"——强调主动调用而非被动检查
- 添加"BEFORE any response or action"——原始措辞只提到"response"，但 Claude 有时会在不先回复的情况下采取行动
- 添加了调用错误技能也没关系的宽慰话——减少犹豫
- 添加新的红旗："I know what that means" → 知道概念 ≠ 使用技能

**添加显式技能请求测试**

`tests/explicit-skill-requests/` 中的新测试套件验证当用户按名称请求技能时，Claude 是否正确调用它们。包括单轮和多轮测试场景。

## v4.0.2 (2025-12-23)

### 修复

**斜杠命令现在仅限用户使用**

为全部三个斜杠命令（`/brainstorm`、`/execute-plan`、`/write-plan`）添加了 `disable-model-invocation: true`。Claude 不能再通过 Skill 工具调用这些命令——它们被限制为只能手动由用户调用。

底层技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍然可供 Claude 自主调用。这个变更防止了 Claude 调用一个反正只是重定向到某个技能的命令时造成的混淆。

## v4.0.1 (2025-12-23)

### 修复

**澄清如何在 Claude Code 中访问技能**

修复了一个混乱的模式：Claude 通过 Skill 工具调用某个技能，然后又试图单独读取该技能文件。`using-superpowers` 技能现在明确说明 Skill 工具直接加载技能内容——无需读取文件。

- 向 `using-superpowers` 添加"How to Access Skills"部分
- 在指令中把"read the skill"改为"invoke the skill"
- 更新斜杠命令使用完全限定的技能名（例如 `superpowers:brainstorming`）

**为 receiving-code-review 添加 GitHub 线程回复指导**（感谢 @ralphbean）

添加了一条说明：应在原始线程中回复内联审查评论，而不是作为顶层 PR 评论。

**为 writing-skills 添加自动化优先于文档化的指导**（感谢 @EthanJStark）

添加指导：机械性约束应自动化而不是文档化——把技能留给需要判断力的场景。

## v4.0.0 (2025-12-17)

### 新功能

**subagent-driven-development 中的两阶段代码审查**

Subagent 工作流现在在每个任务后使用两个独立的审查阶段：

1. **规格合规审查** - 怀疑的审查者验证实现与规格完全一致。捕获缺失的需求**和**过度构建。不信任实现者的报告——读取实际代码。

2. **代码质量审查** - 只在规格合规通过后运行。审查代码整洁、测试覆盖、可维护性。

这捕获了常见的失败模式：代码写得很好，但不匹配所请求的内容。审查是循环，不是一次性的：如果审查者发现问题，实现者修复，然后审查者再检查。

其他 subagent 工作流改进：
- 控制器向 worker 提供完整的任务文本（而不是文件引用）
- Worker 可以在工作之前**和**期间提出澄清问题
- 报告完成前的自审清单
- 计划在开始时读取一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新提示模板：
- `implementer-prompt.md` - 包含自审清单，鼓励提问
- `spec-reviewer-prompt.md` - 对照需求的怀疑性验证
- `code-quality-reviewer-prompt.md` - 标准代码审查

**调试技术合并工具**

`systematic-debugging` 现在捆绑支持技术和工具：
- `root-cause-tracing.md` - 通过调用栈向后追踪 bug
- `defense-in-depth.md` - 在多层添加验证
- `condition-based-waiting.md` - 用条件轮询替换任意超时
- `find-polluter.sh` - 二分脚本，找出哪个测试产生污染
- `condition-based-waiting-example.ts` - 来自真实调试会话的完整实现

**测试反模式参考**

`test-driven-development` 现在包含 `testing-anti-patterns.md`，涵盖：
- 测试 mock 行为而不是真实行为
- 向生产类添加仅测试方法
- 在不理解依赖的情况下 mock
- 隐藏结构性假设的不完整 mock

**技能测试基础设施**

三个用于验证技能行为的新测试框架：

`tests/skill-triggering/` - 验证技能能从朴素提示触发，而无需显式命名。测试 6 个技能，确保仅凭描述就足够。

`tests/claude-code/` - 使用 `claude -p` 进行无头测试的集成测试。通过会话记录（JSONL）分析验证技能使用。包括用于成本跟踪的 `analyze-token-usage.py`。

`tests/subagent-driven-dev/` - 端到端工作流验证，带两个完整的测试项目：
- `go-fractals/` - 带 Sierpinski/Mandelbrot 的 CLI 工具（10 个任务）
- `svelte-todo/` - 带 localStorage 和 Playwright 的 CRUD 应用（12 个任务）

### 重大变更

**DOT 流程图作为可执行规格**

用 DOT/GraphViz 流程图重写了关键技能，作为权威的流程定义。散文成为支持内容。

**The Description Trap**（记录在 `writing-skills` 中）：发现当技能描述包含工作流摘要时，它们会覆盖流程图内容。Claude 遵循简短描述，而不是阅读详细的流程图。修复：描述必须是仅触发式（"Use when X"），不包含流程细节。

**using-superpowers 中的技能优先级**

当多个技能适用时，流程技能（brainstorming、debugging）现在明确排在实现技能之前。"Build X"先触发 brainstorming，然后是领域技能。

**brainstorming 触发条件加强**

描述改为祈使句："You MUST use this before any creative work—creating features, building components, adding functionality, or modifying behavior."

### 破坏性变更

**技能整合** - 六个独立技能被合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 并入 `systematic-debugging/`
- `testing-skills-with-subagents` → 并入 `writing-skills/`
- `testing-anti-patterns` → 并入 `test-driven-development/`
- 移除 `sharing-skills`（已过时）

### 其他改进

- **render-graphs.js** - 从技能中提取 DOT 图并渲染为 SVG 的工具
- **using-superpowers 中的合理化表格** - 可扫描的格式，含新条目："I need more context first"、"Let me explore first"、"This feels productive"
- **docs/testing.md** - 使用 Claude Code 集成测试测试技能的指南

---

## v3.6.2 (2025-12-03)

### 修复

- **Linux 兼容性**：修复多语言钩子包装器（`run-hook.cmd`）以使用 POSIX 兼容语法
  - 在第 16 行把 bash 特定的 `${BASH_SOURCE[0]:-$0}` 替换为标准 `$0`
  - 解决 Ubuntu/Debian 系统上 `/bin/sh` 为 dash 时的"Bad substitution"错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### 变更

- **OpenCode Bootstrap 重构**：从 `chat.message` 钩子切换到 `session.created` 事件进行引导注入
  - 引导现在在会话创建时通过带 `noReply: true` 的 `session.prompt()` 注入
  - 明确告诉模型 using-superpowers 已经加载，以防止冗余的技能加载
  - 把引导内容生成整合到共享的 `getBootstrapContent()` 辅助函数中
  - 更干净的单实现方法（移除了回退模式）

---

## v3.5.0 (2025-11-23)

### 新增

- **OpenCode 支持**：面向 OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 用于跨上下文压缩保持技能的消息插入模式
  - 通过 chat.message 钩子自动注入上下文
  - 在 session.compacted 事件上自动重新注入
  - 三级技能优先级：项目 > 个人 > superpowers
  - 项目本地技能支持（`.opencode/skills/`）
  - 与 Codex 共享代码的共享核心模块（`lib/skills-core.js`）
  - 带正确隔离的自动化测试套件（`tests/opencode/`）
  - 平台特定文档（`docs/README.opencode.md`、`docs/README.codex.md`）

### 变更

- **重构 Codex 实现**：现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一事实来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进文档**：重写 README 以清晰地解释问题/解决方案
  - 移除重复部分和冲突信息
  - 添加完整的工作流描述（头脑风暴 → 计划 → 执行 → 完成）
  - 简化平台安装说明
  - 强调技能检查协议而非自动激活声明

---

## v3.4.1 (2025-10-31)

### 改进

- 优化 superpowers 引导程序以消除冗余的技能执行。`using-superpowers` 技能内容现在直接提供在会话上下文中，并明确指导只对其他技能使用 Skill 工具。这减少了开销，并防止了代理尽管从会话启动时就有内容，却仍手动执行 `using-superpowers` 的混乱循环。

## v3.4.0 (2025-10-30)

### 改进

- 简化 `brainstorming` 技能，回归最初的对话式愿景。移除了带正式清单的重量级 6 阶段流程，转而采用自然的对话：一次问一个问题，然后用 200-300 词的分节形式呈现设计以供验证。保留文档和实现交接功能。

## v3.3.1 (2025-10-28)

### 改进

- 更新 `brainstorming` 技能，要求提问前进行自主侦察，鼓励以推荐驱动的决策，并防止代理把优先级划分委派回给人类。
- 按照 Strunk《The Elements of Style》的原则（省略多余词汇、把否定形式转为肯定形式、改进平行结构）对 `brainstorming` 技能应用了写作清晰度改进。

### 缺陷修复

- 澄清 `writing-skills` 指导，使其指向正确的代理特定个人技能目录（Claude Code 用 `~/.claude/skills`，Codex 用 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### 新功能

**实验性 Codex 支持**
- 添加统一的 `superpowers-codex` 脚本，带 bootstrap/use-skill/find-skills 命令
- 跨平台 Node.js 实现（可在 Windows、macOS、Linux 上工作）
- 命名空间技能：`superpowers:skill-name` 用于 superpowers 技能，`skill-name` 用于个人技能
- 名称匹配时，个人技能覆盖 superpowers 技能
- 干净的技能展示：显示名称/描述而不显示原始 frontmatter
- 有用的上下文：显示每个技能的支持文件目录
- Codex 的工具映射：TodoWrite→update_plan、subagents→手动回退等
- 与最小 AGENTS.md 集成的引导程序，实现自动启动
- 针对 Codex 的完整安装指南和引导说明

**与 Claude Code 集成的关键差异：**
- 单一统一脚本而不是单独的工具
- 针对 Codex 特定等价物的工具替换系统
- 简化的 subagent 处理（手动工作而不是委派）
- 更新的术语："Superpowers skills" 代替 "Core skills"

### 新增文件
- `.codex/INSTALL.md` - Codex 用户的安装指南
- `.codex/superpowers-bootstrap.md` - 带 Codex 适配的引导说明
- `.codex/superpowers-codex` - 统一 Node.js 可执行文件，包含所有功能

**注意：** Codex 支持是实验性的。该集成提供核心 superpowers 功能，但可能需要根据用户反馈进行改进。

## v3.2.3 (2025-10-23)

### 改进

**更新 using-superpowers 技能以使用 Skill 工具而不是 Read 工具**
- 把技能调用指令从 Read 工具改为 Skill 工具
- 更新描述："using Read tool" → "using Skill tool"
- 更新步骤 3："Use the Read tool" → "Use the Skill tool to read and run"
- 更新合理化列表："Read the current version" → "Run the current version"

Skill 工具是在 Claude Code 中调用技能的正确机制。此更新更正了引导指令，引导代理使用正确的工具。

### 变更文件
- 更新：`skills/using-superpowers/SKILL.md` - 把工具引用从 Read 改为 Skill

## v3.2.2 (2025-10-21)

### 改进

**加强 using-superpowers 技能以对抗代理的合理化**
- 添加 EXTREMELY-IMPORTANT 块，使用关于强制技能检查的绝对措辞
  - "If even 1% chance a skill applies, you MUST read it"
  - "You do not have a choice. You cannot rationalize your way out."
- 添加 MANDATORY FIRST RESPONSE PROTOCOL 清单
  - 代理在任何响应前必须完成的 5 步流程
  - 明确的"不这样做就响应 = 失败"的后果
- 添加 Common Rationalizations 部分，含 8 种特定的逃避模式
  - "This is just a simple question" → WRONG
  - "I can check files quickly" → WRONG
  - "Let me gather information first" → WRONG
  - 外加在代理行为中观察到的 5 种常见模式

这些变更针对观察到的代理行为：尽管有明确的指令，它们仍会在技能使用上合理化。强硬的措辞和预防性的反驳旨在让不服从变得更难。

### 变更文件
- 更新：`skills/using-superpowers/SKILL.md` - 添加三层强制措施以防止跳过技能的合理化

## v3.2.1 (2025-10-20)

### 新功能

**代码审查者代理现在包含在插件中**
- 在插件的 `agents/` 目录中添加 `superpowers:code-reviewer` 代理
- 代理针对计划和编码标准提供系统性代码审查
- 之前要求用户拥有个人代理配置
- 所有技能引用都更新为使用带命名空间的 `superpowers:code-reviewer`
- 修复 #55

### 变更文件
- 新增：`agents/code-reviewer.md` - 带审查清单和输出格式的代理定义
- 更新：`skills/requesting-code-review/SKILL.md` - 引用 `superpowers:code-reviewer`
- 更新：`skills/subagent-driven-development/SKILL.md` - 引用 `superpowers:code-reviewer`

## v3.2.0 (2025-10-18)

### 新功能

**brainstorming 工作流中的设计文档**
- 为 brainstorming 技能添加阶段 4：设计文档
- 设计文档现在在实现前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复原始 brainstorming 命令中在技能转换时丢失的功能
- 文档在 worktree 设置和实现规划之前写入
- 用 subagent 测试验证在时间压力下的合规性

### 破坏性变更

**技能引用命名空间标准化**
- 所有内部技能引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（之前只是 `test-driven-development`）
- 影响所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与使用 Skill 工具调用技能的方式一致
- 更新的文件：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### 改进

**设计文档与实现计划的命名**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实现计划继续使用现有的 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，命名区别清晰

## v3.1.1 (2025-10-17)

### 缺陷修复

- **修复 README 中的命令语法**（#44） - 更新所有命令引用为正确的命名空间语法（`/superpowers:brainstorm` 而不是 `/brainstorm`）。插件提供的命令由 Claude Code 自动添加命名空间，以避免插件之间的冲突。

## v3.1.0 (2025-10-17)

### 破坏性变更

**技能名称标准化为小写**
- 所有技能 frontmatter 的 `name:` 字段现在使用与目录名匹配的小写 kebab-case
- 示例：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有技能公告和交叉引用都更新为小写格式
- 这确保了目录名、frontmatter 和文档之间的命名一致

### 新功能

**增强 brainstorming 技能**
- 添加 Quick Reference 表格，显示阶段、活动和工具使用
- 添加可复制的进度跟踪工作流清单
- 添加决定何时重新访问早期阶段的决策流程图
- 添加带具体示例的全面 AskUserQuestion 工具指导
- 添加"Question Patterns"部分，解释何时使用结构化问题与开放式问题
- 把 Key Principles 重构为可扫描的表格

**Anthropic 最佳实践整合**
- 添加 `skills/writing-skills/anthropic-best-practices.md` - 官方 Anthropic 技能编写指南
- 在 writing-skills SKILL.md 中被引用以提供全面指导
- 提供渐进式披露、工作流和评估的模式

### 改进

**技能交叉引用清晰度**
- 所有技能引用现在使用显式的需求标记：
  - `**REQUIRED BACKGROUND:**` - 你必须理解的前提
  - `**REQUIRED SUB-SKILL:**` - 工作流中必须使用的技能
  - `**Complementary skills:**` - 可选但有帮助的相关技能
- 移除旧路径格式（`skills/collaboration/X` → 仅 `X`）
- 更新 Integration 部分，带分类关系（必需与互补）
- 更新带最佳实践的交叉引用文档

**与 Anthropic 最佳实践对齐**
- 修复描述语法和语气（完全第三人称）
- 添加用于扫描的 Quick Reference 表格
- 添加 Claude 可以复制和跟踪的工作流清单
- 对非显而易见的决策点适当使用流程图
- 改进可扫描的表格格式
- 所有技能都远低于 500 行的建议值

### 缺陷修复

- **重新添加缺失的命令重定向** - 恢复在 v3.0 迁移中意外移除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复 `defense-in-depth` 名称不匹配（原为 `Defense-in-Depth-Validation`）
- 修复 `receiving-code-review` 名称不匹配（原为 `Code-Review-Reception`）
- 修复 `commands/brainstorm.md` 中指向正确技能名的引用
- 移除对不存在的相关技能的引用

### 文档

**writing-skills 改进**
- 用显式需求标记更新交叉引用指导
- 添加对 Anthropic 官方最佳实践的引用
- 改进展示正确技能引用格式的示例

## v3.0.1 (2025-10-16)

### 变更

我们现在使用 Anthropic 的第一方技能系统！

## v2.0.2 (2025-10-12)

### 缺陷修复

- **修复本地技能仓库领先于上游时的错误警告** - 初始化脚本在本地仓库有领先于上游的提交时，错误地警告"New skills available from upstream"。该逻辑现在正确区分三种 git 状态：本地落后（应更新）、本地领先（不警告）和分叉（应警告）。

## v2.0.1 (2025-10-12)

### 缺陷修复

- **修复插件上下文中的会话启动钩子执行**（#8，PR #9） - 钩子曾以"Plugin hook error"静默失败，阻止技能上下文加载。通过以下方式修复：
  - 在 Claude Code 的执行上下文中 BASH_SOURCE 未绑定时，使用 `${BASH_SOURCE[0]:-$0}` 回退
  - 在过滤状态标志时添加 `|| true` 以优雅处理空的 grep 结果

---

# Superpowers v2.0.0 发布说明

## 概览

Superpowers v2.0 通过一次重大的架构转变，让技能更容易获得、更可维护、更由社区驱动。

头条变更是**技能仓库分离**：所有技能、脚本和文档都从插件移入一个专门的仓库（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）。这把 superpowers 从一个单体插件转变为一个轻量垫片，管理技能仓库的本地克隆。技能在会话启动时自动更新。用户通过标准 git 工作流 fork 并贡献改进。技能库独立于插件进行版本控制。

除了基础设施之外，本版本还新增了九个专注于问题解决、研究和架构的技能。我们以祈使语气和更清晰的结构重写了核心的 **using-skills** 文档，使 Claude 更容易理解何时以及如何使用技能。**find-skills** 现在输出可以直接粘贴到 Read 工具中的路径，消除了技能发现工作流中的摩擦。

用户获得无缝体验：插件自动处理克隆、fork 和更新。贡献者发现新架构让改进和分享技能变得轻而易举。本版本为技能作为社区资源快速发展奠定了基础。

## 破坏性变更

### 技能仓库分离

**最大的变更：** 技能不再存在于插件中。它们已移入位于 [obra/superpowers-skills](https://github.com/obra/superpowers-skills) 的独立仓库。

**这对你意味着什么：**

- **首次安装：** 插件自动把技能克隆到 `~/.config/superpowers/skills/`
- **Fork：** 在设置期间，你将被提供 fork 技能仓库的选项（如果安装了 `gh`）
- **更新：** 技能在会话启动时自动更新（尽可能快进）
- **贡献：** 在分支上工作，本地提交，向上游提交 PR
- **不再有遮蔽：** 旧的二级系统（个人/核心）被单仓库分支工作流取代

**迁移：**

如果你有现有安装：
1. 你旧的 `~/.config/superpowers/.git` 将被备份到 `~/.config/superpowers/.git.bak`
2. 旧技能将被备份到 `~/.config/superpowers/skills.bak`
3. 将在 `~/.config/superpowers/skills/` 创建 obra/superpowers-skills 的全新克隆

### 移除的功能

- **个人 superpowers 覆盖系统** - 被 git 分支工作流取代
- **setup-personal-superpowers 钩子** - 被 initialize-skills.sh 取代

## 新功能

### 技能仓库基础设施

**自动克隆与设置**（`lib/initialize-skills.sh`）
- 首次运行时克隆 obra/superpowers-skills
- 如果安装了 GitHub CLI，则提供创建 fork 的选项
- 正确设置 upstream/origin 远程仓库
- 处理旧安装的迁移

**自动更新**
- 每次会话启动时从跟踪远程获取
- 尽可能使用快进自动合并
- 需要手动同步时（分支分叉）发出通知
- 使用 pulling-updates-from-skills-repository 技能进行手动同步

### 新技能

**问题解决技能**（`skills/problem-solving/`）
- **collision-zone-thinking** - 把不相关的概念强行结合在一起，以获得涌现的洞察
- **inversion-exercise** - 翻转假设以揭示隐藏的约束
- **meta-pattern-recognition** - 跨领域发现通用原则
- **scale-game** - 在极端情况下测试以揭示基本真理
- **simplification-cascades** - 找到能消除多个组件的洞察
- **when-stuck** - 派发到正确的问题解决技术

**研究技能**（`skills/research/`）
- **tracing-knowledge-lineages** - 理解想法如何随时间演变

**架构技能**（`skills/architecture/`）
- **preserving-productive-tensions** - 保留多种有效方法，而不是强迫过早解决

### 技能改进

**using-skills（原 getting-started）**
- 从 getting-started 改名为 using-skills
- 用祈使语气完整重写（v4.0.0）
- 前置关键规则
- 为所有工作流添加"Why"解释
- 引用中始终包含 /SKILL.md 后缀
- 更清晰地区分刚性规则和灵活模式

**writing-skills**
- 交叉引用指导从 using-skills 移入
- 添加 token 效率部分（词数目标）
- 改进 CSO（Claude Search Optimization）指导

**sharing-skills**
- 为新分支和 PR 工作流更新（v2.0.0）
- 移除个人/核心分离引用

**pulling-updates-from-skills-repository**（新）
- 与上游同步的完整工作流
- 取代旧的"updating-skills"技能

### 工具改进

**find-skills**
- 现在输出带 /SKILL.md 后缀的完整路径
- 使路径可直接与 Read 工具一起使用
- 更新帮助文本

**skill-run**
- 从 scripts/ 移到 skills/using-skills/
- 改进文档

### 插件基础设施

**会话启动钩子**
- 现在从技能仓库位置加载
- 在会话启动时显示完整技能列表
- 打印技能位置信息
- 显示更新状态（成功更新 / 落后于上游）
- 把"技能落后"警告移到输出末尾

**环境变量**
- `SUPERPOWERS_SKILLS_ROOT` 设置为 `~/.config/superpowers/skills`
- 在所有这些路径中一致使用

## 缺陷修复

- 修复 fork 时重复添加上游远程仓库的问题
- 修复 find-skills 输出中重复的"skills/"前缀
- 从 session-start 中移除过时的 setup-personal-superpowers 调用
- 修复整个钩子和命令中的路径引用

## 文档

### README
- 为新技能仓库架构更新
- 醒目地链接到 superpowers-skills 仓库
- 更新自动更新描述
- 修复技能名称和引用
- 更新元技能列表

### 测试文档
- 添加全面测试清单（`docs/TESTING-CHECKLIST.md`）
- 创建用于测试的本地市场配置
- 记录手动测试场景

## 技术细节

### 文件变更

**新增：**
- `lib/initialize-skills.sh` - 技能仓库初始化和自动更新
- `docs/TESTING-CHECKLIST.md` - 手动测试场景
- `.claude-plugin/marketplace.json` - 本地测试配置

**移除：**
- `skills/` 目录（82 个文件） - 现在位于 obra/superpowers-skills
- `scripts/` 目录 - 现在位于 obra/superpowers-skills/skills/using-skills/
- `hooks/setup-personal-superpowers.sh` - 已过时

**修改：**
- `hooks/session-start.sh` - 使用 ~/.config/superpowers/skills 中的技能
- `commands/brainstorm.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 为新架构完整重写

### 提交历史

本版本包含：
- 20+ 个用于技能仓库分离的提交
- PR #1：Amplifier 启发的问题解决和研究技能
- PR #2：个人 superpowers 覆盖系统（后被替换）
- 多次技能优化和文档改进

## 升级说明

### 全新安装

```bash
# In Claude Code
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件自动处理一切。

### 从 v1.x 升级

1. **备份你的个人技能**（如果有）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件：**
   ```bash
   /plugin update superpowers
   ```

3. **下次会话启动时：**
   - 旧安装将自动备份
   - 将克隆全新的技能仓库
   - 如果你有 GitHub CLI，你将被提供 fork 的选项

4. **迁移个人技能**（如果有）：
   - 在你的本地技能仓库中创建分支
   - 从备份复制你的个人技能
   - 提交并推送到你的 fork
   - 考虑通过 PR 回馈

## 接下来是什么

### 面向用户

- 探索新的问题解决技能
- 尝试基于分支的技能改进工作流
- 把技能回馈给社区

### 面向贡献者

- 技能仓库现在位于 https://github.com/obra/superpowers-skills
- Fork → 分支 → PR 工作流
- 参见 skills/meta/writing-skills/SKILL.md 了解文档的 TDD 方法

## 已知问题

目前没有。

## 致谢

- 问题解决技能受 Amplifier 模式启发
- 社区贡献和反馈
- 对技能有效性的大量测试和迭代

---

**完整变更日志：** https://github.com/obra/superpowers/compare/dd013f6...main
**技能仓库：** https://github.com/obra/superpowers-skills
**Issues：** https://github.com/obra/superpowers/issues
