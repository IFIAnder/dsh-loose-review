# dsh-plugin-loose-review

给 DSH（DeepSeek Harness）的「删除等破坏性命令」加一道**模型审查**闸门，并把它做成一个
**独立的权限预设**「🛡️ 自动审查（宽松）」（key：`loose-review`）——在输入框下方的
权限选择器里选中它才生效，不选就完全不介入，不会偷偷改你的「完全访问」行为。

两道防线：

1. **执行前**：命中破坏性规则模式的命令，先让模型判断「会不会造成严重不可逆破坏」。
   判为破坏就弹审批，判为无害直接放行。
2. **执行中**：命令放行之后仍然盯着工作目录，一旦删掉的文件超过阈值就当场 abort；
   然后弹框问你要不要继续，同意就把命令重跑一遍。

## 为什么需要它

DSH 的 `danger-full-access` 预设把审批策略固定成 `never` —— 完全访问下任何命令都直接
执行，没有任何闸门。本插件补上这一道，但刻意做得比官方
`@deepseek-ai/dsh-experimental-auto-review`（Auto 预设）宽松得多：
只发问一句「会不会毁数据」，不问授权、不问范围、不问生产环境。

> **关于图标**：DSH 的权限选择器只给 4 个内置预设（read-only / workspace-write /
> danger-full-access / auto）硬编码了 SVG 图标，自定义预设按设计「get none」。
> 因此本预设用 emoji `🛡️` 写进 `name` 当作图标——这是唯一能随配置生效、且不会被
> DSH 更新覆盖的做法。

它的判据只有一个，而且刻意很窄：

> 这条命令**执行下去**会不会造成**严重且不可逆**的破坏
> （清空工作区、删除用户数据、格式化磁盘、丢掉未提交的改动……）？

**只看后果，不看动机。** 以下这些明确不在审查范围内，prompt 里也禁止模型评价：

- **这条命令为什么执行、想达成什么目的、这个目的是否合理**；
- 是否获得授权、是否在预期之内、是否礼貌合规；
- 该不该用别的方式做、命令本身写得对不对；
- 凭据、隐私、生产环境、可用性、信任边界。

所以一条"莫名其妙、毫无必要、甚至没人要求"的命令，只要它不破坏任何东西，照样放行。

它比 DSH 自带的 `@deepseek-ai/dsh-experimental-auto-review`（Auto 预设）宽松得多：

| | Auto review（官方实验特性） | 🛡️ 自动审查（宽松）· 本插件 |
|---|---|---|
| 怎么启用 | 权限选择器选 `Auto`（EXP） | 权限选择器选「自动审查（宽松）」 |
| 权限行为 | 完全访问 + 审批询问 | 完全访问 + 审批询问（相同） |
| 审查范围 | 每一个工具调用 | 只有规则命中的「破坏性候选」命令 |
| 判据 | 破坏性 + 授权 + 范围 + 生产环境 + 敏感信息外泄 | 只看后果：会不会造成严重不可逆破坏（不看原因/用途/授权） |
| 默认倾向 | 中等风险需明确授权 | 不确定就放行 |
| 失败处理 | 调用失败 | 默认放行（可配 `deny`） |
| Token 成本 | 每次调用一次模型请求 | 只有命中规则才请求 |

## 行为

0. **只在权限预设为「自动审查（宽松）」（`loose-review`）时介入**。选别的预设（包括「完全访问」）时插件完全不动。
1. 只接管 shell 类工具（默认 `bash` / `win_bash` / `pwsh` / `shell` / `exec` / `terminal` / `cmd`）的 `command` 参数。
2. 命中 `allowPatterns` → 直接放行。
3. 用内置破坏性规则预筛：**不命中 → 直接放行，零 token 零延迟**。
4. 命中 → 用当前会话的 provider/model 做一次 `temperature: 0` 的窄审查。
5. `destructive: false` → 放行。
6. `destructive: true` → 拦截：
   - 审批策略是 `ask`（本预设即是）→ 弹审批，你点一下就能放行一次；
   - 审批策略是 `never` → 拒绝本次调用，并把原因交给模型，
     模型会向你说明影响、请你明确同意，或改用可撤销的做法。

同一会话内相同的命令在 `cacheTtlMs`（默认 5 分钟）内只审一次。

### 理由一定说中文

判成破坏时，摆到你面前的理由固定是简体中文，靠三层保证：

1. **prompt 硬约束**：`reason` 字段被要求"必须用简体中文"，并附 4 条中文示例；
2. **兜底**：万一模型还是回了英文、或者干脆没写理由，插件替换成
   `这条命令会删除或覆盖无法恢复的数据，且无法撤销`；
3. **留痕**：被兜底掉的英文原文只写进 `audit.jsonl` 的 `reasonRewrittenFrom`
   字段，不会出现在审批框或工具错误里。

对应的测试见 `_test/destructive-command-review.test.mjs` 的「中文理由保证」一节。

## 执行期熔断（放行之后还会踩刹车）

审查只是第一道闸门。命令被放行、**真的开始跑之后**，插件还会盯着会话工作目录里的
文件数：一旦净减少超过阈值，立刻把这条命令掐掉。

- 触发时把 `exec.signal` abort：`bash` 工具会 `process.kill()` 掉子进程，
  `win_bash` 会 dispose 持久 shell（下次调用自动重建，shell 状态丢失）。
- 阈值 `deletionThreshold`（默认 **200**），轮询间隔 `guardIntervalMs`（默认 250ms）。
- 用**轮询**而不是 `fs.watch`：Windows 上 fs.watch 会合并批量事件，
  实测一次删 150 个文件只报 1 个事件，数不准。
- 跳过 `node_modules` / `dist` / `build` / `out` / `coverage` / `.git` / 各类缓存
  （`guardIgnore`）——清依赖和构建产物属于日常操作，不该被当成灾难。
- 默认监控**会话工作目录**；用 `guardPaths` 可以换成别的路径（可以给多个）。

```yaml
        guardDeletions: true      # 总开关
        deletionThreshold: 200    # 净减少多少个文件就熔断
        guardIntervalMs: 250      # 轮询间隔（毫秒）
        guardScanBudget: 20000    # 单次统计最多访问多少个目录
        guardIgnore: [node_modules, dist, build, .git, ...]
        guardPaths: []            # 空 = 会话工作目录
```

审计里会出现 `action: "abort"` 和理由「执行期熔断：监控目录内已删除 N 个文件」。

### 熔断之后：弹框问你

熔断只是止损，不是判决。命令被掐掉之后，插件会弹一个审批框问你要不要继续：

- **同意**（`allowed-once`）→ 命令**重跑一遍**，这次不再装熔断器，让它跑完。
  审计里会多一条 `action: "resume"`。
- **拒绝**（`rejected`）→ 保持中止，命令不会重跑。

> 注意：审批策略为 `never` 的预设（如「完全访问」）下，DSH 的审批服务会直接返回
> `rejected` 而**不弹框**——这是服务内部的硬约束，插件绕不过去。想要这个确认框，
> 必须处在审批策略为 `ask` 的预设下，比如「🛡️ 自动审查（宽松）」。

重跑意味着命令**从头再来**：已经删掉的不会回来，剩下的会继续删。

**局限**：命令自己 `cd` 到别的目录再删，监控不到（那个目录不在监控范围内）——
把路径写进 `guardPaths` 即可。另外它防的是「删得多」，不是「删得准」：
删 3 个关键文件不会被拦。

## 安装

把插件目录放到你的 DSH profile 下（目录名随意，下面以 `dsh-plugin-loose-review` 为例）：

```
~/.dsh/profiles/desktop/plugins/dsh-plugin-loose-review/
```

Windows 上是 `%USERPROFILE%\.dsh\profiles\desktop\plugins\dsh-plugin-loose-review\`。

`cordis.patch.yml` 里有两处：

```yaml
# ① 注册「自动审查（宽松）」这个权限预设
- id: permission
  name: "@deepseek-ai/dsh-permission-presets"
  config:
    presets:
      # … read-only / workspace-write / danger-full-access 保持原样 …
      loose-review:
        sandbox: danger-full-access
        approval: ask
        name: "🛡️ 自动审查（宽松）"
        description: 完全访问；删除等破坏性命令先由模型判断会不会造成不可逆破坏，判定破坏时询问你。
    defaultPreset: loose-review   # 可选：把它设成默认预设

# ② 挂载插件，并把它的作用范围限定在这个预设
- insert:
    - id: destructive-command-review
      name: './plugins/dsh-plugin-loose-review/index.js'
      config:
        presets: ['loose-review']
```

改完保存即可 —— DSH 会监听 `cordis.patch.yml` 并热重载，通常 5–10 秒生效；
若长时间没反应，重启 DSH。

> 改 `permission` 这种核心服务时，**不要连续快速保存多次**：多个重载事务叠在一起可能让插件树卡在
> inactive 状态（表现为所有工具都报 `cannot get required service ... in inactive context`）。
> 一次改完、等它重载完再动。真卡住了就重启 DSH，配置本身是合法的。

## 卸载

把 `cordis.patch.yml` 里上面对应的 `- insert:` 两行删掉，重启 DSH，再删目录即可。

## 配置

全部可选，写在 `cordis.patch.yml` 该插件条目的 `config:` 下：

```yaml
- insert:
    - id: destructive-command-review
      name: './plugins/destructive-command-review/index.js'
      config:
        enabled: true
        # 只在哪些权限预设下介入；[] = 所有预设都介入。
        # 默认已设为只在本插件专属的「自动审查（宽松）」下生效。
        presets: ['loose-review']
        # 参与审查的工具名（默认已含 bash/win_bash/pwsh/shell/exec/terminal/cmd）
        shellTools: ['bash', 'win_bash', 'pwsh']
        # 命中即放行的正则（优先于破坏性规则）
        allowPatterns:
          - 'rm\s+-rf\s+node_modules'
          - 'git clean -n'
        # 追加破坏性规则
        extraDangerousPatterns:
          - 'my-danger-cli\s+purge'
        # 审查用模型；留空 = 跟随当前会话
        provider: ''
        model: ''
        timeoutMs: 25000
        # 审查失败时：allow（宽松，默认）| deny（严格）
        failureMode: allow
        # 判定破坏时：auto（ask 弹审批 / never 拒绝）| deny | ask
        onDestructive: auto
        cacheTtlMs: 300000
        audit: true
        maxCommandChars: 12000
        skipWhenAutoPreset: true
```

## 诊断

- 审查记录：`plugins/destructive-command-review/audit.jsonl`（每行一条，含命令、判定、耗时、动作）
- 加载痕迹：`plugins/destructive-command-review/loaded.json`（每次加载/热重载覆盖写）
- 排查监听器是否被调用：临时在 `config:` 里加 `trace: true`，调用记录会写到 `trace.log`
- DSH 日志里搜 `destructive-command-review`

**DSH 会热重载 `cordis.patch.yml`**：改完配置保存后约 5–10 秒生效，不必重启。
（插件代码 `index.js` 的改动同样会随 patch 变更一起重新加载；若改完没反应，手动重启 DSH。）

### 怎么确认它真的在工作

**前提：当前会话的权限预设必须是「自动审查（宽松）」。** 选「完全访问」或别的预设时，插件按设计完全不介入。

切到「自动审查（宽松）」后，跑一条无害的删除命令：

```bash
D=$(mktemp -d /tmp/dcr-check-XXXXXX) && echo hi > "$D/a.txt" && rm -rf "$D"
```

10 秒内 `audit.jsonl` 里应出现一条 `"destructive": false` 的记录（模型判定「删自己刚建的文件」不是破坏，放行）。

再跑一条明显破坏性的（在非 git 目录里执行完全无害）：

```bash
cd /tmp && git reset --hard HEAD~1
```

应当弹出审批框，标题是「破坏性命令审查：…。是否仍然执行？」；拒绝即不执行。

## 与官方 Auto 预设（EXP）的关系

DSH 官方的 **Auto** 预设自带 `@deepseek-ai/dsh-experimental-auto-review`，它会在**每个**工具调用前
用更严格的判据（破坏性 + 授权 + 范围 + 生产环境 + 敏感信息外泄）审查一次。

两者是**并列的两个预设**，各管各的：

- 选「自动审查（宽松）」→ 只有本插件审查；官方 Auto review 因为预设名不是 `auto` 而不介入。
- 选 `Auto` → 只有官方 Auto review 审查；本插件自动让位（`skipWhenAutoPreset: true`）。
- 选「完全访问」→ 两者都不介入，命令直接执行。

如果你想让「完全访问」也保留这道闸门（回到插件最初的行为），把插件的 `presets` 改成
`['loose-review', 'danger-full-access']` 即可 —— 它在 `never` 策略下会改为**直接拒绝**而不是弹审批。

## 已知限制

- 只审 shell 工具的命令参数，**不审** `write` / `edit` 覆盖文件、也不审 MCP 工具。
- 规则预筛是**关键词/正则**，可能漏掉写法刁钻的破坏性命令，也可能把无害命令送审。
- 审查靠模型判断，会有误判；「宽松」意味着**宁可放过也不误拦**，不要把它当强制安全边界。
- 管道里隐藏的破坏（`... | xargs rm`）能命中规则，但 `script.sh` 内部做什么无法从命令行看出。
- 模型会按「命令看起来要做什么」判断，可能把在非目标目录里执行的危险命令也判成破坏性
  （例如在 `/tmp` 里 `git reset --hard`）——这是刻意的保守取舍，拦截只影响这一次调用。
- 插件内部出错时一律放行并记日志，绝不因为插件 bug 让你无法干活。
- `cacheTtlMs` 内相同命令复用判定，所以短时间内改了同一命令的危险参数不会重新审查（换命令文本才会）。
