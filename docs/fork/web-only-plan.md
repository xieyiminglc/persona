# Persona Web-only fork：评估与改造计划

- 日期：2026-11-27
- 评估的上游代码：nwverse/persona \`c5e28e2ee\`（2026-11-27 07:43 +0800）。上游 main 的 HEAD \`e21064f55\` 编译失败，原因见第 2 节。
- 本文件所在分支 \`web-only-eval\` 基于 \`c5e28e2ee\`，只新增本文件，没有改上游代码。
- 目标：只保留 Web 客户端；保留主对话助手、任务派发与督办、待办、Agent 管理；用 herdr 管理多台电脑上的 Agent；以后能持续合并上游。

## 0. 结论

1. **core-only 能编译**。在 \`c5e28e2ee\` 上，\`personad\`、\`persona-hub\`、\`persona-manager\`、\`persona-node\` 和 \`persona-link\` 都能用 \`-tags persona_core_only\` 编出来。上游 main 的 HEAD 现在编不过，默认构建也一样失败，问题不在 core-only。
2. **core-only 能运行**。实测了这条链路：core-only hub + 一次性 Postgres + core-only Web，Web 用链接登录，用 persona-link 配对一个 node，再由 herdr 管理 claude Agent。Web 的机器页和侧栏 Agents 列表里能看到这个 Agent。上游自带的 \`scripts/ci/core-only-smoke.py\` 在同一个 commit 上也是 PASS。
3. **主对话、督办、待办、Agent 管理都属于 Core**（#13430）。core-only 构建会保留它们，Web 也已经有对应页面。依赖手机的流程只有三类，都能在 Web 上替代或直接放弃：推送（改用 Web Push，需要 https）、\`phone_*\` 工具、紧急来电。卡片和审批 Web 本来就能处理。
4. **建议不要物理删除** \`apple/ android/ harmony/ desktop/ native/\`。过去 30 天上游在这些目录有 4,000 多次提交。删掉后每次同步上游都会出现大量 modify/delete 冲突。推荐保留这些目录，只让 fork 自己的 CI 和构建跑 server 与 web；本地可以用 sparse-checkout 不检出它们。第 4 节也给了“一定要删”时的完整清单。
5. **其他引擎靠配置关闭**：每台机器的引擎设置只勾 herdr，不需要改代码。tmux 观察器是常驻的，配置关不掉，但它只读、无害。
6. 自己加的功能优先做成**扩展**（SDK，不碰 Core）。只有必须进模块注册表时才写 in-tree 模块（modapi）。分阶段计划见第 8 节。

## 1. 评估环境与做法

| 项 | 值 |
| --- | --- |
| 上游 commit | \`c5e28e2ee\`（HEAD \`e21064f55\` 编不过） |
| Go | 1.27.1。\`go.mod\` 要求这个版本，\`GOTOOLCHAIN=auto\` 会自动下载 |
| Node / npm | Node 22，\`npm ci\` |
| Postgres | \`postgres:18-alpine\` 一次性容器，映射到 127.0.0.1:25432 |
| hub 端口 | HTTP 127.0.0.1:28660，TLS 127.0.0.1:28662（都不是默认端口） |
| hub 数据目录 | \`PERSONA_HOME=~/.cache/persona-coreonly/home\` |
| node 数据目录 | \`PERSONA_HOME=~/.cache/persona-coreonly/node-home\` |
| HOME / XDG | 都指向 \`~/.cache/persona-coreonly/\` 下的临时目录 |
| herdr | 0.9.5。另起一个独立的 herdr server，\`HERDR_SOCKET_PATH=~/.cache/persona-coreonly/herdr/herdr.sock\` |
| 浏览器 | Chrome headless，独立 profile |

**隔离原则**：同一台机器上已经有一个正在使用的 Persona node（\`~/.persona\`）和 herdr server。评估全程没有读写 \`~/.persona\`，没有占用它的端口和 socket，也没有连接那个 herdr server。用完后，临时的 hub、manager、node、link、herdr server 和 Postgres 容器都已关闭或删除，事后确认原有 node 和 herdr 进程仍在运行。

## 2. core-only 能编吗

| commit | 目标 | 结果 |
| --- | --- | --- |
| \`e21064f55\`（上游 HEAD） | core-only 四个程序 | ❌ \`persona-hub\` 编出来了，其余失败（错误见下） |
| \`e21064f55\` | 默认构建 \`./internal/node ./internal/assistant\` | ❌ 同样的错误，与 core-only 无关 |
| 往回 28 个 first-parent commit，到 \`ece7466fd\` | core-only | ❌ 全部失败 |
| \`c5e28e2ee\` | core-only \`personad persona-hub persona-manager persona-node persona-link\` | ✅ \`go version -m\` 显示 \`-tags=persona_core_only\`、\`vcs.revision=c5e28e2ee…\` |
| \`c5e28e2ee\` | \`python3 scripts/ci/core-only-smoke.py\` | ✅ PASS（输出见下） |

HEAD 的编译错误：

\`\`\`
internal/node/codex_resume.go:97:39: cannot use m (variable of struct type rolloutMeta) as quickMeta value in argument to n.codexEncryptedFailed
internal/node/sessions.go:399:31: cannot use u (variable of struct type agySessionMeta) as string value in argument to n.mergeAgyTranscript
internal/assistant/agent_presets.go:63:14: s.tierModel undefined (type *Service has no field or method tierModel)
internal/assistant/todo_scope.go:124:9: not enough return values
\`\`\`

- 第一个坏的 commit 是 \`ece7466fd\`（07:43:59 +0800），其后又有 \`d24a2b16d\` 等。它们都是一批 “fix: continue #… verification repair” PR，彼此之间存在语义冲突。
- 上游 main 上 \`release-pr\` / \`release-publish\` 两条 workflow 对 \`e21064f55\` 也显示 success，所以 **main 的绿灯不能说明它能编译**。
- 对 fork 的影响：同步上游时必须先确认能编译，见第 7 节的同步门禁。

上游 smoke 的输出（\`c5e28e2ee\`）：

\`\`\`
built personad, persona-hub, persona-manager, persona-node (core-only) in 5s
hub healthy on 127.0.0.1:<随机端口>, modules loaded: none
PASS core-only smoke: built, hub healthy, modules loaded: none, absent: briefing, github, health, imessage, knowledge-ui, lark, mail, office, phone-extra, recording, screen-share, voice, volc (...), node commands absent (iphone, office), stored recordings listed, opened and found, files unchanged without the recording module, clean shutdown (24s)
\`\`\`

## 3. core-only 能跑吗（hub + Postgres + Web + node + herdr）

| 步骤 | 结果 | 证据 |
| --- | --- | --- |
| 起 hub（\`PERSONA_LOCAL_NODE=0\`），hub 会把 manager 一起起来 | ✅ | \`/api/v1/health\` 返回 \`ok:true\`。features 里与模块有关的只剩 \`modules\` 和只在客户端生效的 \`module.glasses\`。manager 日志有 \`modules: loaded none\`。\`/api/v1/managers\` 返回 200 |
| core-only Web 构建 | ✅ 2 分 14 秒 | 用 \`PERSONA_WEB_MODULES=none\`。首屏 en 481,760 B，ja 484,259 B，在 \`FIRST_SCREEN_BUDGET\` 以内 |
| hub 提供 Web | ✅ | 产物放到 \`$PERSONA_HOME/web\` 后，\`GET /\` 返回 200 |
| Web 登录 | ✅ | 打开 \`/?token=<hub token>\`。hub 日志记录了 \`device token … "Web (Chrome · Linux · …)" (web, full)\`，即链接里的 token 已换成这个浏览器自己的设备 token |
| 配对 node | ✅ | 在 hub 侧运行 \`personad hub pair --machine evalnode --add\` 得到 \`persona://link?…\`，在 node 侧运行 \`persona-link pair\`。link 通过 TLS 28662 连上，信任方式为 pin。\`/api/v1/machines\` 显示 \`online:true, transport:link, engine:herdr, engine_hosts:[herdr], herdr_version:0.9.5\` |
| herdr 工作区 | ✅ | 在独立 herdr 里建了工作区 \`eval-demo\`。\`/api/v1/workspaces\` 列出了它，\`source=herdr\` |
| herdr 管理的 Agent | ✅ | 在 \`eval-demo\` 的 tab \`agent2\` 里用 \`herdr agent start --kind claude\` 起了 claude。\`/api/v1/sessions\` 出现一条 \`agent=claude, space_name=eval-demo, slot_name=agent2, state=idle\`。Web 机器页显示 “Agent 1 · eval-demo · Claude Code agent2”，侧栏 Agents 显示 “In a tab 1”，点进去是会话页 |
| 主对话助手 | ⚠️ 页面正常 | 显示 “Model not connected: claude isn't signed in”。原因是隔离 HOME 里的 claude 没登录，和 core-only 无关。本次没有接模型，所以没有实测对话 |
| 待办、定时任务、设置页 | ✅ 页面正常 | To-dos 为空列表，可以新建。Settings 里有 Notifications（Web Push 开关所在的页面） |

### 踩到的问题与临时绕法

1. **上游 HEAD 编不过**：退到最近一个能编的 \`c5e28e2ee\`，没有改上游代码。
2. **开发构建的 hub 不带 Web**：\`server/internal/hub/webdist\` 只有发布脚本会填，所以本地编出的 hub 访问 \`GET /\` 是 404。绕法：把 Web 构建放到 \`$PERSONA_HOME/web\`，hub 会优先用这里（\`docs/web/web.md\` “Running it”）。
3. **\`npm run build:core-only\` 的产物不能直接部署**：它输出到 \`web/node_modules/.cache/persona-core-only\`，注释写明 “never served”。绕法：先 \`npm run pake:build\`（生成密码登录用的 WASM），再 \`PERSONA_WEB_MODULES=none npx vite build --outDir <目录>\`。
4. **同一台机器上起第二个 node，必须隔离 HOME**。node 启动时会往 HOME 写 Agent 集成配置（实测写了 \`<HOME>/.gemini/config/hooks.json\`），还会用 claude 探测模型。如果直接用真实 HOME，会改到正在运行的那个 node 的配置。
5. **隔离 HOME 之后，node 仍然读到了真实用户目录下的 Codex 历史**，出现了 2 条 \`codex_history\` 只读会话。说明有的扫描路径不跟随 \$HOME。推断是按系统用户目录或 \`CODEX_HOME\` 找的，本次没有定位。这些数据只进了一次性 Postgres，容器已经连同数据卷一起删除。
6. **停在首次引导页的 claude 不会被识别为 Agent**。第一次起的 claude 停在主题选择页，Persona 没有列出它。补齐 \`.claude.json\` 的 onboarding 和项目信任标记后重新起一个，进入 REPL 后就出现了。推断 node 依赖 Claude Code 写出的会话文件来识别。
7. **node 日志反复报** \`this personad carries no Antigravity runtime package (built without go generate ./internal/runtimepkg)\`：本地编译没有运行 \`go generate ./internal/runtimepkg\`。这不影响 herdr 和 claude；正式构建要先跑这一步。
8. **node 显示 \`can_deploy=false\`**：手工 \`nohup\` 起的 node 没有被 systemd 或 launchd 托管，hub 无法远程升级它。正式部署用 \`persona-link install\`，或者 \`persona-node node --attach --supervised\`（\`docs/operations/link.md\` “Without systemd”）。
9. **Go 1.27.1**：\`GOTOOLCHAIN=auto\` 会去 GOPROXY 下载这个工具链。离线机器要先装好。

## 4. 删除平台目录：建议与依赖清单

### 4.1 建议：先不删

上游过去 30 天（2026-10-27 起）的提交数，用 \`git rev-list --count --since=2026-10-27 upstream/main -- <dir>\` 统计：

| 目录 | 提交数 |
| --- | --- |
| apple/ | 2425 |
| android/ | 1515 |
| harmony/ | 59 |
| desktop/ | 40 |
| native/ | 97 |
| web/ | 2396 |
| server/ | 5107 |
| scripts/ | 2308 |
| schema/ | 243 |
| spec/ | 378 |
| 全仓库 | 10603 |

删掉后，上游每改一次这些目录，合并时都会产生 modify/delete 冲突。\`merge=ours\` 只在文件两边都存在时才起作用，对已删除的文件无效。另外，删目录必须同时改 \`schema/\`、\`scripts/\` 和 CI 里的 Core 文件（见 4.2），而这些文件上游也经常改，冲突会持续出现。

推荐做法：

- 仓库保留这些目录。fork 的 CI 只启用 server 和 web 相关的 job。
- 本地工作区用 sparse-checkout 不检出这些目录，合并时不受影响：\`git sparse-checkout set --no-cone '/*' '!/apple/' '!/android/' '!/harmony/' '!/desktop/' '!/native/'\`。
- 这样仓库体积不变，但磁盘上没有这些目录，搜索也不会搜到它们。

### 4.2 如果一定要删：要一起处理的依赖

**Web（\`web/\`）**：不 import 任何平台目录，删除后 Web 的构建和测试不受影响。

- \`web/scripts/build-pake.mjs\` 里的 pake 指密码登录用的 PAKE（编译 \`server/cmd/password-wasm\`），与桌面壳无关。
- Web 依赖下面几样，必须保留：
  - \`spec/desktop-bridge.d.ts\`：\`web/src/lib/desktop-types.d.ts\` 引用它，Dockerfile 也会 COPY。
  - \`spec/fixtures/**\`：测试夹具。
  - \`schema/extension/**\` 和 \`schema/configuration/**\`。

**schema/**

- \`schema/wire/*.schema.json\` 的 \`x-out\` 会往 \`apple/\`、\`android/\`、\`harmony/\`（个别文件还有 \`native/\`）生成 Swift、Kotlin、ArkTS、Rust 代码，例如 \`todo-wait.schema.json:8-11\`。删目录时要删掉这些条目，以及只服务于它们的 \`x-guard\`，否则生成器会写到不存在的路径，\`--check\` 会失败。
- \`schema/core-paths.txt\` 有 19 行引用这些目录。\`scripts/test/test_core_paths.py\` 要求每条显式路径都存在，删目录后会因为这些过期条目失败。
- \`schema/module-classification.txt\` 有 55 处引用这些目录，同样要删，否则 \`scripts/test/test_module_classification.py\` 会失败。

**spec/ 与代码生成**

- \`scripts/gen/extension_schema.py:50-51\` 的 \`SWIFT_OUT\` / \`KOTLIN_OUT\` 写进 apple、android，第 86 行同时生成三种语言。要改成只生成 Go 和 TS。
- \`scripts/hooks/pre-commit\` 在 spec 的 token 和图标变动时，会检查 \`android/…/Tokens.kt\` 和 \`apple/Shared/Design\`，这一步要摘掉。
- \`spec/tokens\`、\`spec/icons\` 本身可以保留。

**CI**

- \`.github/workflows/\` 下整个删除：\`android-test.yml\`、\`apple-test.yml\`、\`core-test.yml\`（它构建的是 \`native/app-core\`，不是服务端 Core）、\`desktop-test.yml\`。
- \`ci.yml\`：删 \`android\`、\`apple\`、\`core\`、\`desktop\` 四个 job，同步修改 \`changes\` 的 outputs 和 \`required.needs\`。
- \`release.yml\` 的 desktop 系列 job、\`release-pr.yml\` 里 Android、Apple 和 Windows 桌面包的步骤也要删。
- \`scripts/ci/paths.py\` 的 \`FILTERS\` / \`LANES\` 要改，连带 \`required.py\`、\`check_fast_map.py\`、\`affected_drift.py\` 及它们的测试。
- \`scripts/ci/\` 下的 \`android*.sh/py\`、\`apple*.py/sh\`、\`core.sh\`、\`desktop*.sh\` 整批删除。\`guards.sh\` 和 \`scripts.sh\` 里调用 Apple / Android 检查的那几行也要删。

**server/**

- 没有任何 \`go:embed\` 指向这些目录，编译不受影响。
- 测试会受影响：\`server/internal/proto/resolution_fixtures_test.go:320-323\` 直接读 \`apple/…/*.swift\`，删目录后要删掉这组用例。

**scripts/test、docs、其他**

- \`scripts/test/\` 下数十个 \`test_apple_*\`、\`test_android_*\`、\`test_harmony_*\` 要删。\`test_doc_paths.py\` 的目录表也要改。
- \`docs/apple\`、\`docs/android\`、\`docs/harmony\`、\`docs/desktop\` 可以删，但 \`docs/README.md\`、\`docs/feature-map.md\` 里指向它们的链接要一起清理，否则文档路径检查会失败。
- \`deploy/android/\` 可以删。\`.gitattributes\` 里的相关行可以删。
- \`site/\` 下载页里的平台列表要手工修改（未细读）。

## 5. 依赖手机的流程，以及只有 Web 时怎么办

| 流程 | 实现位置 | Web 现状 | 只有 Web 时的做法 |
| --- | --- | --- | --- |
| \`ask_user\` 问题卡 | \`server/internal/prompts/mcp/assistant/ask_user.md\`（提示词里写的是 “on their iPhone”）；hub 侧是 interaction 加 push | ✅ 已支持。渲染在 \`web/src/views/chat/Interactions.tsx\`，作答在 \`web/src/lib/card-answer.ts\` | 不用改。提示词里的 “iPhone” 只是措辞，改它属于 Core 改动，建议不改 |
| 审批卡（\`mail_send\`、设置修改、任务信任等） | \`server/internal/approval\`、\`server/internal/connector\` | ✅ 已支持。\`approval.go:50\` 的 \`WaitingText\` 写明卡片 “stays pending in the Persona app and web” | 不用改 |
| 权限、计划、终端、MCP 表单卡（LCU elicitation） | \`server/internal/lcuconfirm\` | ✅ 已支持 | 不用改 |
| \`personad ask\` / \`persona-node ask\`（脚本阻塞等用户答复） | \`docs/integrations/script-cli.md\` | ✅ 没有 session 的卡片 Web 能答（#1882）。没有任何可送达设备时 exit 4；加 \`--strict\` 时报 \`no_device\` | Web 页面开着（ws_notify）或开了 Web Push 就算可送达设备 |
| 推送：APNs | \`server/internal/apns\`、\`server/internal/hub/push.go\`（\`Pusher\` 同时管 APNs 和 Web Push） | 与 Web 无关 | 没有手机时自然不会发。不要删 \`push.go\`，它里面也有 Web Push |
| 推送：Web Push | 服务端 \`server/internal/webpush\` 加 \`hub/webpush*.go\`；Web 端 \`web/src/sw.ts\`、\`web/src/lib/web-push.ts\`、\`web/src/views/settings/WebPushSettings.tsx\`（#7459） | ✅ 已实现。卡片、回合完成、提醒、\`notify_user\` 都会推送（\`docs/web/web.md\` “Web Push”） | 限制有四条：要 https（本机地址除外）、不能挂在子路径下、iPhone 必须先添加到主屏幕、hub 要能出站访问各浏览器的推送服务。需要在每个浏览器的 Settings › Notifications 里手动打开。已知缺陷见 #15155 |
| 页面开着时的通知 | \`web/src/lib/hub-notify.ts\`，ws_notify（#8145） | ✅ 已支持 | 不用改 |
| \`phone_*\` 工具（通讯录、日历、健康等） | 由 iOS / Android App 注册 | ❌ Web 不提供 | 放弃。没有手机在线时，模型本来就看不到这些工具 |
| iPhone 控制（\`iphone_*\`） | \`phone-extra\` 模块 | 不在 core-only 里 | 放弃。core-only 已经去掉，smoke 确认 \`iphone\` 命令不存在 |
| 紧急来电（PushKit / CallKit） | \`docs/assistant/urgent-call.md\` | Web 只显示通话记录 | 放弃。默认关闭，没有手机时按 \`no_phone\` 降级成 P0 推送加 Needs you |
| 配对、登录 | \`POST /api/v1/auth/pair-codes\`、\`/api/v1/auth/tokens\` | ✅ 已支持。Web 可以生成登录链接，也支持 \`/?token=\` 登录（本次实测） | 不需要手机扫码 |
| 督办 | \`server/internal/assistant/supervise_*\`、\`server/internal/supervise\` | ✅ 已支持。后台督办快照明确去掉了 iPhone 行（\`assistant/context.go\` 的 \`NoPhone\`） | 不用改。需要用户拍板时出卡片，并进 Needs you 和待办，这些 Web 都能处理 |

## 6. 关掉 herdr 之外的引擎

- 支持的引擎在 \`server/internal/proto/enginehosts.go:17\`：\`herdr\`、\`herdr-core\`（Herdr App）、\`orca\`、\`tern\`、\`rex\`、\`paseo\`、\`cindy\`、\`t3code\`。
- **按机器选择**：存在 manager 数据库的设置 \`machine_engine_hosts\` 里（\`server/internal/devices/engine.go:19-28\`）。
  - 修改方式：Web 机器页的 Engine 设置，或 \`PUT /api/v1/machines/{mid}/engine\`，请求体 \`{"hosts":["herdr"],"default":"herdr"}\`（\`engine.go:208\`）。
  - 节点连上 hub 后，只注册选中的引擎（\`server/internal/node/engine.go:94\` \`engineHostsFor\`）。没选中的引擎不建 client，也不轮询。
  - 本次实测，新配对的机器是 \`engine_hosts:[herdr]\`。
  - 注意：\`SuggestEngineHosts\`（\`enginehosts.go:97\`）会把探测到的引擎全部列为建议。装了 Orca 等的机器，配对后要确认引擎只勾了 herdr。
- **botmux**：在机器页关掉，或设 \`hub.json machines[].botmux: false\`。数据库里的值优先于 hub.json（\`docs/operations/config.md:49,301-306\`）。
- **tmux**：\`server/internal/node/hostbotmux.go:121\` 每次都会注册 tmux 观察器，配置关不掉。它只在 Claude 跑在你自己的 tmux 窗格里时读屏幕、发按键，不列会话。建议保留。
- **硬锁定（可选，要改 Core）**：让 \`NormalizeEngineChoice\`（\`enginehosts.go:42\`）只接受 \`herdr\`。只需改一个函数，冲突面小，但需要每次合并时检查。
- **herdr 本身**：
  - 需要电脑上有 \`herdr\` 命令（PATH 或登录 shell）。\`machines[].herdr_autoinstall\` 可以让节点自动下载 herdr。
  - \`herdr_autostart\` 默认开启。节点会用 \`systemd-run --user --unit=persona-herdr-server\` 起 herdr。同一台机器跑两个 node 时，这个 unit 名会冲突，本次因此自己提前起了 herdr server。
  - 本次 node 的 \`HERDR_SOCKET_PATH\` 指向独立 socket，日志说明它会用这个 socket，而不是自己默认的 \`persona-<id>\` herdr 会话。

## 7. 怎样持续合并上游

### 7.1 分支模型

- \`main\`：只做上游镜像，每次用 \`git merge --ff-only upstream/main\` 更新到一个**通过同步门禁**的 commit。
- \`web\`（fork 的主线）：自己的改动都在这里。从 \`main\` 用 \`merge\` 同步，不 rebase。上游的约定也是 merge，不 force push。
- 同步门禁：一个 fork 自己的脚本，例如 \`scripts/fork/sync.sh\`，检查以下几项都通过才合并：
  - \`go build -tags persona_core_only ./cmd/...\`
  - \`go test -tags persona_core_only ./internal/modules/all/... ./internal/archguard/...\`
  - \`scripts/ci/core-only-smoke.py\`
  - \`cd web && PERSONA_WEB_MODULES=none npm run test:core-only && PERSONA_WEB_MODULES=none npx vite build\`

  不通过就退回到上一个能过的上游 commit。本次 HEAD 编不过，说明这一步必不可少。
- 开启 \`git config rerere.enabled true\`，同样的冲突只需解决一次。

### 7.2 改动放哪里

| 类别 | 放哪里 | 冲突风险 |
| --- | --- | --- |
| 自己的功能（页面、API、助手工具、数据） | **扩展**：\`sdk/\` 加 \`personad extension new|test|pack|install\`，示例 \`sdk/examples/quicknotes\`，文档 \`docs/extensions/core-only-features.md\`（#16244 / #16324）。可以装到不改的 core-only hub 上，有 \`nav_page\`、路由、设置、MCP 工具 | 几乎没有。代码放在仓库外，或 fork 里的独立目录 |
| 需要强类型助手工具、模块开关、客户端模块注册的功能 | **in-tree 模块**：\`server/internal/modules/<fork-id>/\`（modapi），加上 \`server/internal/modules/all/<fork-id>.go\` 新文件 | 中。还要改 \`schema/module-classification.txt\` 和 \`scripts/ci/module-e2e.py\`，这两个是共享文件 |
| Web 界面调整（品牌、布局、隐藏入口） | \`web/src/\`（panel 层，不受 core-change 检查），优先放在新文件或 \`web/src/modules/<fork-id>/\` | 中偏高。web/ 30 天 2,396 次提交。尽量不改 \`web/src/lib/\` 里的 headless core 文件（\`schema/core-paths.txt\` 列出的那些） |
| 构建与部署脚本 | fork 自己的 \`scripts/fork/\`、\`docs/fork/\` | 无 |

**必须碰的 Core 文件**：

- **不删平台目录**：0 个。core-only 是构建参数，引擎和 botmux 都是配置，Web Push 已经内置。
- **删平台目录**：要改 \`schema/core-paths.txt\`、\`schema/module-classification.txt\`、\`schema/wire/*.schema.json\`、\`scripts/gen/extension_schema.py\`、\`scripts/ci/paths.py\` 及相关脚本、\`.github/workflows/*\`、\`server/internal/proto/resolution_fixtures_test.go\`。
- **可选**：
  - 引擎硬锁定：\`server/internal/proto/enginehosts.go\`。
  - 提示词里的 iPhone 措辞：\`server/internal/prompts/**\`。

  这两项都不建议做。

### 7.3 要关注的上游动向

- #13430：Core 的定义正在改成“无 UI 的能力层”，三端界面都只是默认 panel。如果 Web 以后被搬进 panel 或扩展目录，fork 的 Web 改动会随之移动。
- #13438 / #13554：可能把 Agent 直连和机器工作台做成内置扩展，涉及大量文件移动。
- #14916：Core 路径清单和 core-change 检查。只约束向上游提 PR，不约束 fork 内部，但它说明了哪些文件最敏感。
- #16239：上游已经发布 core 版本的安装包（\`install.sh --core\`）。如果不需要改服务端，可以直接用上游的 core 包，只自己构建 Web。

## 8. 分阶段改造计划

| 阶段 | 内容 | 验收标准 | 工作量 |
| --- | --- | --- | --- |
| P0 基线与同步门禁 | 建 \`main\`（镜像）和 \`web\`（主线）两个分支；写 \`scripts/fork/sync.sh\`（门禁见 7.1）；fork CI 只启用 server（core-only）、web（core-only）、scripts 这几类 job | 门禁脚本在一个已知能编的 commit 上通过，在 \`e21064f55\` 这类坏 commit 上拒绝合并；fork CI 全绿 | 0.5–1 天 |
| P1 Web-only 部署 | 一台 hub 机器：core-only 程序（自己构建，或上游 \`--core\` 安装包）加 Postgres 加 https 入口（反向代理或 hub 自带 TLS）；Web 构建放到 \`$PERSONA_HOME/web\`；每台电脑装 core-only 的 \`persona-node\` 和 \`persona-link\`（\`--core --node --pair\`），由 systemd 托管；引擎只勾 herdr | 浏览器能登录；两台以上电脑在线；每台机器的 herdr Agent 出现在 Web；页面关闭后桌面 Chrome 能收到 Web Push；在 Web 上能答 \`ask_user\` 卡和审批卡；node \`can_deploy=true\` | 1–2 天 |
| P2 裁剪平台客户端（可选） | 推荐方案：本地 sparse-checkout，不改仓库。备选方案：一个“删除平台目录”提交，同时完成 4.2 的全部修改，并写一个合并脚本，自动 \`git rm\` 掉已删目录里的冲突 | 推荐方案：工作区里没有这些目录，合并照常。备选方案：\`test_core_paths.py\`、\`test_module_classification.py\`、\`extension_schema.py --check\`、fork CI 全绿，并且连续两次同步上游都靠脚本自动解决冲突 | 推荐方案 0.5 天；备选方案 2–3 天，之后每次同步约 0.5 小时 |
| P3 无手机体验 | 写一份 Web Push 启用说明（https 和每个浏览器的开关）；确认紧急来电是关闭的；可选：在扩展里给助手加一段“用户只用 Web”的提示，而不是改 Core 提示词 | \`notify_user\` 和 \`ask_user\` 在只有 Web 的情况下都能送达，\`no_device\` 计数为 0；助手不再让用户“在 iPhone 上确认” | 0.5–1 天 |
| P4 自己的功能 | 每个功能做成扩展，按 \`scripts/ci/core-sample.py\` 的方式写装上、关掉、卸载的测试 | 能装到未改动的 core-only hub；关掉后入口消失、数据保留 | 按功能估算 |

### 开放问题（推荐选项）

1. **删目录还是 sparse-checkout？** 推荐 sparse-checkout，理由见 4.1。
2. **纯 core-only，还是保留个别模块？** core-only 会去掉 \`lark\`（飞书 / Lark 机器人渠道和 lark-cli connector）、\`mail\`、\`github\` connector 等。
   - 推荐先用 core-only。
   - 如果需要飞书渠道，改用 \`-tags "persona_without_voice persona_without_volc …"\`，逐个去掉不要的模块，只保留 \`lark\`。
   - Web 同样构建出对应的模块集合，不要再用 \`PERSONA_WEB_MODULES=none\`。
3. **hub 用什么方式部署？** 上游的 Docker 镜像是完整版，还没有 core-only 镜像（\`docs/operations/core-install.md\`）。推荐 \`install.sh --hub --core\` 加 systemd，或者自己写一个带 \`persona_core_only\` 的 Dockerfile。
4. **要不要硬锁定引擎？** 推荐不锁，只靠配置。

## 9. 复现步骤

下面的命令使用临时目录和非默认端口，不读写 \`~/.persona\`。\`<…>\` 是要自己填的值。

\`\`\`bash
C=~/.cache/persona-coreonly; R=~/codebase/persona     # R：fork 的检出
WT=~/worktrees/_scratch/persona/build-bisect           # 临时 worktree
git -C $R worktree add --detach $WT c5e28e2ee

# 1) 编译 core-only 程序
mkdir -p $C/{bin,logs,home,node-home,fakehome/.config,tmp,herdr,xdg-run}
(cd $WT/server && GOFLAGS=-trimpath go build -tags persona_core_only -o $C/bin/ \
   ./cmd/personad ./cmd/persona-hub ./cmd/persona-manager ./cmd/persona-node ./cmd/persona-link)
(cd $WT && python3 scripts/ci/core-only-smoke.py)          # 期望最后一行是 PASS

# 2) 构建 core-only Web
(cd $WT/web && npm ci && npm run pake:build && PERSONA_WEB_MODULES=none npx vite build --outDir $C/web-core-only)

# 3) 一次性 Postgres 和 hub
docker run -d --name persona-coreonly-eval-pg -e POSTGRES_USER=persona -e POSTGRES_PASSWORD=<pw> -e POSTGRES_DB=persona \
  -p 127.0.0.1:25432:5432 postgres:18-alpine -c fsync=off -c synchronous_commit=off -c full_page_writes=off
TOKEN=$(head -c 32 /dev/urandom | od -An -tx1 | tr -d ' \n')
cat > $C/home/hub.json <<JSON
{"listen":"127.0.0.1:28660","token":"$TOKEN","public_url":"http://127.0.0.1:28660",
 "database_url":"postgres://persona:<pw>@127.0.0.1:25432/persona?sslmode=disable","machines":[],
 "public":{"tls_listen":"127.0.0.1:28662"}}
JSON
chmod 600 $C/home/hub.json && cp -a $C/web-core-only $C/home/web
HUBENV="PATH=/usr/local/bin:/usr/bin:/bin HOME=$C/fakehome PERSONA_HOME=$C/home TMPDIR=$C/tmp LANG=C.UTF-8"
(cd $C/tmp && env -i $HUBENV PERSONA_LOCAL_NODE=0 $C/bin/persona-hub run >> $C/logs/hub.log 2>&1 &)
curl -s http://127.0.0.1:28660/api/v1/health; grep 'modules: loaded' $C/logs/hub.log

# 4) 独立 herdr server 和 node（HOME、XDG、herdr socket 都隔离）
NODEENV="PATH=$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin HOME=$C/fakehome TMPDIR=$C/tmp LANG=C.UTF-8 TERM=xterm-256color \
 XDG_CONFIG_HOME=$C/fakehome/.config XDG_RUNTIME_DIR=$C/xdg-run HERDR_SOCKET_PATH=$C/herdr/herdr.sock"
(env -i $NODEENV herdr server >> $C/logs/herdr.log 2>&1 &)
LINK=$(env -i $HUBENV $C/bin/personad hub pair --machine evalnode --add --name eval-node | grep '^persona://')
env -i $NODEENV PERSONA_HOME=$C/node-home $C/bin/persona-link pair "$LINK"
(env -i $NODEENV PERSONA_HOME=$C/node-home $C/bin/persona-node node --attach >> $C/logs/node.log 2>&1 &)
(env -i $NODEENV PERSONA_HOME=$C/node-home $C/bin/persona-link run --log $C/logs/link.log &)

# 5) 在独立 herdr 里起一个 claude Agent（先在 $C/fakehome/.claude.json 写 hasCompletedOnboarding 和项目信任标记）
env -i $NODEENV herdr workspace create --cwd $C/tmp/demo-project --label eval-demo --no-focus
env -i $NODEENV herdr tab create --workspace w1 --cwd $C/tmp/demo-project --label agent2 --no-focus   # 记下 pane_id
env -i $NODEENV herdr agent start demo --kind claude --pane <pane_id>
curl -s -H "Authorization: Bearer $TOKEN" http://127.0.0.1:28660/api/v1/sessions     # 应出现 space_name=eval-demo 的 claude

# 6) Web：浏览器打开 http://127.0.0.1:28660/?token=$TOKEN，再进入 Machines › eval-node

# 7) 收尾
pkill -TERM -f "$C/bin/persona-link"; pkill -TERM -f "$C/bin/persona-node"; pkill -TERM -f "$C/bin/persona-hub"
env -i $NODEENV herdr server stop
docker rm -fv persona-coreonly-eval-pg
git -C $R worktree remove $WT
\`\`\`

## 10. 证据、未做项与来源

**本地证据**：在评估机的 \`~/.cache/persona-coreonly/\` 下，没有提交进仓库。

- \`logs/\`：
  - \`build-core-only.log\`：HEAD 的失败记录。
  - \`build-core-only-c5e28e2ee.log\`
  - \`core-only-smoke.log\`
  - \`web-build.log\`
  - \`hub.log\`、\`node.log\`、\`link.log\`、\`herdr.log\`
- \`shots/\`：
  - \`01-login-by-link.png\`
  - \`04-machine-agents.png\`：机器页上的 herdr Agent。
  - \`05-agent-session.png\`
  - \`06-todos.png\`
  - \`07-settings.png\`

截图里有主机信息和本机其他会话的标题，所以不放进这个公开仓库。

**本次没做**：

- Web Push 的真实送达：需要 https 和推送服务可达，本次 hub 只监听 127.0.0.1。
- 接上模型后的主对话：隔离 HOME 里的 claude 没登录。
- 跨机器配对：node 和 hub 在同一台机器上，用的是不同的 \`PERSONA_HOME\`。
- 第 4.2 节删目录方案的实际演练：只做了分析，没有在分支上删除试跑。
- fork 自己的 CI。

**来源**：

- 上游 issue / PR：
  - #13430 Architecture: Persona Core and separate feature modules on every platform…：https://github.com/nwverse/persona/issues/13430
  - #13438 RFC: Modularize built-in core domains into first-class extensions (tracking)：https://github.com/nwverse/persona/issues/13438
  - #16244 Core-only: document and sample for adding a feature without changing Core：https://github.com/nwverse/persona/issues/16244
  - #16324 docs: add core-only extension guide and Quick notes sample：https://github.com/nwverse/persona/pull/16324
  - #16239 Release: core-only install packages for hub, manager and node, with an install check in CI：https://github.com/nwverse/persona/issues/16239
  - #14916 Stricter control of changes to Persona Core: core path list, CODEOWNERS, core-change check：https://github.com/nwverse/persona/issues/14916
  - #7459 Web Push: notify a browser when no Persona page is open：https://github.com/nwverse/persona/issues/7459
  - #15155 Web Push: timeout completion can miss the newer dispatch outcome：https://github.com/nwverse/persona/issues/15155
  - #200 终端引擎支持 tmux（herdr、Orca 之外）：https://github.com/nwverse/persona/issues/200
  - #15396 RFC: Pluggable Web/Desktop UI via a new \`web_shells\` extension point：https://github.com/nwverse/persona/issues/15396
  - #13434 Web: a module host and web/src/modules…：https://github.com/nwverse/persona/issues/13434
- 仓库文档：
  - \`docs/architecture/core-and-modules.md\`
  - \`docs/operations/core-install.md\`
  - \`docs/extensions/core-only-features.md\`
  - \`docs/web/web.md\`（“Running it”、“Sign-in”、“Web Push”）
  - \`docs/integrations/engines.md\`
  - \`docs/integrations/script-cli.md\`
  - \`docs/operations/link.md\`
  - \`docs/operations/config.md\`
  - \`schema/core-paths.txt\`
  - \`scripts/ci/server.sh\`
  - \`scripts/ci/core-only-smoke.py\`
