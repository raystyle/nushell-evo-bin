# nu_plugin_browse

Nushell 的 headless 浏览器插件，基于 [chaser-oxide](https://github.com/nickolay/chaser-oxide)（CDP 协议），需要系统安装 Chrome 或 Chromium。

## 安装

```nu
cargo build -p nu_plugin_browse
plugin add target/debug/nu_plugin_browse.exe
plugin use browse
```

## 命令总览

| 命令 | 说明 |
|------|------|
| `browse <url>` | 临时浏览器，获取页面后自动关闭 |
| `browse open [url]` | 打开/连接持久浏览器 |
| `browse goto <url>` | 已有 session 导航到新 URL |
| `browse status [session]` | 查询 session 状态（不启动浏览器） |
| `browse state` | 列出所有 session 及状态 |
| `browse close [session]` | 关闭 session（`--all` 关闭全部） |

---

## 命令详情

### `browse <url>` — 临时模式

启动临时浏览器，请求完成后自动关闭。始终无头模式，不支持 `--with-head`。

**参数：** `<url>` (必填)，`--no-stealth`，`--wait`，`--hook`，`--eval`/`-e`，`--i-eval`，`--ntrace`

**不支持：** `--with-head`，`--session`

```nu
browse https://example.com
browse https://example.com --i-eval "document.title"
browse https://example.com --hook ./hook.js --ntrace '.*'
```

**返回 record：**

```nu
{
    status:  string,   # "success"
    url:     string,   # 请求的 URL
    content: string,   # 渲染后的 HTML
    message?: {        # 仅在 --hook 或 --i-eval 时存在
        pre?:  {       # hook JS 结果（仅当有错误时存在）
            errors: list<string>,    # hook JS 运行时/语法错误
        },
        post?: {       # eval 结果
            output?: any,    # eval JS 返回值（JSON 解析）
            errors?: list<string>,  # eval JS 错误
        },
    },
    network?: list<record>,    # 仅在 --ntrace 时存在
    idle_reason: {             # 始终存在
        type: string,          # "skipped" | "normal" | "timeout" | "binding"
        reason?: string,       # 仅 binding 类型：JS 传入的 reason
    },
}
```

---

### `browse open [url]` — 持久模式

打开持久浏览器，跨调用复用 cookie/localStorage 等状态。不传 URL 时连接已有 session。

**参数：** `[url]` (可选)，`--session`/`-s`，`--no-stealth`，`--with-head`，`--wait`，`--hook`，`--eval`/`-e`，`--i-eval`，`--ntrace`

```nu
browse open https://example.com                # 打开并导航
browse open https://grok.com --session grok    # 命名 session
browse open                                    # 连接已有 session
browse open --i-eval "document.title"            # 在当前页面执行 JS
browse open --i-eval "document.title" -s grok    # 在命名 session 执行 JS
browse open https://example.com --with-head    # 显示浏览器窗口
```

**返回 record：**

```nu
{
    session: string,   # session 名称
    status:  string,   # "open" | "error"
    url:     string,   # 当前页面 URL
    content: string,   # 渲染后的 HTML（无 URL 时为 ""）
    message?: {        # 仅在 --hook 或 --i-eval 时存在
        pre?:  {
            errors: list<string>,  # hook JS 错误
        },
        post?: {
            output?: any,
            errors?: list<string>,
        },
    },
    network?: list<record>,    # 仅在 --ntrace 时存在
    idle_reason: {             # 始终存在
        type: string,          # "skipped" | "normal" | "timeout" | "binding"
        reason?: string,       # 仅 binding 类型
    },
}
```

| 场景 | `status` | `content` | `message` |
|------|----------|-----------|-----------|
| 打开 + 导航 | `"open"` | HTML | absent |
| 导航 + `--i-eval` 成功 | `"open"` | HTML | `{post: {output: <value>}}` |
| 导航 + `--i-eval` 错误 | `"error"` | HTML | `{post: {errors: [...]}}` |
| 导航 + `--hook` | `"open"` | HTML | `{pre: {errors?: [...]}}`（仅当有错误时） |
| 仅 `--i-eval`（已有页面）| `"open"` | HTML | `{post: {output: <value>}}` |
| 仅 `--i-eval` 错误 | `"error"` | HTML | `{post: {errors: [...]}}` |
| 无 URL、无 eval | `"open"` | `""` | absent |

`browse <url> --open` 等价于 `browse open <url>`。

---

### `browse goto <url>` — 导航

在已打开的 session 上导航到新 URL，不重启浏览器。session 必须已通过 `browse open` 打开。

**参数：** `<url>` (必填)，`--session`/`-s`，`--no-stealth`，`--hook`，`--eval`/`-e`，`--i-eval`，`--wait`，`--ntrace`

```nu
browse goto https://example.com
browse goto https://grok.com/chat --session grok
browse goto https://example.com --i-eval "document.title"
```

**返回 record：** 与 `browse open` 导航场景格式相同。

| 场景 | `status` | `content` | `message` |
|------|----------|-----------|-----------|
| 导航 | `"open"` | HTML | absent |
| 导航 + `--i-eval` 成功 | `"open"` | HTML | `{post: {output: <value>}}` |
| 导航 + `--i-eval` 错误 | `"error"` | HTML | `{post: {errors: [...]}}` |
| session 未打开 | 抛 LabeledError | — | — |

---

### `browse close [session]` — 关闭

关闭持久浏览器并清理 session 文件。profile 目录保留。对 frozen 状态的浏览器会自动查找并强制终止 Chrome 进程。

**参数：** `[session]` (可选)，`--all`

```nu
browse close           # 关闭默认 session
browse close grok      # 关闭命名 session
browse close --all     # 关闭所有 session
```

**返回 table：**

```nu
[
    {session: "default", status: "closed"},
]
```

每行 `{session, status}` 结构相同。`--all` 返回多行，单次关闭返回一行。

关闭已关闭的 session 仍返回 `{session, status: "closed"}`，幂等操作。

`--all` 不能同时指定 session 名（`browse close grok --all` 会报错）。

---

### `browse status [session]` — 状态查询

查询 session 状态，不启动浏览器。

**参数：** `[session]` (可选)

```nu
browse status          # 默认 session
browse status grok     # 命名 session
```

**返回 record：**

```nu
{
    session: string,   # session 名称
    status:  string,   # "open" | "frozen" | "closed"
    url:     string,   # 当前页面 URL（frozen/closed 时为 ""）
    port?:   int,      # CDP 调试端口（frozen/open 时存在）
}
```

| `status` | 含义 | `url` | `port` |
|----------|------|-------|--------|
| `"open"` | 浏览器运行中，CDP 连接正常 | 页面 URL | 端口号 |
| `"frozen"` | session 文件存在但浏览器无响应 | `""` | 端口号 |
| `"closed"` | 无活跃 session | `""` | absent |

---

### `browse state` — 列出所有 session

扫描所有 profile 目录，返回每个 session 的状态。

**参数：** 无

```nu
browse state
```

**返回 table（`list<record>`）：**

每行格式与 `browse status` 相同：

```nu
[
    {session: "default", status: "open", url: "https://...", port: 12345},
    {session: "grok", status: "closed", url: "", port: null},
]
```

---

## 参数说明

### `--session <name>` / `-s <name>`

指定 session 名称，支持多个独立浏览器实例并行。仅持久模式（`browse open`/`goto`）和 `browse --open` 可用。

- 名称仅允许 `[a-zA-Z0-9_-]`，最长 64 字符
- 默认 `"default"`
- 默认 profile: `.nu_browse_profile/`，命名 profile: `.nu_browse_profile_<name>/`

### `--eval <js>` / `-e <js>`

在主世界（main world）中执行 JS。可访问页面全局变量、框架内部状态。支持管道输入。与 `--i-eval` 互斥，`--eval` 优先。

- 返回值自动 JSON 解析：数字→`int`/`float`，字符串→`string`，数组→`list`，对象→`record`，`null`→`null`
- 错误进入 `message.post.errors`，不抛 Nushell 异常

```nu
browse https://example.com --eval "document.title"
"document.title" | browse https://example.com --eval $in
browse https://example.com --eval "window.appState"
```

### `--i-eval <js>`

在隔离世界（isolated world）中执行 JS。不污染页面全局变量，无法访问页面 JS 变量。stealth 安全（不触发 `Runtime.enable`）。

```nu
browse https://example.com --i-eval "document.querySelectorAll('a').length"
```

### `--hook <path_or_js>`

在页面脚本执行前注入 JS（全局 hook）。通过 CDP `addScriptToEvaluateOnNewDocument` 注册，每次页面加载/导航都会执行。脚本原样注入，不包装、不捕获返回值。

支持两种输入方式：
- **文件路径**：传入 `.js` 文件路径，插件读取文件内容注入
- **内联 JS 文本**：直接传入 JS 代码字符串（当路径不存在时自动识别为内联代码）

- `message.pre.errors`：运行时错误和语法错误（通过 `Runtime.exceptionThrown` 捕获）
- 错误不影响主流程（`status` 仍为 `"open"`/`"success"`）
- 如需读取 hook 设置的变量，配合 `--i-eval` 或 `--eval` 使用
- **与 `--ntrace` 组合使用时**：hook JS 中可调用 `window.__ntraceDone({reason: '...'})` 提前结束网络空闲等待（见 [页面等待机制](#页面等待机制)）
- **无 `--ntrace` 时**：`window.__ntraceDone` 仍然可用，可替代固定等待（等待上限为 `--wait` 或 5s）

```nu
# 文件路径：hook.js 内容为 window.__NU_TEST = 'injected';
browse https://example.com --hook ./hook.js --eval "window.__NU_TEST"

# 内联 JS：直接传入代码
browse https://example.com --hook "window.__NU_TEST = 'injected';" --eval "window.__NU_TEST"

# hook + ntrace 提前终止：hook 检测到关键内容后通知插件停止等待
# scanner.js: if (document.querySelector('.data-loaded')) window.__ntraceDone({reason: 'data_ready'});
browse https://example.com --hook ./scanner.js --ntrace '.*'
```

### `--ntrace <pattern>`

网络追踪，记录请求和响应的 headers、状态码、response body。启用后自动等待网络空闲（所有请求完成），最多 30s。

| pattern | 匹配范围 |
|---------|----------|
| `".*"` | 所有请求和响应 |
| `"request"` | 仅请求 |
| `"response"` | 仅响应 |
| `"request:<regex>"` | URL 匹配的请求 |
| `"response:<regex>"` | URL 匹配的响应 |
| `"<regex>"` | URL 匹配的请求和响应 |

**network 列表中每条 record：**

request 类型：

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | `"request"` |
| `method` | string | HTTP 方法 |
| `url` | string | 请求 URL |
| `headers` | record | 请求头 |

response 类型：

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | `"response"` |
| `id` | string | CDP request ID |
| `status` | int | HTTP 状态码 |
| `url` | string | 响应 URL |
| `mime` | string | MIME 类型 |
| `headers` | record | 响应头 |
| `body` | any | 响应体（JSON 自动解析为 record，其余为 string） |

```nu
browse https://example.com --ntrace '.*'
browse https://example.com --ntrace 'response:json'
browse https://example.com --ntrace 'request:api\.example\.com'
```

### `--no-stealth`

默认启用 stealth 模式（修改 `navigator.webdriver` 等属性规避反爬检测）。`--no-stealth` 关闭。

### `--with-head`

显示浏览器窗口，仅持久模式可用。临时模式始终无头。

### `--wait <duration>` / `-w <duration>`

页面加载后额外等待指定时间。实际行为取决于参数组合（见 [页面等待机制](#页面等待机制)）。

```nu
browse https://example.com --wait 2sec
```

---

## 错误处理

| 错误类型 | 处理方式 | 示例 |
|----------|----------|------|
| eval JS 错误 | `message.post.errors` 字段，`status: "error"` | `ReferenceError`、`TypeError` |
| hook JS 错误 | `message.pre.errors` 字段，不影响 `status` | 运行时错误、语法错误 |
| URL 无效 | 抛 LabeledError | `browse baidu.com` |
| 临时模式无 URL | 抛 LabeledError | `browse` |
| session 未打开 | 抛 LabeledError | `browse goto ...` |
| session 名非法 | 抛 LabeledError | `--session "bad.name"` |
| 临时模式使用 `--session` | 抛 LabeledError | `browse url --session x`（需加 `--open`） |
| `browse close` 同时指定 session 和 `--all` | 抛 LabeledError | `browse close grok --all` |
| 浏览器启动/连接失败 | 抛 LabeledError | Chrome 未安装 |

## Session 生命周期

1. `browse open <url>` → 启动浏览器 → 写 `.session` 文件 → `ManuallyDrop` 保持进程
2. `browse goto <url>` → 连接已有 session → 导航
3. `browse close` → 关闭浏览器 → 删除 `.session` 文件 → profile 目录保留
4. `browse status` → 读 `.session` → 尝试 CDP 连接判断 open/frozen/closed
5. frozen 状态 → `browse close` 通过端口查找并 kill Chrome 进程

---

## 页面等待机制

页面导航后，插件会根据参数组合选择不同的等待策略。所有命令的输出都包含 `idle_reason` 字段，表示等待阶段的结束原因。

### 等待策略（按优先级）

| 参数组合 | 等待行为 | `idle_reason.type` | 说明 |
|----------|---------|-------------------|------|
| 仅 `--ntrace` | 网络空闲循环（30s 上限） | `"normal"` / `"timeout"` | 等所有请求完成 + 300ms debounce 确认，或 30s 超时 |
| `--ntrace` + `--hook` | 网络空闲循环 + binding 短路 | `"normal"` / `"timeout"` / `"binding"` | 同上，但 hook JS 可通过 `__ntraceDone` 提前终止 |
| `--hook`（无 `--ntrace`） | binding 等待（`--wait` 或 5s 上限） | `"binding"` / `"skipped"` | 等 hook JS 调用 `__ntraceDone`，超时则 `skipped` |
| `--wait`（无 `--hook`、无 `--ntrace`） | 固定等待 | `"skipped"` | sleep 指定时间后返回 |
| 无任何参数 | 立即返回 | `"skipped"` | `chaser.goto()` 完成后直接取内容 |

> **注意**：`--ntrace` 和 `--hook` 的等待会吞并 `--wait`——即当 `--ntrace` 或 `--hook` 生效时，`--wait` 不作为额外 sleep 叠加，而是作为等待上限的一部分。

### idle_reason 类型

| `type` | 含义 | 触发条件 |
|--------|------|---------|
| `"skipped"` | 无等待 / 等待超时 | 无 `--ntrace` 且无 `--hook`，或 `--hook` 等待超时 |
| `"normal"` | 网络空闲确认 | `--ntrace` 时所有请求完成 + debounce |
| `"timeout"` | 网络空闲超时 | `--ntrace` 时 30s 内未达到 idle |
| `"binding"` | JS 提前终止 | hook JS 调用了 `__ntraceDone`（`reason` 字段为 JS 传入的值） |

```nu
# 无等待 → skipped
browse https://example.com
# → idle_reason: {type: "skipped"}

# 固定等待 → skipped（内容更完整）
browse https://example.com --wait 2sec
# → idle_reason: {type: "skipped"}

# 网络空闲 → normal
browse https://example.com --ntrace '.*'
# → idle_reason: {type: "normal"}

# hook 等待 → binding（hook 主动终止）
# hook.js: setTimeout(() => window.__ntraceDone({reason: 'ready'}), 100);
browse https://example.com --hook ./hook.js
# → idle_reason: {type: "binding", reason: "ready"}

# hook 等待超时 → skipped
# hook.js: （不调用 __ntraceDone）
browse https://example.com --hook ./hook.js
# → idle_reason: {type: "skipped"}（5s 后超时）
```

### Network Idle（`--ntrace`）

启用 `--ntrace` 时，插件注册 CDP 网络事件监听，跟踪请求计数器：

1. `RequestWillBeSent` → 计数 +1（排除 SSE/WebSocket 长连接）
2. `LoadingFinished` / `LoadingFailed` → 计数 -1
3. 计数归零 + 300ms debounce 确认 → `idle_reason: "normal"`
4. 30s 超时 → `idle_reason: "timeout"`
5. hook JS 调用 `__ntraceDone` → `idle_reason: "binding"`

### Binding 提前终止（`--hook`）

使用 `--hook` 时，插件自动注入 `window.__ntraceDone` 函数（无需 `--ntrace`）。Hook JS 可以在检测到关键内容就绪后调用它，跳过剩余等待：

```js
// hook.js — 在 SPA 中等待关键数据加载
const observer = new MutationObserver(() => {
    if (document.querySelector('.product-list[data-loaded="true"]')) {
        window.__ntraceDone({reason: 'products_loaded'});
        observer.disconnect();
    }
});
observer.observe(document.body, {childList: true, subtree: true});
```

- 有 `--ntrace` 时：binding 可以提前终止网络空闲等待
- 无 `--ntrace` 时：binding 可以替代固定等待（上限为 `--wait` 或 5s）

---

## JS 执行世界

Chrome/CDP 中 JS 执行有多个隔离的"世界"（world），插件提供三种执行模式：

| 参数 | 执行世界 | 访问范围 | `Runtime.enable` | Stealth |
|------|---------|---------|-------------------|---------|
| `--i-eval` | Isolated world (`"chaser"`) | DOM + 页面对象（`grantUniversalAccess`） | 不触发 | 安全 |
| `--eval` | Main world（页面默认） | 完全访问页面全局变量、框架状态 | 触发（可被检测） | 不安全 |
| `--hook` | Main world（页面加载前） | 完全访问页面对象 | 不触发 | 安全 |

### `--i-eval` 隔离世界

通过 `Page.createIsolatedWorld` 创建名为 `"chaser"` 的隔离世界，启用 `grantUniversalAccess`。**可访问 DOM 和页面对象**，但不污染页面全局变量，不触发 `Runtime.enable`。

```nu
# 访问 DOM — 安全方式
browse https://example.com --i-eval "document.querySelectorAll('a').length"
```

### `--eval` 主世界

直接在页面的 Main World 中执行，可访问页面 JS 框架的内部状态（React state、Vue data 等），但会触发 `Runtime.enable`，可被反爬系统检测。

```nu
# 访问页面框架内部状态
browse https://example.com --eval "window.__NEXT_DATA__"
```

### `--hook` 注入

在页面脚本执行前注入，运行在 Main World 中。适合修改页面行为、设置全局变量。

```nu
# hook.js: 拦截 fetch 请求
window.__fetchLog = [];
const origFetch = window.fetch;
window.fetch = function(...args) {
    window.__fetchLog.push(args[0]);
    return origFetch.apply(this, args);
};

browse https://example.com --hook ./hook.js --eval "JSON.stringify(window.__fetchLog)"
```

### 注意事项

- `--i-eval` 和 `--eval` 互斥，`--eval` 优先
- `--hook` 与 `--i-eval`/`--eval` 在不同世界中执行，变量不互通（需通过 DOM 中转）
- `--hook` + `--ntrace` 组合时，`window.__ntraceDone` 自动注入到 Main World
