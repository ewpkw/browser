# CHROME.md —— chrome 分支的当前状态与上游同步手册

本文档的唯一用途：**让每一次上游同步又快又不出错**。因此只写两类内容——① 当前状态（分支目的、人设取值、fork 增量、已知限制），② 可重放流程（同步 SOP、验收、环境与编译、从零重建）。一次性迁移记录（某次要重抓哪个依赖、上次冲突长什么样）不写，处理完即丢。

> **fork 基线**：上游 `lightpanda-io/browser` 的 `main` @ `e8aa75939`。
>
> **版本类信息一律从仓库读，不在本文档写死**，否则必然过期：
>
> | 要查的东西 | 唯一来源 |
> |---|---|
> | Zig 版本 | `build.zig.zon` 的 `minimum_zig_version` |
> | V8 版本与 `zig-v8` release tag | `.github/actions/install/action.yml` 里 `v8:` 与 `zig-v8:` 的 `default:` |
> | 预编译 V8 缓存路径 | `Makefile` 的 `V8_CACHE`（由上面两项拼出） |
> | 依赖清单与 URL/hash | `build.zig.zon` 的 `.dependencies` |
> | CLI 选项与取值范围 | `src/Config.zig` 的 `Commands` / `CommonOptions`，帮助文本 `src/help.zon` |

---

## 1. 分支目的与不变量

上游 Lightpanda 有意不冒充其它浏览器：默认 UA 是 `Lightpanda/1.0`、`--user-agent` 含 `Mozilla` 会被拒、CDP `Emulation.setUserAgentOverride` 传 Mozilla UA 被静默忽略、`navigator.*` 默认值跟着编译机与 headless 环境走。本分支的定位是：**对外按真实 Windows Chrome/Edge 的取值呈现**——接受并使用 Mozilla/Chrome/Edge 的 UA 与全部派生信号（client hints、`navigator.*`、语言与 Intl locale），并保证 HTTP 头与 JS 侧读数互相一致。

这条目的优先于"跟随上游"。同步时必须守住的五条不变量：

1. **含 Mozilla 的 UA 一律被接受并生效**（CLI `--user-agent`、CDP `Emulation.setUserAgentOverride`、`Network.setExtraHTTPHeaders` 三条路都不许被挡）。
2. **HTTP 头与 JS 侧同源自洽**：`Sec-Ch-Ua` ↔ `navigator.userAgentData.brands`；`Sec-Ch-Ua-Full-Version-List` ↔ `getHighEntropyValues().fullVersionList`；`Accept-Language` ↔ `navigator.language(s)`；`userAgent` 去前缀 ↔ `appVersion`。任一对不上就是伪造特征。
3. **低熵 brands 用短版本号，高熵 `fullVersionList` 用完整构建号**，两者不得混用。
4. **人设不随编译机变化**：`platform`/`architecture`/`bitness` 等一律写死 Windows x86_64 对应值，不读 `builtin`（ARM 机上编译会露馅）。
5. **CDP 与 CLI 能覆盖 baseline**：默认头一律用最低优先级，不加 `.source = .fixed`。

本文档只描述生产代码（`make build` 编译进二进制的部分）。**本分支不修改任何测试文件、不改任何 `test {}` 块**——上游测试保持原样，见第 10 节。

---

## 2. 维护原则（同步时的决策顺序）

长期跟随上游是硬需求，**每一行 fork 改动都是下一次同步的冲突成本**。按 1→6 顺序决策：

1. **上游已有开关的，绝不自己硬编码**：先问"上游有没有 CLI/配置项能做到同一件事"，有就只改默认值。范例：`navigator.language(s)` / `Accept-Language` / Intl locale 上游由 `--locale` 统一驱动，本分支只改 `Config.HttpHeaders.default_locale` 一行，另外三处覆盖全部删掉。
2. **改前先还原**：冲突文件一律 `git checkout --theirs <file>` 取上游原版，再把第 5 节列的那几行贴回去。**禁止**在 fork 旧结构上手工缝合——那样会整块丢掉上游对该文件的改进。
3. **能删就删**：不再被引用的 fork 常量/函数直接删掉，不为"以后可能用"保留 delta（`Config` 里那组 `sec_ch_ua_*` 死常量就是这么清掉的）。
4. **单行优先，不动结构**：值型改动只改 `return` 那一行；不改函数签名、不改 `pub`/`fn` 可见性（上游把 getter 收成 `fn` 就跟着收）、不重排邻近代码、不删上游未被引用的辅助函数（Zig 对容器级未使用声明不报错）。
5. **保留不可达的上游代码**：例如 `emulation.zig` 里 `if (reserved) { … }` 在本分支永不进入，仍然原样保留。删得越多，下次 diff 越大。
6. **改动必要性判定**：只有**阻碍"模拟正常浏览器"**的才改。不阻碍的一律不改，即使看起来是限制——例如上游 `Makefile` 删掉 `MAKEOVERRIDES`（实测两种 `ZIGFLAGS` 写法都可用）、CLI 层拒绝 `--http-header "Sec-Ch-Ua: …"`（真实浏览器没这个入口，且 CDP 路径已通），这两处都保持上游原样。`Makefile`、`build.zig`、`build.zig.zon` 必须与上游**零 diff**。

核对命令（任何时候）：

```bash
git diff upstream/main..HEAD --stat -- src   # 必须恰好第 4 节那 7 个文件
```

---

## 3. 当前人设取值

| 通道 | 值 |
|---|---|
| `User-Agent` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0` |
| `Sec-Ch-Ua` | `"Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"` |
| `Sec-Ch-Ua-Full-Version-List` | `"Not=A?Brand";v="99", "Microsoft Edge";v="151.0.7813.2", "Chromium";v="151.0.7813.2"` |
| `Sec-Ch-Ua-Platform` / `-Mobile` / `-Arch` / `-Bitness` / `-WoW64` | `"Windows"` / `?0` / `"x86"` / `"64"` / `?0` |
| `Accept-Language` | `zh-CN,zh;q=0.9,en;q=0.8`（由 `--locale` 推导） |
| `Sec-Fetch-Dest` / `-Mode` / `-Site` | **逐请求推导**，取值见 5.3(b) |
| `Sec-Fetch-User` / `Upgrade-Insecure-Requests` / `Priority` | 仅顶层导航：`?1` / `1` / `u=0, i` |
| `navigator.language` / `.languages` | `zh-CN` / `[zh-CN, zh, en]` |
| `navigator.appVersion` | UA 去掉 `Mozilla/` 前缀 |
| `navigator.platform` / `vendor` / `doNotTrack` | `Win32` / `Google Inc.` / `"1"` |
| `navigator.hardwareConcurrency` / `deviceMemory` / `maxTouchPoints` | `32` / `32` / `10` |
| `navigator.userAgentData.brands` | `Not=A?Brand 99` / `Microsoft Edge 151` / `Chromium 151`（短版本号） |
| `getHighEntropyValues()` | `architecture=x86`、`bitness=64`、`platform=Windows`、`platformVersion=15.0.0`、`uaFullVersion=151.0.7813.2`、`model=""`、`wow64=false`、`formFactor=[Desktop]` |
| `navigator.plugins.length` | `5` |
| Intl / `toLocaleString` 默认 locale | `zh-CN`（上游 `Platform.init` 在 `InitializeICU` 前 `setenv("LC_ALL", tag)`） |

- `--user-agent-suffix` 拼在默认 UA 之后；`--user-agent` 整体覆盖。
- 运行时可切换人设而不改代码：`--locale en-US`、CDP `Emulation.setUserAgentOverride({ acceptLanguage })` 会同时刷新 `Accept-Language` 头、`navigator.language(s)`、Intl。
- **若将来升级 Edge 版本**，要同时改四处才自洽：`Config.user_agent_base`（UA 段始终是 `<短>.0.0.0`）、`Config.HttpHeaders.brands`（`.version` 短号 + `.full_version` 完整号）、`Navigator.getAppVersion`（= UA 去前缀）、`NavigatorUAData.uaFullVersion`（= brands 里 Edge/Chromium 的 `.full_version`）。真实取值可查 `https://headers.depar.ch/microsoft-edge`。`Not=A?Brand` 的 `.full_version` 保持 `"99"` 是真实 Chrome 对该 GREASE 品牌的行为，不是漏改。

---

## 4. fork 增量总览（7 个文件，136 增 / 45 删）

| # | 文件 | 增/删 | 改动点 |
|---|------|---|---|
| 1 | `src/Config.zig` | 8 / 8 | 默认 UA、brands、`default_locale`、`validateUserAgent` 行为、CLI 错误提示文案（5 处） |
| 2 | `src/help.zon` | 4 / 5 | `--user-agent` / `--user-agent-suffix` / `--locale` 帮助文本（3 条） |
| 3 | `src/network/HttpClient.zig` | 87 / 3 | `baselineHeaders()` 扩为 `[9]` 且两个 Sec-Ch-Ua 头不标 `.fixed`；新增 `seedFetchMetadata()` + `seedHeaders()` 里一行调用（2 处） |
| 4 | `src/browser/webapi/Navigator.zig` | 7 / 13 | 7 处单行取值 |
| 5 | `src/browser/webapi/NavigatorUAData.zig` | 26 / 14 | `uaPlatform`、3 处调用改 `shortBrandList()`、高熵 4 个值写死、新增 `shortBrandList()`（4 处 + 1 个新函数） |
| 6 | `src/browser/webapi/PluginArray.zig` | 1 / 1 | `length` 0 → 5 |
| 7 | `src/server/cdp/domains/emulation.zig` | 3 / 1 | 删 `error.Reserved` prong（+3 行注释） |

> 「逻辑改动点」不等于 `git diff` 的 hunk 数（相邻改动会被并成一个 hunk，例如 Navigator 7 行取值只构成 4 个 hunk）。核对以第 5 节的逐项列表为准。
> `src/browser/webapi/WorkerNavigator.zig` 曾是第 8 个 fork 文件（被迫跟随 `Navigator.getLanguages` 返回类型）。语言人设改由 `default_locale` 驱动后它已回归上游原样，不再是冲突点。

---

## 5. 逐文件改动详情（同步后照这里重贴）

### 5.1 `src/Config.zig`

**(a) 默认 UA**

```zig
// 上游
const user_agent_base: [:0]const u8 = "Lightpanda/1.0";
// 本分支
const user_agent_base: [:0]const u8 = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
```

**(b) Client-Hint brands**

```zig
// 上游
pub const brands = [_]Brand{
    .{ .brand = "Lightpanda", .version = "1", .full_version = lp.build_config.version },
};
// 本分支
pub const brands = [_]Brand{
    .{ .brand = "Not=A?Brand", .version = "99", .full_version = "99" },
    .{ .brand = "Microsoft Edge", .version = "151", .full_version = "151.0.7813.2" },
    .{ .brand = "Chromium", .version = "151", .full_version = "151.0.7813.2" },
};
```

- 品牌名是 `Not=A?Brand`（等号 + 问号），不是 `Not-A.Brand`；三项顺序不能变（GREASE 品牌在前是 Chrome 固定行为）。
- `full_version` 必须写字面量，不能引用 `lp.build_config.version`——那是 Lightpanda 自己的版本号，出现在 client hint 里等于自报家门。
- `sec_ch_ua` / `sec_ch_ua_full_version_list` 两个 comptime 拼接常量保持上游原样，它们从 `brands` 派生。

**(c) `default_locale`：语言人设的唯一开关（直接用上游 `--locale` 机制）**

```zig
// 上游
const default_locale: [:0]const u8 = "en-US";
// 本分支
// This branch defaults to zh-CN: it drives Accept-Language, navigator.language
// and navigator.languages together, exactly like a real Chrome would.
const default_locale: [:0]const u8 = "zh-CN";
```

一个标签经 `HttpHeaders.acceptLanguageFor()` 推导后同时驱动四处：`Accept-Language` 头、`navigator.languages`、`navigator.language`、Intl 默认 locale。头与 JS 同源 → 交叉比对不会露馅；且运行时可切换人设（见第 3 节）。

**(d) `validateUserAgent`：删掉 Mozilla 拒绝逻辑**

```zig
// 上游（多一个 if 块）
    if (std.ascii.indexOfIgnoreCase(ua, "mozilla") != null) {
        return error.Reserved;
    }
// 本分支：只保留 for 循环里的非打印字符检测
pub fn validateUserAgent(ua: []const u8) !void {
    for (ua) |c| {
        if (!std.ascii.isPrint(c)) {
            return error.NonPrintable;
        }
    }
}
```

三个生产调用点，同步时都要看：

| 调用点 | 写法 | 影响 |
|---|---|---|
| `Transfer.verifyHeader`（`HttpClient.zig`） | `catch \|err\| { log.warn; 丢弃 }` | 错误集收窄不影响编译，Mozilla UA 通过 |
| `Network.setExtraHTTPHeaders`（`domains/network.zig`） | 同上 | 同上 |
| `Emulation.setUserAgentOverride`（`domains/emulation.zig`） | **对错误集做穷尽 `switch`** | 必须同步删 `error.Reserved` prong，否则编译失败（见 5.7） |

**(e) `userAgentValidator` 的提示文案**

```zig
// 上游
log.fatal(.app, "invalid user-agent", .{ .err = err, .hint = "must be printable ASCII and can't contain Mozilla" });
// 本分支
log.fatal(.app, "invalid user-agent", .{ .err = err, .hint = "must be printable ASCII" });
```

### 5.2 `src/help.zon`

```
--user-agent <STRING>
  上游:   … Must not impersonate other browsers; any value containing
          "Mozilla" is forbidden. The browser still sends Sec-Ch-Ua. …
  本分支: Override the User-Agent header entirely. The browser still sends
          Sec-Ch-Ua. Incompatible with --user-agent-suffix.

--user-agent-suffix <STRING>
  上游:   Suffix appended to the Lightpanda/X.Y User-Agent.
  本分支: Suffix appended to the default User-Agent.

--locale <TAG>
  上游:   … Defaults to en-US.
  本分支: … Defaults to zh-CN.
```

`--locale` 那句必须跟 5.1(c) 一起改：上游把它写成字面量（没用 `{1s}` 占位符机制）。

### 5.3 `src/network/HttpClient.zig`

**(a) `baselineHeaders()`**

```zig
// 上游
// Headers _all_ requests include.
pub fn baselineHeaders(self: *const Client) [4]Transfer.RequestHeader {
    return .{
        .{ .name = "User-Agent", .value = self.getUserAgent() },
        .{ .name = "Sec-Ch-Ua", .value = lp.Config.HttpHeaders.sec_ch_ua, .source = .fixed },
        .{ .name = "Sec-Ch-Ua-Full-Version-List", .value = lp.Config.HttpHeaders.sec_ch_ua_full_version_list, .source = .fixed },
        // Omitting Accept-Language triggers bot-protection on some CDNs
        // (Akamai) when Accept-Encoding is present.
        .{ .name = "Accept-Language", .value = self.getAcceptLanguage() },
    };
}

// 本分支（[4] → [9]，两个 Sec-Ch-Ua 头去掉 .source = .fixed）
// Headers _all_ requests include.
// Sec-Ch-Ua / Sec-Ch-Ua-Full-Version-List are intentionally *not* marked
// `.source = .fixed` (unlike upstream): this branch exists to behave like a
// real Chrome/Edge, so a CDP client driving Network.setExtraHTTPHeaders (or
// Emulation.setUserAgentOverride with `headers`) must be able to move these
// client hints in lockstep with the UA. Marking them fixed only made every
// such request emit an "ignore overriding fixed header" warn and silently
// dropped the client's value.
pub fn baselineHeaders(self: *const Client) [9]Transfer.RequestHeader {
    return .{
        .{ .name = "User-Agent", .value = self.getUserAgent() },
        .{ .name = "Sec-Ch-Ua", .value = lp.Config.HttpHeaders.sec_ch_ua },
        .{ .name = "Sec-Ch-Ua-Full-Version-List", .value = lp.Config.HttpHeaders.sec_ch_ua_full_version_list },
        // Omitting Accept-Language triggers bot-protection on some CDNs
        // (Akamai) when Accept-Encoding is present.
        .{ .name = "Accept-Language", .value = self.getAcceptLanguage() },
        // Client Hints for Chrome fingerprint
        .{ .name = "Sec-Ch-Ua-Platform", .value = "\"Windows\"" },
        .{ .name = "Sec-Ch-Ua-Mobile", .value = "?0" },
        .{ .name = "Sec-Ch-Ua-Arch", .value = "\"x86\"" },
        .{ .name = "Sec-Ch-Ua-Bitness", .value = "\"64\"" },
        .{ .name = "Sec-Ch-Ua-WoW64", .value = "?0" },
    };
}
```

- `.fixed` 会让 CDP 下发的同名头被静默丢弃并刷 `ignore overriding fixed header` warn。`HeaderSource` 默认 `.user_agent`（最低优先级），任何上层来源都能覆盖。
- **上游至今保留 `.fixed`，每次同步都要检查这两行有没有被加回来。**
- `Accept-Language` 必须用 `self.getAcceptLanguage()`，不要用已废弃的 `HttpHeaders.accept_language` 常量，否则 `--locale` 与 CDP `acceptLanguage` 不作用于真实请求头。
- 5 个 Client-Hint 的值只写在这里，`Config.zig` 里不再有对应常量。

**(b) 新增 `seedFetchMetadata()`：逐请求生成 Sec-Fetch 家族**

不能塞进 `baselineHeaders()`：这一组头互相关联，对所有请求写死同一组值会让一个 `<script>` 自称 `sec-fetch-dest: document`，比干脆不发更显眼。做法是在 `Transfer.seedHeaders()` 里插一行，旁边加一个新函数：

```zig
// seedHeaders() 里只插这一行：紧跟 baselineHeaders 循环之后、--http-header 循环之前
try self.seedFetchMetadata();
```

函数全文（同步后原样贴回即可，不碰任何上游函数体）：

```zig
fn seedFetchMetadata(self: *Transfer) !void {
    const req = &self.req;

    const maybe_dest: ?[]const u8 = switch (req.resource_type) {
        .document => "document",
        .stylesheet => "style",
        .script => "script",
        .image => "image",
        .xhr, .fetch, .eventsource => "empty",
        .worker => null, // destination 细分太细，不猜：worker 请求不发 fetch 元数据
    };
    const dest = maybe_dest orelse return;

    // Fetch 标准的四种 mode 与 Chrome 的 sec-fetch-mode 值一字不差
    const mode: []const u8 = switch (req.request_mode) {
        .navigate => "navigate",
        .cors => "cors",
        .no_cors => "no-cors",
        .same_origin => "same-origin",
    };

    // 导航一律 none：Frame 在 newRequest 之前就把自身 origin 设成了目标 URL，
    // 这一层拿不到发起文档；据此算 same-origin 等于让会话第一个请求谎称"由同源
    // 页面打开"，那本身就是 headless 特征。子资源的 req.origin 才是发起文档。
    const site: []const u8 = if (req.resource_type == .document) "none" else blk: {
        const origin = req.origin orelse break :blk "none";
        if (URL.isSameOrigin(req.url, origin)) {
            break :blk "same-origin";
        }
        if (Cookie.areHostsSameSite(URL.getHostname(req.url), URL.getHostname(origin))) {
            break :blk "same-site";
        }
        break :blk "cross-site";
    };

    try self.addHeader("Sec-Fetch-Dest", dest, .{});
    try self.addHeader("Sec-Fetch-Mode", mode, .{});
    try self.addHeader("Sec-Fetch-Site", site, .{});

    // 只有顶层导航带这三个：?1 表示由用户触发（驱动器替用户加载页面正是这个形态）
    if (req.resource_type == .document) {
        try self.addHeader("Sec-Fetch-User", "?1", .{});
        try self.addHeader("Upgrade-Insecure-Requests", "1", .{});
        try self.addHeader("Priority", "u=0, i", .{});
    }
}
```

- 依赖 `URL`（`src/browser/URL.zig`）与 `Cookie`（`storage/Cookie.zig`）两个 import，`HttpClient.zig` 顶部本来就有，不需要新加。
- `Cookie.areHostsSameSite` 已实现 eTLD+1 比较（走 public suffix list），不要自己重写。
- 六个头全部用 baseline 优先级（`HeaderOpts` 默认 `.user_agent`），不加 `.fixed`。
- 不会引发多余预检：`CorsGate` 只对 `.author`（脚本写的）头判 unsafe header。
- 实测结果：顶层导航 `document/navigate/none/?1` + U-I-R + `u=0, i`；classic script `script/no-cors/same-origin|cross-site`；module script `script/cors/cross-site`；XHR `empty/cors/same-origin`。

### 5.4 `src/browser/webapi/Navigator.zig`

7 处单行取值：

| 函数 | 上游 | 本分支 |
|---|---|---|
| `getDoNotTrack` | `null` | `"1"` |
| `getAppVersion` | `"1.0"` | `"5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0"` |
| `getHardwareConcurrency` | `4` | `32` |
| `getDeviceMemory` | `8.0` | `32` |
| `getMaxTouchPoints` | `0` | `10` |
| `getVendor` | `""` | `"Google Inc."` |
| `getPlatform` | 按 `builtin.os.tag` 分支（Linux 上 `Linux x86_64`） | 固定 `"Win32"`（Chrome 在 64 位系统上也返回 `Win32`） |

**其余一律保持上游原样：**

- `getLanguages` / `getLanguage` **不改**：用上游实现（读 `http_client.getLanguages()`），语言人设由 5.1(c) 驱动。
- 可见性随上游：`getDoNotTrack`、`getCookieEnabled`、`getMaxTouchPoints`、`getWebdriver`、`javaEnabled`、`sendBeacon`、`getPlugins`、`getPermissions`、`getGeolocation`、`getStorage`、`getUserAgentData`、`register/unregisterProtocolHandler` 上游已是 `fn`，**不要改回 `pub`**。
- 改完 `getPlatform` 后 `const builtin = @import("builtin");` 在本文件不再被引用，**保留那行不要删**（删了反而增 diff）。

### 5.5 `src/browser/webapi/NavigatorUAData.zig`

**(a) `uaPlatform()` 固定 `"Windows"`**（上游是按 `builtin.os.tag` 分支）

**(b) 新增 `shortBrandList()`，上游 `brandList()` 一字不改**

上游只有一个 `brandList()`，内部取 `b.full_version`。照抄的后果：`navigator.userAgentData.brands`（真实 Chrome 这里是主版本号 `151`）带上 `151.0.7813.2`，与 HTTP 头 `Sec-Ch-Ua` 对不上 → 交叉比对立刻判伪造。做法是保留上游函数（高熵用），只加一个低熵版：

```zig
// 本分支新增的唯一函数
fn shortBrandList() []const Brand {
    const out = comptime blk: {
        const src = &Config.HttpHeaders.brands;
        var arr: [src.len]Brand = undefined;
        for (src, 0..) |b, i| {
            arr[i] = .{ .brand = b.brand, .version = b.version };
        }
        const final = arr;
        break :blk final;
    };
    return &out;
}
```

**(c) 三处低熵调用点改成 `shortBrandList()`，高熵 4 个值写死**

```zig
fn getBrands(...) { return shortBrandList(); }            // 原 brandList()
pub fn toJSON(...) { .brands = shortBrandList(), … }       // 原 brandList()

fn getHighEntropyValues(...) !js.Promise {
    _ = hints;
    const brands = brandList();        // 上游行，保留：.fullVersionList 要用
    return exec.js.local.?.resolvePromise(.{
        .brands = shortBrandList(),    // 原 brandList()
        .mobile = false,
        .platform = uaPlatform(),
        .architecture = "x86",         // 原 uaArchitecture()
        .bitness = "64",               // 原 uaBitness()
        .model = "",
        .platformVersion = "15.0.0",   // 原 ""
        .uaFullVersion = "151.0.7813.2", // 原 if (brands.len > 0) brands[0].version else "1.0.0.0"
        .fullVersionList = brands,     // 上游行，未改
        .wow64 = false,
        .formFactor = [_][]const u8{"Desktop"},
    });
}
```

- `architecture`/`bitness` 必须写死：上游的 `uaArchitecture()`/`uaBitness()` 读 `builtin.cpu.arch`，在 ARM 机上编译就输出 `arm`/`32`，与 Windows x64 人设矛盾。
- 上游的 `uaFullVersion = brands[0].version` 取的是 brand 列表**第一项**，对本分支就是 `Not=A?Brand` 的 `"99"`（垃圾值），必须覆盖。
- `uaArchitecture()`/`uaBitness()` 变成未被引用的上游函数，**不要删**。
- 可见性随上游：`getBrands`/`getMobile`/`getPlatform`/`getHighEntropyValues` 是 `fn`，只有 `toJSON` 是 `pub fn`。

### 5.6 `src/browser/webapi/PluginArray.zig`

```zig
// 上游
pub const length = bridge.property(0, .{ .template = false });
// 本分支
pub const length = bridge.property(5, .{ .template = false });
```

底层数据源未改：`[...navigator.plugins]` 仍是空数组，`item()`/`namedItem()` 返回 `undefined`。只过"看长度"的检测。
这是 comptime 常量属性，会被烘进 `src/snapshot.bin` —— **改了必须重跑 `make build`（它前置生成快照）**，否则运行期仍是旧值。

### 5.7 `src/server/cdp/domains/emulation.zig` — `setUserAgentOverride`

只删一行 `error.Reserved => true,`（对应 5.1(d)），其余上游代码一字不改：

```zig
// 上游
const ua = params.userAgent;
const reserved = if (Config.validateUserAgent(ua)) false else |err| switch (err) {
    error.NonPrintable => return cmd.sendError(-32602, "User agent contains non-printable characters", .{}),
    error.Reserved => true,
};

// 本分支
const ua = params.userAgent;
// `error.Reserved` (a UA containing Mozilla) is intentionally not handled on
// this branch: Config.validateUserAgent never returns it, so `reserved` is
// always false and the Mozilla UA is accepted.
const reserved = if (Config.validateUserAgent(ua)) false else |err| switch (err) {
    error.NonPrintable => return cmd.sendError(-32602, "User agent contains non-printable characters", .{}),
};
```

后面紧跟的上游逻辑全部保留：`params.acceptLanguage` 处理（`Mime.isHttpHeaderValue` 校验 + `setAcceptLanguageOverride`）、`if (reserved) { … }` 块（本分支永不进入）、`setUserAgentOverride` + `bc.user_agent_changed`。

- 错误集收窄后 `switch (err)` 只列 `error.NonPrintable` 就是穷尽的。若上游给 `validateUserAgent` 加了新错误，这里会报「switch 未穷尽」，补上新错误的 prong 即可。
- 上游 `acceptLanguage` 处理必须收下（与 5.1(c) 配套）：它让 CDP/Playwright 下发的 `acceptLanguage` 同时覆盖 `Accept-Language` 头与 `navigator.languages`。

---

## 6. 上游同步 SOP

### 6.1 拓扑

```
lightpanda-io/browser（SSH，只读）= remote `upstream`
        │
        ▼
  本机 main 分支（跟踪上游，永远快进，保持纯净）
        │
        └── chrome 分支 ── 第 4 节那 7 个生产文件的改动（不碰测试代码）
                           备份推到 remote `origin`
```

```bash
# 远程配置（一次性；换机器 / 新仓库时要重做）
git remote rename origin upstream
git remote add origin git@github.com:<你的账号>/browser.git   # 占位符：换成你自己的备份仓库
git remote -v   # 两个都必须是指向 github.com 的 SSH 地址
```

> 本机所有 GitHub 远程一律走 SSH；HTTPS 地址不认 SSH key，且大陆直连会超时。

### 6.2 日常同步

```bash
# 0. 干跑：先看会撞哪几个文件（不动工作区）
git fetch upstream
git merge-tree --write-tree --name-only chrome upstream/main

# 1. main 快进到上游
git checkout main && git merge --ff-only upstream/main

# 2. chrome 合并 main
git checkout chrome && git merge main

# 3. 解冲突：只会出现在第 4 节那 7 个文件（测试代码永远与上游一致，天然不冲突）
for f in $(git diff --name-only --diff-filter=U); do git checkout --theirs "$f"; done
#    然后照第 5 节把 fork 行逐个贴回去，再 git add <file>

# 4. 静态核对（本分支不跑 make test）
zig fmt --check ./*.zig ./**/*.zig
git diff upstream/main --stat -- src        # 必须恰好 7 个文件

# 5. 依赖与环境：diff 这两个文件，变了就按 8.3/8.4 重抓
git diff <旧基线>..upstream/main -- build.zig.zon .github/actions/install/action.yml
make download-v8
zig build --fetch                           # EXIT=0 且无输出 = 可完全离线解析

# 6. 编译 + 验收（第 7 节），通过后提交
ZIGFLAGS="-Dcpu=skylake_avx512" make build
cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda && lightpanda version
git commit && git push origin chrome

# 7. 按 6.4 回写本文档
```

若上游改动让某处 fork 覆盖变多余（上游提供了同类开关），**优先删掉 fork 覆盖、改用上游开关**，并删掉本文档对应条目——这是增量不持续膨胀的唯一办法。

### 6.3 从零重建 chrome 分支（换新服务器 / 新仓库）

不需要 cherry-pick 旧 commit：第 5 节就是完整的 fork 源码，直接在上游最新代码上重贴一遍。

```bash
git clone git@github.com:lightpanda-io/browser.git browser && cd browser
git remote rename origin upstream
git remote add origin git@github.com:<你的账号>/browser.git   # 占位符
git checkout -b chrome                      # 基线 = 当时的 upstream/main
# 照第 5 节 5.1 → 5.7 逐文件贴 fork 改动（贴前先过第 2 节原则 1）
git diff upstream/main --stat -- src        # 必须恰好 7 个文件
zig fmt --check ./*.zig ./**/*.zig
# 环境按 8.1 → 8.7 装齐，编译 + 按第 7 节验收
git add -A && git commit -m "feat(chrome): 对齐真实 Chrome/Edge 的 UA 与客户端特征（详见 CHROME.md）" && git push -u origin chrome
```

> 备选搬代码法（旧分支基线很旧时才用）：`git checkout <旧chrome> -- <7 个文件>`，但**必须逐个与第 5 节核对**，否则会把旧基线的上游代码一起搬进来。
> 提交前 CHROME.md 必须与代码一致：文档与代码不同步比代码有 bug 更贵。

### 6.4 每次同步后必须回文档更新的地方

| 章节 | 要核什么 |
|---|---|
| 文档头 | `fork 基线` 改成新的上游 HEAD |
| 第 3 节 | 人设取值有没有因上游新机制而变动（如新增 client hint / locale 相关开关） |
| 第 4、5 节 | 与实际 `git diff upstream/main -- src` 逐条对齐；被上游机制取代的 fork 改动整节删掉 |
| 第 7 节 | 验收基准值是否需要刷新（实测后回写） |
| 第 8 节 | 编译/依赖流程本身有没有变（`build.zig` 选项、Makefile 目标、目录迁移） |
| 第 9 节 | 上游 API 补齐了哪条限制（补齐就从清单里删掉） |

核对命令（三条全绿才算同步完成）：

```bash
git diff upstream/main..HEAD --stat -- src   # 恰好 7 个文件
zig fmt --check ./*.zig ./**/*.zig            # 与 CI 一致
zig build --fetch                            # 依赖可离线解析
```

### 6.5 节奏

| 频率 | 操作 |
|---|---|
| 每周 | `git fetch upstream`，`git merge-tree` 干跑评估冲突 |
| 上游有 UA / client hint / navigator / locale / sec-fetch 相关变更 | 立即同步，确认第 1 节五条不变量未被削弱 |
| `build.zig.zon` / `action.yml` 变更 | 按 6.2 第 5 步重抓依赖，核对 V8 tag 与 Zig 版本 |

---

## 7. 验收清单

不要求 `make test` 全绿（见第 10 节）。前六步 CLI，第七、八步 CDP。第 2、3 步是必做的锁链校验：第 1 节不变量 2 的四组对应关系任一不成立，就是对外特征自相矛盾。

> URL 是**位置参数**，不是 `--url`（`src/cli.zig` 只对 `.options` 做名字匹配，positional 不参与；写 `--url` 会报 `unknown argument` 直接 fatal）。

```bash
# 1. 编译 + 安装
ZIGFLAGS="-Dcpu=skylake_avx512" make build
cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda && lightpanda version
#    日志必须出现两次 "Using prebuilt V8: ..."（snapshot 与主程序各一次）

# 2. HTTP 请求头：与第 3 节表格逐项核对
lightpanda fetch --dump html "https://httpbin.org/headers"
#   Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
#   Sec-Fetch-Dest: document / Mode: navigate / Site: none / User: ?1
#   Upgrade-Insecure-Requests: 1  Priority: u=0, i      <-- 只在顶层导航出现

# 3. JS 侧指纹与 HTTP 头锁得住（核心一致性检查）
#    脚本必须包在显式 <body> 里：写成 data:text/html,<script>…document.body… 会在 <head>
#    里执行，此时 document.body 为 null，真实浏览器同样报 TypeError
lightpanda fetch --dump markdown 'data:text/html,<body><script>document.body.textContent=[navigator.language,navigator.languages.join(","),navigator.hardwareConcurrency,navigator.deviceMemory,navigator.maxTouchPoints,navigator.vendor,navigator.platform,navigator.doNotTrack,navigator.userAgentData.brands.map(function(b){return b.brand+";v="+b.version}).join(" | ")].join("\n")</script></body>'
#   期望：zh-CN / zh-CN,zh,en / 32 / 32 / 10 / Google Inc. / Win32 / 1 /
#         Not=A?Brand;v=99 | Microsoft Edge;v=151 | Chromium;v=151
#   brands 必须是短版本号 151（出现 151.0.7813.2 = 5.5 的低熵覆盖丢了）

# 4. 语言开关仍由上游机制驱动（证明没退回硬编码）
lightpanda fetch --locale en-US --dump html "https://httpbin.org/headers"
#   Accept-Language 应变回 en-US,en;q=0.9

# 5. 浏览器特征检测站复核（公开站点，用来比对特征是否与真实浏览器一致）
#    bot.sannysoft 同步渲染，直接 dump；browserscan/creepjs 的结果卡片由异步 JS + worker +
#    iframe 算，默认参数下 dump 不到判定（只能拿到静态外壳），必须开资源加载并等静默
lightpanda fetch --dump html "https://bot.sannysoft.com"
lightpanda fetch --load-resources iframe,worker --wait-until networkidle --wait-ms 15000 --dump markdown "https://www.browserscan.net/bot-detection"
lightpanda fetch --load-resources iframe,worker --wait-until networkidle --wait-ms 15000 --dump markdown "https://abrahamjuliot.github.io/creepjs/"

# 5b. 已实测基准（同一套命令的正常输出长这样，偏离就是回归）
#   browserscan Bot Detection：WebDriver / WebDriver Advance / Selenium / Webdriverio /
#     NightmareJS / PhantomJS / Awesomium / Cef / CefSharp / Coaches / FMiner / Born /
#     Phantomas / Rhino / Headless Chrome / CDP / Dev Tool —— 共 17 项全 Normal，0 Abnormal
#   bot.sannysoft：仅 window.chrome（Chrome New / HEADCHR_CHROME_OBJ）与 getBattery（CHR_BATTERY）
#     三项 failed（上游 API 缺失，见第 9 节）；其余包括 Plugins Length=5、languages=[zh-CN,zh,en]、
#     PHANTOM_PROPERTIES / SELENIUM_DRIVER / HEADCHR_PLUGINS / HEADCHR_IFRAME 均 ok
#   creepjs：因 getExtentOfChar / outerHeight / mediaDevices 缺失算不出评分（卡片大量
#     "0% of engine"、"keys (0)"），不作为回归依据

# 6. CLI 层 Sec-Ch-Ua 拦截范围（上游行为，本分支不改，只验它没扩大）
lightpanda fetch --http-header 'Sec-Ch-Ua: "Chromium";v="999"' --dump html "https://httpbin.org/headers"
#   预期 fatal："Sec-Ch-Ua is not overridable"
lightpanda fetch --http-header 'Sec-Ch-Ua-Platform: "macOS"' --dump html "https://httpbin.org/headers"
#   预期正常执行且 Sec-Ch-Ua-Platform 变成 "macOS"（上游是全等匹配，没堵住 5 个 hint）
```

### 第七步：CDP 验证（本分支两大核心目的只能在这里验）

零依赖，Node ≥ 22 自带 `WebSocket`：

```bash
lightpanda serve --host 127.0.0.1 --port 9222 2>&1 | tee /tmp/lp.log   # 你自己启动
# 另开终端：
cat > /tmp/cdp_check.mjs <<'MJS'
const BASE = "http://127.0.0.1:9222";
const UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
const { webSocketDebuggerUrl } = await (await fetch(`${BASE}/json/version`)).json();
const ws = new WebSocket(webSocketDebuggerUrl);
await new Promise((r) => (ws.onopen = r));
let seq = 0; const pending = new Map();
ws.onmessage = (e) => {
  const m = JSON.parse(e.data), p = pending.get(m.id);
  if (p) (pending.delete(m.id), m.error ? p.reject(m.error) : p.resolve(m.result));
};
const send = (method, params = {}, sessionId) => new Promise((resolve, reject) => {
  const id = ++seq; pending.set(id, { resolve, reject });
  ws.send(JSON.stringify({ id, method, params, ...(sessionId ? { sessionId } : {}) }));
});
const evalJs = async (sid, expression) =>
  (await send("Runtime.evaluate", { expression, returnByValue: true }, sid)).result.value;

const { targetId } = await send("Target.createTarget", { url: "about:blank" });
const { sessionId } = await send("Target.attachToTarget", { targetId, flatten: true });
const sid = sessionId;

// A: 含 Mozilla 的 UA 必须被接受（对应 5.1(d) + 5.7）
await send("Emulation.setUserAgentOverride", { userAgent: UA, acceptLanguage: "en-US,en;q=0.9" }, sid);
console.log("A1 navigator.userAgent =", await evalJs(sid, "navigator.userAgent"));
console.log("A2 navigator.languages =", await evalJs(sid, "JSON.stringify(navigator.languages)"));

// B: 真实请求头跟着变
await send("Page.navigate", { url: "https://httpbin.org/headers" }, sid);
await new Promise((r) => setTimeout(r, 2500));
console.log("B headers =", await evalJs(sid, "document.body.innerText"));

// C: Network.setExtraHTTPHeaders 覆盖 Sec-Ch-Ua（对应 5.3(a) 拆 .fixed）
//    Network.enable 必须在前面：extra_headers 是在 Network.httpRequestStart 里应用的，
//    该回调由 Network.enable → bc.networkEnable() 的 notification.register 注册；
//    不 enable 就没人订阅通知，setExtraHTTPHeaders 静默无效（不报错、不刷 warn）。
await send("Network.enable", {}, sid);
await send("Network.setExtraHTTPHeaders", { headers: { "Sec-Ch-Ua": '"Chromium";v="999"', "x-cdp-probe": "42" } }, sid);
await send("Page.navigate", { url: "https://httpbin.org/headers" }, sid);
await new Promise((r) => setTimeout(r, 2500));
console.log("C headers =", await evalJs(sid, "document.body.innerText"));
ws.close();
MJS
node /tmp/cdp_check.mjs
```

| 输出 | 必须是 | 否则说明 |
|---|---|---|
| `A1` | 完整的 Edge 151 Mozilla UA | 5.7 的 `error.Reserved` prong 没删净，被静默忽略 |
| `A2` | `["en-US","en"]` | `acceptLanguage` 没接入 `setAcceptLanguageOverride` |
| `B` | UA = A1 值、`Accept-Language: en-US,en;q=0.9` | 覆盖只作用 JS、没进真实请求头 |
| `C` | `Sec-Ch-Ua` = `"Chromium";v="999"`，且出现 `X-Cdp-Probe: 42` | 上游把 `.source = .fixed` 加回来了，或漏发 `Network.enable` |
| `/tmp/lp.log` | **不得出现** `User agent must not contain Mozilla`、`ignore overriding fixed header` | 分支核心目的失效 |

### 第八步：Sec-Fetch 逐资源类型自洽性（验 5.3(b)）

`--dump` 只能看到顶层导航；子资源要看 CDP 的 `Network.requestWillBeSent`：

```bash
cat > /tmp/cdp_secfetch.mjs <<'MJS'
const { webSocketDebuggerUrl } = await (await fetch("http://127.0.0.1:9222/json/version")).json();
const ws = new WebSocket(webSocketDebuggerUrl);
await new Promise((r) => (ws.onopen = r));
let seq = 0; const pending = new Map();
ws.onmessage = (e) => {
  const m = JSON.parse(e.data);
  if (m.method === "Network.requestWillBeSent") {
    const h = m.params.request.headers;
    const pick = Object.fromEntries(Object.entries(h).filter(([k]) => /^sec-fetch|^upgrade-insecure|^priority$/i.test(k)));
    console.log(String(m.params.type).padEnd(12), JSON.stringify(pick));
  }
  const p = pending.get(m.id);
  if (p) (pending.delete(m.id), m.error ? p.reject(m.error) : p.resolve(m.result));
};
const send = (method, params = {}, sessionId) => new Promise((resolve, reject) => {
  const id = ++seq; pending.set(id, { resolve, reject });
  ws.send(JSON.stringify({ id, method, params, ...(sessionId ? { sessionId } : {}) }));
});
const { targetId } = await send("Target.createTarget", { url: "about:blank" });
const { sessionId } = await send("Target.attachToTarget", { targetId, flatten: true });
await send("Network.enable", {}, sessionId);
await send("Page.navigate", { url: "https://bot.sannysoft.com" }, sessionId);
await new Promise((r) => setTimeout(r, 8000));
ws.close();
MJS
node --check /tmp/cdp_secfetch.mjs && node /tmp/cdp_secfetch.mjs
```

| 条目类型 | 应包含 |
|---|---|
| Document | `dest: document`、`mode: navigate`、`site: none`、`user: ?1`、`upgrade-insecure-requests: 1`、`priority: u=0, i` |
| Script（classic `<script src>`） | `dest: script`、**`mode: no-cors`**、`site: same-origin` 或 `cross-site`，**无** user / U-I-R / priority |
| Script（module 或 `crossorigin`） | `dest: script`、`mode: cors`、`site: …` |
| Image | `dest: image`、`mode: no-cors`、`site: …` |
| Stylesheet | `dest: style`、`mode: cors`、`site: …` |
| XHR / Fetch | `dest: empty`、`mode: cors`、`site: …` |
| Worker | 不发任何 sec-fetch（表里不猜，见第 9 节） |

---

## 8. 环境与编译

全新机器 / 换服务器：按 8.1 → 8.7 完整走一遍（或直接跑 8.7 的一键脚本）。日常同步上游：只复查 8.3（V8 tag）与 8.4（依赖）。

### 8.1 系统依赖（Ubuntu 22.04+ / Debian 12+）

```bash
apt-get update && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config libglib2.0-dev clang make curl git
```

### 8.2 Zig（版本取自 `build.zig.zon`）

```bash
ZIG_VERSION="$(grep -oP '(?<=minimum_zig_version = ")[^"]+' build.zig.zon)"
curl -LO https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz
tar xf zig-x86_64-linux-${ZIG_VERSION}.tar.xz
rm -rf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}
mv zig-x86_64-linux-${ZIG_VERSION} /usr/local/lib
ln -sf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}/zig /usr/local/bin/zig
zig version
```

### 8.3 预编译 V8 库（约 127 MB）

```bash
make download-v8
```

路径由仓库派生：`Makefile` 从 `action.yml` 读 `zig-v8`（tag）与 `v8`（版本），组成 `V8_CACHE := .lp-cache/prebuilt-v8/<tag>/libc_v8_<v8>_<os>_<arch>.a`；`build.zig` 的 `findPrebuiltV8()` 用同一套规则自动发现，**正常不需要传 `-Dprebuilt_v8_path`**。

两个坑：

1. tag 升级时资源文件名可能不变但字节不同（文件名只编 V8 版本），所以缓存路径里的 `<tag>` 一层不能省；链接期报 undefined `v8__*` 基本就是缓存里躺着旧 tag 的 `.a`。
2. `download-v8` 有 `test -f` 守卫，中途被杀掉的半截文件会被当成已就绪。GitHub 直连超时时走镜像，并先落 `.part`：

```bash
TAG=$(awk -F\' '/^  zig-v8:/{f=1} f&&/default:/{print $2; exit}' .github/actions/install/action.yml)
V8=$(awk -F\' '/^  v8:/{f=1} f&&/default:/{print $2; exit}' .github/actions/install/action.yml)
DIR=.lp-cache/prebuilt-v8/$TAG; mkdir -p $DIR
REL="https://ghfast.top/https://github.com/lightpanda-io/zig-v8-fork/releases/download/$TAG"

# 静态库：发布名与缓存名一致
A="libc_v8_${V8}_linux_x86_64.a"
curl -fL --retry 3 --no-progress-meter -o "$DIR/$A.part" "$REL/$A" && mv -f "$DIR/$A.part" "$DIR/$A"

# 共享库：发布名带版本，但缓存名固定是 libc_v8.so（exe 的 DT_NEEDED 记录的就是这个名字）
S="libc_v8_${V8}_linux_x86_64.so"
curl -fL --retry 3 --no-progress-meter -o "$DIR/libc_v8.so.part" "$REL/$S" && mv -f "$DIR/libc_v8.so.part" "$DIR/libc_v8.so"

ls -l $DIR      # 字节数应与 curl -sIL "$REL/$A" 返回的 content-length 一致
```

> `libc_v8.so` 是 `-Ddev_fast` 用的（Linux x86_64 的 Debug 构建默认命中），`make download-v8` 也会抓它；缺了 `make build-dev` 会退回源码编 V8。

### 8.4 依赖离线抓取（大陆网络）

本机直连 GitHub HTTPS 会超时（`HttpConnectionClosing` / `Timeout`），SSH 可用。镜像/SSH 取回的字节与原站一致 → `zig fetch` 算出的 hash 与 `build.zig.zon` 精确匹配 → 落全局缓存 `~/.cache/zig/p/`，`build.zig.zon` 一行都不用改。

要抓哪些**不凭记忆也不照抄本文档**，直接从仓库读；同步后用 diff 判断哪些项变了：

```bash
git diff <旧基线>..HEAD -- build.zig.zon

# 1) tarball 类（.url = https://github.com/…tar.gz）——逐个加镜像前缀
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep -v '^git+'); do
  timeout 600 zig fetch "https://ghfast.top/$u" || echo "FAILED: $u"
done

# 2) git+https 类——GIT_CONFIG 临时把 https 改写成 SSH，不动全局 git 配置
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep '^git+https'); do
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0="url.git@github.com:.insteadOf" GIT_CONFIG_VALUE_0="https://github.com/" \
    timeout 600 zig fetch "$u" || echo "FAILED: $u"
done

# 3) 预编译 V8 .a / .so（裸二进制，不是 zig 包）见 8.3
# 4) 验证：EXIT=0 且无输出 = 完全离线可解析
zig build --fetch
```

> 校验：`zig fetch` 输出的 hash 必须与 `build.zig.zon` 对应 `.hash` 完全一致。备选镜像 `https://gh-proxy.com/`。
> 注意：即使不传 `-Dprebuilt_v8_path`，`.v8` 依赖 tarball（Zig/C 绑定源码）仍必须能解析，预编译 `.a` 替代不了它。
> 判定"哪些包是过期的"必须走依赖图闭包（从根 `build.zig.zon` 出发，逐包读各自的 `build.zig.zon` 递归到不动点）——根清单里没有的**传递依赖**照样在用，只看根清单会误删。

### 8.5 编译命令

```bash
# 生产（release + V8 snapshot）。编译机是 AMD EPYC 9T25（Zen 5），直接 make build
# 会带 Zen 5 特有指令，在 Intel Skylake-SP 生产机上运行会 SIGILL
ZIGFLAGS="-Dcpu=skylake_avx512" make build

ZIGFLAGS="-Dcpu=baseline" make build          # 要兼容更老 CPU 时
ZIGFLAGS="-Dcpu=skylake_avx512" zig build check   # 只做编译检查不链接，全量前的快速门
make build-dev                                 # Debug（Linux x86_64 走 dev_fast + libc_v8.so）
make clean                                     # 清理（保留 .lp-cache 里的 V8 缓存）
```

- `ZIGFLAGS` 统一用**环境变量形式**（上游 `Makefile` 自己给的写法：`# ZIGFLAGS=-Ddev_fast=false make test`）。命令行形式 `make build ZIGFLAGS="…"` 在 GNU Make 4.3 上实测也可用（子 make 收到 `MAKEFLAGS=[… -- ZIGFLAGS=-Dcpu=…\ -Dprebuilt_v8_path=…]`，空格被转义成单个赋值），所以不是兼容性问题，只是写法统一。
- **`Makefile` 与 `build.zig` 保持与上游零 diff**（第 2 节原则），不要为了 ZIGFLAGS 去恢复 `MAKEOVERRIDES =`。
- `make build` **不编译 V8 引擎**：第一步编 `snapshot_creator` 生成 `src/snapshot.bin`（做快照，不是编 V8），第二步编主程序并链接预编译 `.a`。改一个 `.zig` 文件只需几分钟（依赖与对象文件全部复用 `.zig-cache`）；只有 `rm -rf .zig-cache` 或 `make clean` 之后才是冷编译。退回源码编 V8（10+ 分钟）只有两种：日志出现 `No prebuilt V8 at …`，或开了 `-Dtsan` / `-Dasan`。
- 快照对本分支是**必需**的：`PluginArray.length=5` 这类 comptime 常量属性会被烘进 `snapshot.bin`，不重生成快照则运行期仍是旧值。
- C/Rust 依赖不跟随 `-Doptimize`，固定 ReleaseFast（`-Ddebug_deps` 才转 Debug），debug/release 共用一套依赖对象缓存。
- 产物：`./zig-out/bin/lightpanda`；安装位置 `~/.local/bin/lightpanda`；快照 `src/snapshot.bin`；V8 缓存 `.lp-cache/prebuilt-v8/<tag>/`。
- 格式化必须与 CI 一致：`zig fmt --check ./*.zig ./**/*.zig`（`zig build` 依赖 fmt step）。

### 8.6 Rust（国内镜像 rsproxy.cn）

```bash
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y
source $HOME/.cargo/env
mkdir -p ~/.cargo && cat > ~/.cargo/config.toml << 'CARGO_EOF'
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"
[net]
git-fetch-with-cli = true
CARGO_EOF
```

`RUSTUP_*` 两个变量一并写进 `~/.bashrc`。另需 `export LIGHTPANDA_DISABLE_TELEMETRY=1`，`PATH` 里要有 `/usr/local/bin`、`$HOME/.cargo/bin`、`$HOME/.local/bin`。

### 8.7 一键初始化（新服务器）

```bash
#!/bin/bash
set -euo pipefail
PROJECT_DIR="${1:-$(pwd)}"
apt-get update -qq && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config libglib2.0-dev clang make curl git

ZIG_VERSION="$(grep -oP '(?<=minimum_zig_version = ")[^"]+' "$PROJECT_DIR/build.zig.zon")"
cd /tmp && curl -fsSLO "https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz"
tar xf "zig-x86_64-linux-${ZIG_VERSION}.tar.xz"
rm -rf "/usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}"
mv "zig-x86_64-linux-${ZIG_VERSION}" /usr/local/lib
ln -sf "/usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}/zig" /usr/local/bin/zig && rm -f "zig-x86_64-linux-${ZIG_VERSION}.tar.xz"
zig version

export RUSTUP_DIST_SERVER="https://rsproxy.cn" RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y
export PATH="$HOME/.cargo/bin:$PATH"
mkdir -p ~/.cargo && cat > ~/.cargo/config.toml << 'CARGO_EOF'
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"
[net]
git-fetch-with-cli = true
CARGO_EOF

cd "$PROJECT_DIR"
make download-v8 || true                      # 直连失败就走 8.3 的镜像法
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep -v '^git+'); do
  zig fetch "https://ghfast.top/$u" || echo "FAILED: $u"; done
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep '^git+https'); do
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0="url.git@github.com:.insteadOf" GIT_CONFIG_VALUE_0="https://github.com/" \
    zig fetch "$u" || echo "FAILED: $u"; done
zig build --fetch

export LIGHTPANDA_DISABLE_TELEMETRY=1
ZIGFLAGS="-Dcpu=skylake_avx512" make build
mkdir -p "$HOME/.local/bin" && cp -f zig-out/bin/lightpanda "$HOME/.local/bin/lightpanda"
echo "完成：lightpanda serve --host 0.0.0.0 --port 9222"
```

用法：`chmod +x init.sh && sudo ./init.sh /path/to/browser`

### 8.8 排障

| 现象 | 处理 |
|---|---|
| html5ever / Rust 依赖编译失败 | `cargo --version`；确认 8.6 的镜像配置 |
| `zig version` 与 `minimum_zig_version` 不符 | 按 8.2 重装 |
| V8 下载失败 / 只有半截文件 | 按 8.3 的镜像法（`.part` + `mv -f`） |
| `zig build` 卡在依赖解析 / `HttpConnectionClosing` | 按 8.4 离线抓取，再 `zig build --fetch` 验证 |
| 输出里没有 `Using prebuilt V8: …` | 缓存不在 `action.yml` 所写 tag 的目录里：重跑 `make download-v8` |
| 链接期报 undefined `v8__*` 符号 | 缓存里是旧 tag 的 `.a`；重跑 `make download-v8`（缓存按 tag 分目录） |
| `make build` 比往常慢很多 | 正常：清过 `.zig-cache` / `make clean` 就是冷编译。若日志没有 `Using prebuilt V8: …` 才是在源码编 V8 |
| 生产机运行时 `SIGILL` | 编译时没加 `-Dcpu=skylake_avx512`（或改用 `-Dcpu=baseline`） |
| 个别站点报 SSL 连接错误 | 见第 9 节 ALPN 条目，先试 `--http-version 1.1` |
| CDP `setExtraHTTPHeaders` 静默无效 | 漏发 `Network.enable`（见第 7 节第七步 C 的说明） |
| `fetch` 报 `unknown argument --url` | URL 是位置参数：`lightpanda fetch [flags] "<url>"` |

---

## 9. 已知限制与"明确不做"的决定

1. **TLS 指纹（JA3/JA4）**：curl 链接 **BoringSSL**（`build.zig.zon` 的 `boringssl-zig`，`build.zig` 里 `CURL_DEFAULT_SSL_BACKEND = "openssl"` 只是兼容层名字），比通用 OpenSSL 更接近 Chrome 的握手段，但 ClientHello 的扩展顺序、GREASE、压缩方法仍与真实 Chrome 不同。要完全一致需要 curl-impersonate 级改造。
2. **ALPN 失败更敏感**：BoringSSL 在 ALPN 未协商成功时比 OpenSSL 严格，个别站点会报 SSL 连接错误；可用 `--http-version 1.1` 退化绕开（只接受 `auto` / `1.1`）。
3. **HTTP/2 指纹**：SETTINGS 帧参数、WINDOW_UPDATE、伪头顺序与真实浏览器不同。
4. **Canvas / WebGL**：headless 无 GPU，Canvas 为软件渲染特征；bot.sannysoft 的 Canvas/WebGL 行为空。
5. **WebRTC**：headless 下无 WebRTC 或返回异常 IP；`navigator.getUserMedia` 未实现。
6. **`--http-header "Sec-Ch-Ua: …"` 被 CLI 层硬性拒绝**（`hint = "Sec-Ch-Ua is not overridable"`，上游既有行为）。**决定：不改**——真实浏览器没有这个入口，屏蔽它不影响对外特征，放开只会多一处冲突面。精确范围：仅拦 `Sec-Ch-Ua` 一个名字（`eqlIgnoreCase` 全等匹配）；`Sec-Ch-Ua-Full-Version-List` 与 5 个 `-Platform/-Mobile/-Arch/-Bitness/-WoW64` 都可用 `--http-header` 覆盖；CDP 路径下 `Sec-Ch-Ua` 已实测可覆盖（需先 `Network.enable`）。
7. **`navigator.plugins` 内容仍为空**（只改了 `length`，见 5.6）。
8. **CDP 无法运行时改 Intl/Date 的 locale/timezone**：上游 `Emulation.setLocaleOverride` / `setTimezoneOverride` 是 noop（需要 zig-v8-fork 暴露 `Isolate::DateTimeConfigurationChangeNotification` 与 ICU 默认 locale 绑定），只能靠进程参数 `--locale` / `--timezone`。
9. **`window.chrome`、`navigator.getBattery`、`navigator.getUserMedia`、`window.outerHeight/outerWidth` 未实现**（上游 API 缺失）：bot.sannysoft 上表现为 `Chrome (New) missing (failed)`、`HEADCHR_CHROME_OBJ FAIL`、`CHR_BATTERY FAIL`，creepjs 报 `ReferenceError: outerHeight is not defined`。`window.chrome` 是个很小的空对象桩，getBattery 要实现 Promise 对象——**待决策，未做**。
10. **Sec-Fetch 的覆盖边界**（5.3(b) 已补齐主链路）：`worker` 类型不发 fetch 元数据（规范 destination 细分太细，不猜）；子资源不发 `Priority`（真实 Chrome 每请求都带，但 urgency 依赖一堆启发式）；`Accept-Encoding` 仍是 `deflate, gzip, br`，真实 Edge 是 `gzip, deflate, br, zstd`（由 curl 编译选项决定）。
    两条已知不一致：① 开 `--load-resources iframe` 后子帧文档请求会自称 `dest: document` + `user: ?1`（Chrome 是 `iframe` 且不带 user flag）——`Request` 上找不出可靠的"是否顶层"信号（`document_frame_id` 对子帧也等于自身 `frame_id`，见 `global_scope.zig:164`），宁可不猜；② 导航的 `sec-fetch-site` 恒为 `none`，页内跳转时 Chrome 会给 `same-origin`/`same-site`/`cross-site`，要真值必须把发起文档从 `Frame.navigate` 当 `cookie_origin` 传下来（上游注释已为这件事留了字段），但那会新增一个 fork 文件，按最小增量原则暂不做，代价是带 Referer 的页内跳转会 `none` + Referer 轻度不一致。
11. **换 UA 人设时 client hints 不跟着变**：`--user-agent` 只改 UA（实测 UA 变 Chrome/120 而 `Sec-Ch-Ua` 仍是 151），`--http-header` 又拦住 `Sec-Ch-Ua`（见第 6 条）。要换版本人设：走 CDP（`Network.enable` + `setExtraHTTPHeaders`，实测可覆盖），或同时改代码里的 `user_agent_base` + `brands`（第 3 节末尾那四处）。
12. 小噪声（非问题）：httpbin 把请求头名按首字母大写重新格式化，回显成 `Sec-Ch-Ua-Wow64`；实际发出的是 `Sec-Ch-Ua-WoW64`（`baselineHeaders` 可证）。

---

## 10. 测试策略

**本分支不维护任何测试代码的适配。** 上游测试断言的是上游默认行为，本分支故意改了生产行为，因此下列测试必然失败——这是预期结果，**不修、不删、不改**（`test {}` 块不参与 `make build`，不影响生产二进制），`make test` 不作为验收步骤：

| 失败测试 | 原因 |
|---|---|
| `Config: validateUserAgent`、`Config: parseArgs refuses a mozilla user-agent` | 断言 Mozilla UA 被拒 |
| `Config: locale drives http_headers` | 断言默认 `Accept-Language` 为 `en-US,en;q=0.9` |
| `cdp.Emulation: setUserAgentOverride ignores mozilla` / `… case insensitive` | 断言含 Mozilla 的 UA 被忽略 |
| `cdp.Emulation: setUserAgentOverride acceptLanguage drives navigator.languages` | 首句断言默认 `navigator.language === 'en-US'` |
| `cdp.network setExtraHTTPHeaders rejects a Mozilla User-Agent` / `… smuggled via a colon in the key` | 断言拒绝 Mozilla UA |
| `WebApi: Navigator` / `WebApi: NavigatorUAData` 系列 | 断言 `vendor === ''`、`platform` 跟编译机、`doNotTrack === null`、默认并发/内存/触点数、`brands` 为 `Lightpanda` 全版本号 |
| fetch / CDP Network 域里断言"请求头集合恰好等于某几个"的用例 | 5.3(b) 给每个请求新增了 Sec-Fetch 元数据 |

> 唯一需要守住的纪律：**永远不要把这类测试"反向改写"成断言接受 Mozilla UA**。那会让测试变绿但污染测试代码，并使 `git diff upstream/main --stat -- src` 多出文件——违反第 2 节原则 6。
