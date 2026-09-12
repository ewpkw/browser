# Chrome 分支：把 Lightpanda 伪装成真实 Chrome/Edge

> **fork 基线**：上游 `lightpanda-io/browser` 的 `main` @ `e8aa75939`。本文档描述 `chrome` 分支相对该基线的**全部**生产代码改动，以及环境、编译、同步、验收的完整流程。
> 第 3、4 节就是当前 fork 增量的全部内容，与 `git diff upstream/main..HEAD --stat -- src` 一一对应；任何超出这两节的文件都说明本分支被污染了。
>
> **版本类信息不在本文档里写死**，永远从仓库本身读，免得文档过期：
>
> | 要查的东西 | 唯一来源 |
> |---|---|
> | Zig 版本 | `build.zig.zon` 的 `minimum_zig_version` |
> | V8 版本与 `zig-v8` release tag | `.github/actions/install/action.yml` 里 `v8:` 与 `zig-v8:` 两个 `default:` |
> | 预编译 V8 缓存路径 | `Makefile` 的 `V8_CACHE`（由上面两项拼出） |
> | 全部依赖及其 URL/hash | `build.zig.zon` 的 `.dependencies` |

---

## 0. 本分支存在的意义（唯一目的）

上游 Lightpanda 在多个地方**主动拒绝**把自己伪装成别的浏览器：默认 UA 是 `Lightpanda/1.0`、`--user-agent` 含 `Mozilla` 会被拒绝、CDP `Emulation.setUserAgentOverride` 传 Mozilla UA 会被静默忽略、`navigator.*` 到处露出 headless/Linux 特征。

本分支的存在就是为了**反过来**：接受并使用 Mozilla/Chrome/Edge 的 UA 与全部派生信号，让 Cloudflare、Akamai、DataDome 一类反爬在前端特征层把本进程当作一个正常的 Windows Chrome/Edge 客户端。

这条原则优先于"跟随上游"：**Mozilla UA 必须被接受，任何同步都不能让这条路重新被封死。**

本文档只描述生产代码（`make build` 编译进二进制的部分）。本分支**不修改任何测试文件、不改任何 `test {}` 块**——上游测试（包括上游自己写的"拒绝 Mozilla UA"断言）一律保持原样，不参与适配。

---

## 1. 维护原则（每次同步上游时的决策顺序）

长期跟随上游是硬需求，**每一行 fork 改动都是下一次同步的冲突成本**。按下面 1→6 的顺序决策：

1. **上游已有开关的，绝不再自己硬编码。**
   每发现一处想覆盖的地方，先问："上游有没有 CLI/配置项能做到同一件事？"有就只改默认值。
   范例：`navigator.languages` / `navigator.language` / `Accept-Language` / Intl 默认 locale 上游现在统一由 `--locale` 驱动，本分支因此只改 `Config.HttpHeaders.default_locale` 一行，另外三处（`Navigator.getLanguages`、`Navigator.getLanguage`、`WorkerNavigator.getLanguages`）全部回归上游原样。
2. **改前先还原，再贴回 fork 行。**
   冲突文件一律 `git checkout --theirs <file>` 取上游原版，然后把本文档第 4 节列出的那几行重新贴上去。**禁止**在 fork 的旧结构上手工缝合——那样会把上游对该文件的改进整块丢掉。
3. **能删就删，不留"以后可能用"的 delta。**
   fork 里任何不再被引用的常量/函数一律删掉。历史上 `Config.HttpHeaders` 里那组 `sec_ch_ua_platform` / `_mobile` / `_arch` / `_bitness` / `_wow64` 常量从未被 `baselineHeaders()` 引用，属于纯冲突放大器，已删除；Client-Hint 字面值只存在于 4.3 一处。
4. **单行优先，不动结构。**
   值型改动只改 `return` 那一行：不改函数签名、不改 `pub`/`fn` 可见性、不重排邻近代码、不删上游未被引用的辅助函数（Zig 对容器级未使用声明不报错）。上游把 `pub fn` 收成 `fn` 时跟着收。
5. **保留不可达的上游代码。**
   例如 `emulation.zig` 里 `if (reserved) { ... }` 块在本分支永不进入，仍然原样保留：删得越多，下次 diff 越大。理想状态是"只删一行、其余一字不改"。
6. **每处 fork 改动都必须在第 4 节有对应条目。**
   核对命令：

   ```bash
   git diff upstream/main..HEAD --stat -- src     # 必须恰好是第 3 节那 7 个文件
   ```

   多出来的文件（尤其测试文件）说明有人偏离了本原则，按第 5 节原则回退。

---

## 2. 目标人设：最终对外的全部取值

| 通道 | 实际值 |
|---|---|
| `User-Agent` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0` |
| `Sec-Ch-Ua` | `"Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"` |
| `Sec-Ch-Ua-Full-Version-List` | `"Not=A?Brand";v="99", "Microsoft Edge";v="151.0.7813.2", "Chromium";v="151.0.7813.2"` |
| `Sec-Ch-Ua-Platform` / `-Mobile` / `-Arch` / `-Bitness` / `-WoW64` | `"Windows"` / `?0` / `"x86"` / `"64"` / `?0` |
| `Accept-Language` | `zh-CN,zh;q=0.9,en;q=0.8`（由 `--locale` 推导） |
| `navigator.language` / `.languages` | `zh-CN` / `[zh-CN, zh, en]`（与 `Accept-Language` 同源） |
| `navigator.appVersion` | UA 去掉 `Mozilla/` 前缀（必须与 `navigator.userAgent` 交叉一致） |
| `navigator.platform` / `vendor` / `doNotTrack` | `Win32` / `Google Inc.` / `"1"` |
| `navigator.hardwareConcurrency` / `deviceMemory` / `maxTouchPoints` | `32` / `32` / `10` |
| `navigator.userAgentData.brands` | 短版本号：`Not=A?Brand 99` / `Microsoft Edge 151` / `Chromium 151` |
| `navigator.userAgentData.getHighEntropyValues()` | `architecture=x86`、`bitness=64`、`platform=Windows`、`platformVersion=15.0.0`、`uaFullVersion=151.0.7813.2`、`model=""`、`wow64=false`、`formFactor=[Desktop]` |
| `navigator.plugins.length` | `5` |
| Intl / `toLocaleString` 默认 locale | `zh-CN`（上游 `Platform.init` 在 `InitializeICU` 前 `setenv("LC_ALL", tag)`） |

`--user-agent-suffix` 拼在上面那个默认 UA 之后；`--user-agent` 整体覆盖（本分支允许含 Mozilla）。

### 2.1 升级人设版本（Edge/Chromium 出新版本）时的必改清单

人设版本号在代码里是**有意写死**的字面量（不靠 `builtin` / `lp.build_config` 派生，否则编译机差异会泄馅），代价就是升级时要把相关的几处一起改掉，彼此不一致 = 反爬特征：

| # | 位置 | 改什么 |
|---|---|---|
| 1 | `Config.zig` `user_agent_base` | UA 里的 `Chrome/<短>` 与 `Edg/<短>`（真实 Edge 的 UA 段始终是 `<短>.0.0.0`） |
| 2 | `Config.zig` `HttpHeaders.brands` | `.version`（短号）与 `.full_version`（完整构建号） |
| 3 | `Navigator.zig` `getAppVersion` | UA 去掉 `Mozilla/` 前缀的那一串，必须与 1 逐项相同 |
| 4 | `NavigatorUAData.zig` `getHighEntropyValues()` | `uaFullVersion` 必须等于 2 里 Edge/Chromium 的 `.full_version` |

**三条锁链必须成立**（第 9 节有验收命令）：

- `Sec-Ch-Ua` 头 ≡ `navigator.userAgentData.brands`（都取 `.version`）
- `Sec-Ch-Ua-Full-Version-List` 头 ≡ `getHighEntropyValues().fullVersionList`（都取 `.full_version`）
- `navigator.userAgent` 去前缀 ≡ `navigator.appVersion`；`uaFullVersion` ≡ `fullVersionList` 中本浏览器品牌的版本

> 不做“从 `brands` 派生 `uaFullVersion`”的自动化：那要在 `NavigatorUAData.zig` 里新增一段 comptime 查找，把 1 行 fork delta 变成 6~10 行，与第 1 节“最小增量”相悖；而升级 Edge 版本是低频人工动作，靠本清单 + 第 9 节验收命令兜底更便宜。另外：`Not=A?Brand` 的 `.full_version` 保持 `"99"`（与 `.version` 相同）是真实 Chrome 对该 GREASE 品牌的行为，不是漏改。

> 版本与真实值对不上时去查：`https://headers.depar.ch/microsoft-edge`（可查各版本的 `Sec-Ch-Ua*` 真实取值）。

---

## 3. 修改文件总览（7 个文件）

| # | 文件 | 逻辑改动点 | 增/删 | 作用域 |
|---|------|---|---|---|
| 1 | `src/Config.zig` | 5 处 | 8 / 8 | 默认 UA、Client-Hint brands、`default_locale`、`validateUserAgent` 行为、CLI 错误提示文案 |
| 2 | `src/help.zon` | 3 条文本 | 4 / 5 | `--user-agent` / `--user-agent-suffix` / `--locale` 帮助文本 |
| 3 | `src/network/HttpClient.zig` | 1 处 | 16 / 3 | `baselineHeaders()`：新增 5 个 Client-Hint，且不再把 `Sec-Ch-Ua` 系头标为不可覆盖 |
| 4 | `src/browser/webapi/Navigator.zig` | 7 行取值 | 7 / 13 | `navigator.*` 取值对齐真实 Edge/Windows |
| 5 | `src/browser/webapi/NavigatorUAData.zig` | 4 处 + 1 个新函数 | 26 / 14 | `navigator.userAgentData` 的 `platform`、高熵值、低熵 `brands` 取短版本号 |
| 6 | `src/browser/webapi/PluginArray.zig` | 1 行 | 1 / 1 | `navigator.plugins.length` 非零 |
| 7 | `src/server/cdp/domains/emulation.zig` | 删 1 行（+3 行注释） | 3 / 1 | `Emulation.setUserAgentOverride` 不再因 UA 含 Mozilla 而忽略 |

合计 **65 增 / 45 删**。全部是生产行为改动，没有一处测试代码改动。

> 「逻辑改动点」不等于 `git diff` 的 hunk 数（相邻改动会被合并成一个 hunk，例如 Navigator 7 行取值只构成 4 个 hunk），核对时以第 4 节的逐项列表为准。

---

## 4. 逐文件改动详情

### 4.1 `src/Config.zig`

**(a) 默认 UA**

```zig
// 上游
const user_agent_base: [:0]const u8 = "Lightpanda/1.0";
// 本分支
const user_agent_base: [:0]const u8 = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
```

**(b) Client-Hint brands（品牌名、顺序、版本号都对齐真实 Edge 151）**

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

- 品牌名是 `Not=A?Brand`（等号 + 问号），不是 `Not-A.Brand`；三项顺序不能变（GREASE 品牌在前是 Chrome 的固定行为）。
- `.version`（短版本号）喂 `Sec-Ch-Ua` 与 `navigator.userAgentData.brands`；`.full_version` 喂 `Sec-Ch-Ua-Full-Version-List` 与 `fullVersionList`。两者不得混用，见 4.5。
- `full_version` 必须写死字面量，不能引用 `lp.build_config.version`——那是 Lightpanda 自己的版本号，出现在 client hint 里等于自报家门。
- `sec_ch_ua` / `sec_ch_ua_full_version_list` 两个 comptime 拼接常量保持上游原样，它们从 `brands` 派生，改了 `brands` 就自动跟着变。

**(c) `default_locale`：语言人设的唯一开关（直接用上游 `--locale` 机制）**

```zig
// 上游
const default_locale: [:0]const u8 = "en-US";
// 本分支
// This branch defaults to zh-CN: it drives Accept-Language, navigator.language
// and navigator.languages together, exactly like a real Chrome would.
const default_locale: [:0]const u8 = "zh-CN";
```

一个标签经 `HttpHeaders.acceptLanguageFor()` 推导后同时驱动四处：

| 作用点 | `zh-CN` 的结果 |
|---|---|
| `Accept-Language` 头 | `zh-CN,zh;q=0.9,en;q=0.8` |
| `navigator.languages` | `[zh-CN, zh, en]`（`Client.getLanguages()` 从 `AcceptLanguage.languages` 切出） |
| `navigator.language` | `zh-CN`（= `languages[0]`） |
| Intl / `toLocaleString` | `zh-CN` |

好处：头与 JS 侧从同一源推导，反爬做交叉比对时不会露馅；且**运行时可切换人设而不改代码**——`--locale en-US`、`--locale ja-JP`，或 CDP `Emulation.setUserAgentOverride({ acceptLanguage })` 都会同时刷新头与 JS。

**(d) `validateUserAgent`：删掉 Mozilla 拒绝逻辑，只保留非打印字符检测**

```zig
// 上游
pub fn validateUserAgent(ua: []const u8) !void {
    for (ua) |c| {
        if (!std.ascii.isPrint(c)) {
            return error.NonPrintable;
        }
    }

    if (std.ascii.indexOfIgnoreCase(ua, "mozilla") != null) {
        return error.Reserved;
    }
}
// 本分支：只删最后那个 if 块
pub fn validateUserAgent(ua: []const u8) !void {
    for (ua) |c| {
        if (!std.ascii.isPrint(c)) {
            return error.NonPrintable;
        }
    }
}
```

**调用方现状（同步时必须一起检查，共 3 处生产调用点）：**

| 调用点 | 写法 | 结果 |
|---|---|---|
| `Transfer.verifyHeader`（`src/network/HttpClient.zig`） | `catch \|err\| { log.warn; 丢弃 }` | 错误集变窄不影响编译；含 Mozilla 的 UA 通过 |
| `Network.setExtraHTTPHeaders`（`src/server/cdp/domains/network.zig`） | 同上 | 同上 |
| `Emulation.setUserAgentOverride`（`src/server/cdp/domains/emulation.zig`，见 4.7） | **对错误集做穷尽 `switch`** | 必须同步删掉 `error.Reserved` prong，否则编译失败 |

**(e) `userAgentValidator` 的 CLI 报错提示文案**

```zig
// 上游
log.fatal(.app, "invalid user-agent", .{ .err = err, .hint = "must be printable ASCII and can't contain Mozilla" });
// 本分支
log.fatal(.app, "invalid user-agent", .{ .err = err, .hint = "must be printable ASCII" });
```

### 4.2 `src/help.zon`

```
--user-agent <STRING>
  上游:   Override the User-Agent header entirely. Must not impersonate other
          browsers; any value containing "Mozilla" is forbidden. The browser
          still sends Sec-Ch-Ua. Incompatible with --user-agent-suffix.
  本分支: Override the User-Agent header entirely. The browser still sends
          Sec-Ch-Ua. Incompatible with --user-agent-suffix.

--user-agent-suffix <STRING>
  上游:   Suffix appended to the Lightpanda/X.Y User-Agent.
  本分支: Suffix appended to the default User-Agent.

--locale <TAG>
  上游:   ... Defaults to en-US.
  本分支: ... Defaults to zh-CN.
```

`--locale` 那一句必须与 4.1(c) 同步改：上游把它写死成字面量（没用 `{1s}` 占位符机制），不改就会帮助文本与实际默认值不一致。

### 4.3 `src/network/HttpClient.zig` — `baselineHeaders()`

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

// 本分支
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

要点：

- `[4]` → `[9]`，新增 5 个 Client-Hint；值只写在这里（`Config.zig` 里不再有对应常量）。
- 两个 `Sec-Ch-Ua` 系头**去掉 `.source = .fixed`**。上游加 `.fixed` 是为了保护自己的"永不接受 Mozilla UA"；对本分支反而是障碍：CDP 客户端（如 `Network.setExtraHTTPHeaders`）带 `Sec-Ch-Ua` 下发时，`Transfer.setHeader` 命中 `.fixed` 分支 → **每个请求刷一条 `ignore overriding fixed header` warn，并且客户端的值被静默丢弃**。去掉后行为与真实浏览器一致：CDP 层能覆盖 baseline。`HeaderSource` 默认 `.user_agent`（最低优先级），任何上层来源都能覆盖它。
- `Accept-Language` 用上游的 `self.getAcceptLanguage()`，不要用已废弃的 `HttpHeaders.accept_language` 常量，否则 `--locale` 与 CDP `acceptLanguage` 不会作用于真实请求头。
- 上游至今保留 `.fixed`，**每次同步都要检查这两行有没有被加回来**。

### 4.4 `src/browser/webapi/Navigator.zig`

7 处单行取值改动：

| 函数 | 上游 | 本分支 |
|---|---|---|
| `getDoNotTrack` | `null` | `"1"` |
| `getAppVersion` | `"1.0"` | `"5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0"`（= UA 去掉 `Mozilla/` 前缀，反爬会与 `navigator.userAgent` 交叉比对） |
| `getHardwareConcurrency` | `4` | `32` |
| `getDeviceMemory` | `8.0` | `32`（Chrome 返回整数形式的 GB 数） |
| `getMaxTouchPoints` | `0` | `10`（`0` 是明显的无触屏 / headless 特征） |
| `getVendor` | `""` | `"Google Inc."`（真实 Chrome 固定值） |
| `getPlatform` | 按 `builtin.os.tag` 分支（Linux 上为 `Linux x86_64`） | 固定 `"Win32"`（对应 UA 的 `Windows NT 10.0; Win64; x64`；Chrome 在 64 位系统上也返回 `Win32`，不是 `Win64`） |

**除此之外一律保持上游原样，特别注意：**

- `getLanguages` / `getLanguage` **不改**：用上游实现（从 `http_client.getLanguages()` 取），语言人设由 4.1(c) 的 `default_locale` 驱动。
- 可见性随上游：`getDoNotTrack`、`getCookieEnabled`、`getMaxTouchPoints`、`getWebdriver`、`javaEnabled`、`sendBeacon`、`getPlugins`、`getPermissions`、`getGeolocation`、`getStorage`、`getUserAgentData`、`register/unregisterProtocolHandler` 上游已是 `fn`，本分支不得改回 `pub`（它们只被同文件 `JsApi` 引用）。
- 改完 `getPlatform` 后，文件顶部 `const builtin = @import("builtin");` 在本文件不再被引用；Zig 对容器级未使用声明不报错，**保留上游那行，不要删**。
- `getAppName`（`Netscape`）、`getAppCodeName`（`Mozilla`）、`getProduct`（`Gecko`）、`getGlobalPrivacyControl`（`false`）、`getUserAgent`（走 `http_client`）保持上游实现。

### 4.5 `src/browser/webapi/NavigatorUAData.zig`

**(a) `uaPlatform()` 固定 `"Windows"`**

```zig
// 上游：按 builtin.os.tag 分支（Linux 上返回 "Linux"）
fn uaPlatform() []const u8 {
    return "Windows";
}
```

**(b) 低熵 brands 必须用短版本号（保真度关键）**

上游只有一个 `brandList()`，内部固定取 `b.full_version`。直接照抄的后果：`navigator.userAgentData.brands`（真实 Chrome/Edge 这里必须是主版本号 `151`）会带上 `151.0.7813.2`，与 HTTP 头 `Sec-Ch-Ua`（用 `.version` = `151`）不一致——任何把两者交叉比对的反爬立刻判定为伪造客户端。

做法（遵守第 1 节原则：上游函数一字不动，只加一个）：

```zig
// 上游代码，保持原样：高熵用（full_version）
fn brandList() []const Brand { ... arr[i] = .{ .brand = b.brand, .version = b.full_version }; ... }

// 本分支新增的唯一函数：低熵用（version），必须与 Config.HttpHeaders.sec_ch_ua 逐项一致
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

**(c) 三处低熵调用点改为 `shortBrandList()` + 高熵值写死**

```zig
fn getBrands(_: *const NavigatorUAData) []const Brand {
    return shortBrandList();                      // fork: 原为 brandList()
}

pub fn toJSON(_: *const NavigatorUAData) struct { ... } {
    return .{
        .mobile = false,
        .brands = shortBrandList(),               // fork: 原为 brandList()
        .platform = uaPlatform(),
    };
}

fn getHighEntropyValues(_: *const NavigatorUAData, hints: []const []const u8, exec: *const Execution) !js.Promise {
    _ = hints;

    const brands = brandList();                   // 上游行，保留（下面 .fullVersionList 用）

    return exec.js.local.?.resolvePromise(.{
        .brands = shortBrandList(),               // fork: 原为 brandList()
        .mobile = false,
        .platform = uaPlatform(),
        .architecture = "x86",                    // fork: 原为 uaArchitecture()
        .bitness = "64",                          // fork: 原为 uaBitness()
        .model = "",
        .platformVersion = "15.0.0",              // fork: 原为 ""
        .uaFullVersion = "151.0.7813.2",          // fork: 原为 if (brands.len > 0) brands[0].version else "1.0.0.0"
        .fullVersionList = brands,                // 上游行，未改
        .wow64 = false,
        .formFactor = [_][]const u8{"Desktop"},
    });
}
```

- `architecture` / `bitness` 必须写死：上游的 `uaArchitecture()` / `uaBitness()` 读 `builtin.cpu.arch`，一旦在 ARM 机器上编译就会输出 `arm` / `32`，与 Windows x64 人设直接矛盾。
- 上游的 `uaFullVersion = brands[0].version` 取的是 brand 列表**第一项**，对本分支就是 `Not=A?Brand` 的 `"99"`（垃圾值），必须覆盖。
- `uaArchitecture()` / `uaBitness()` 在本分支变成未被引用的上游函数，**不要删**（增加 diff；Zig 不报错）。
- 可见性随上游：`getBrands` / `getMobile` / `getPlatform` / `getHighEntropyValues` 是 `fn`，只有 `toJSON` 是 `pub fn`。

### 4.6 `src/browser/webapi/PluginArray.zig`

```zig
// 上游
pub const length = bridge.property(0, .{ .template = false });
// 本分支
pub const length = bridge.property(5, .{ .template = false });
```

真实 Chrome 默认有 5 个内置插件（PDF Viewer 等）；仅长度非 0 就能过掉只看 `navigator.plugins.length` 的检测。底层数据源未改，`[...navigator.plugins]` 迭代仍为空数组，`item()` / `namedItem()` 仍返回 `undefined`——检测走到这一步才会露馅，目前不需要处理。
上游把 `item` / `[str]` 的 bridge 包装改写成内联 `wrap` 函数（`5b2279ba1`），与本行无交叉，自动合并即可。

### 4.7 `src/server/cdp/domains/emulation.zig` — `setUserAgentOverride`

只删一行 `error.Reserved => true,`（对应 4.1(d)），其余上游代码一字不改：

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

后面紧跟的上游逻辑全部保留（`acceptLanguage` 处理、`if (reserved)` 块、`setUserAgentOverride`）：

```zig
if (params.acceptLanguage) |accept_language| {
    if (!Mime.isHttpHeaderValue(accept_language)) {
        return cmd.sendError(-32602, "Accept-Language contains CR, LF or NUL", .{});
    }
    try http_client.setAcceptLanguageOverride(accept_language);
}

if (reserved) { /* 本分支永不进入，按第 1 节原则原样保留 */ }

try http_client.setUserAgentOverride(ua);
bc.user_agent_changed = true;
```

说明：

- 本分支的核心目的之一：CDP 传入含 Mozilla 的 UA 必须被接受并生效，不能再静默忽略。
- 错误集收窄后 `switch (err)` 只剩 `error.NonPrintable` 就是穷尽的（推断为 `error{NonPrintable}`）。若上游以后给 `validateUserAgent` 加新错误，这里会报「switch 未穷尽」，补上新错误的 prong 即可。
- 上游新增的 `acceptLanguage` 处理必须收下（与 4.1(c) 配套）：它让 CDP/Playwright 下发的 `acceptLanguage` 同时覆盖 `Accept-Language` 头与 `navigator.languages`。上游注释"Applied even when the user agent is refused"在本分支变成无条件适用（UA 根本不会被拒）。

---

## 5. 测试代码策略

**本分支不维护任何测试代码的适配。** 上游测试断言的是上游默认行为，本分支故意改了生产行为，因此下列测试必然失败——这是预期结果，**不修、不删、不改**（`test {}` 块不参与 `make build`，不影响生产二进制）：

| 失败测试 | 原因 |
|---|---|
| `Config: validateUserAgent` | 断言 `expectError(error.Reserved, validateUserAgent("mozilla/1.0"))` |
| `Config: parseArgs refuses a mozilla user-agent` | 断言 `--user-agent mozilla/1.0` 失败 |
| `Config: locale drives http_headers` | 断言默认 `Accept-Language` 为 `en-US,en;q=0.9` |
| `cdp.Emulation: setUserAgentOverride ignores mozilla` / `... case insensitive` | 断言含 Mozilla 的 UA 被忽略 |
| `cdp.Emulation: setUserAgentOverride acceptLanguage drives navigator.languages` | 第一句断言默认 `navigator.language === 'en-US'`；后续 `acceptLanguage` 覆盖部分能过 |
| `cdp.network setExtraHTTPHeaders rejects a Mozilla User-Agent` / `... smuggled via a colon in the key` | 断言拒绝 Mozilla UA |
| `WebApi: Navigator` 系列 | 断言 `vendor === ''`、`platform` 跟编译机、`doNotTrack === null`、`hardwareConcurrency=4`、`deviceMemory=8`、`maxTouchPoints=0`、languages 默认值 |
| `WebApi: NavigatorUAData` 系列 | 断言 `brands` 为 `Lightpanda` / 全版本号、`platform` 跟编译机 |

> 曾经犯过的错：把这类测试"反向改写"成断言接受 Mozilla UA，使测试变绿但污染了测试代码——已全部回退。本分支永远不改测试；`make test` 不作为验收步骤。

---

## 6. 代码层解决不了 / 未处理的限制

1. **TLS 指纹（JA3/JA4）**：curl 链接的是 **BoringSSL**（`build.zig.zon` 的 `boringssl-zig` 依赖，`build.zig` 里 `CURL_DEFAULT_SSL_BACKEND = "openssl"` 走 OpenSSL 兼容后端），比通用 OpenSSL 更接近 Chrome 的握手段，但 ClientHello 的扩展顺序、GREASE、压缩方法等仍与真实 Chrome 不同。要完全一致需要 curl-impersonate 级别的改造。
2. **ALPN 失败更敏感**：BoringSSL 在 ALPN 未协商成功时比 OpenSSL 严格，个别站点点开会报 SSL 连接错误；可先用 `--http-version 1.1` 退化到 HTTP/1.1 绕开（`--http-version` 只接受 `auto` / `1.1`）。
3. **HTTP/2 指纹**：SETTINGS 帧参数、WINDOW_UPDATE、伪头顺序与真实浏览器不同。
4. **Canvas / WebGL 指纹**：headless 无 GPU，Canvas 为软件渲染特征。
5. **WebRTC**：headless 下无 WebRTC 或返回异常 IP。
6. **`--http-header "Sec-Ch-Ua: ..."` 被 CLI 层硬性拒绝**（`Config.zig` 里 `hint = "Sec-Ch-Ua is not overridable"`，上游既有行为）。**按本分支「所有线路按上游走」的原则：不改动**——真实浏览器不存在这个入口，屏蔽它不影响伪装效果，放开只会多一处 fork 冲突面。精确范围：
   - 仅拦 `Sec-Ch-Ua` 这一个名字（`eqlIgnoreCase` 是**全等**匹配，不是前缀）；`Sec-Ch-Ua-Full-Version-List`、`Sec-Ch-Ua-Platform`、`-Mobile`、`-Arch`、`-Bitness`、`-WoW64` **都可以**用 `--http-header` 覆盖 baseline。
   - CDP 路径下 `Sec-Ch-Ua` **已实测可覆盖**（4.3 拆掉 `.source = .fixed` 的结果），但**前提是先发 `Network.enable`**：`extra_headers` 是在 `Network.httpRequestStart` 里应
     用的，而该回调由 `Network.enable → bc.networkEnable()` 注册（`notification.register(.http_request_start, …)`）；不 enable 就没人订阅该通知，
     `setExtraHTTPHeaders` 会**静默无效**（无报错、无 warn）。
   - 验收命令见第 9 节第 6 步（能直接看到拒绝与放行的分界）。
7. **`navigator.plugins` 内容仍为空**（只改了 `length`，见 4.6）。
8. **CDP 无法在运行时改 Intl/Date 的 locale/timezone**：上游 `Emulation.setLocaleOverride` / `setTimezoneOverride` 是 noop（需要 zig-v8-fork 暴露 `Isolate::DateTimeConfigurationChangeNotification` 与 ICU 默认 locale 绑定），要改只能靠进程启动参数 `--locale` / `--timezone`。
9. **`window.chrome`、`navigator.getBattery`、`navigator.getUserMedia`、`window.outerHeight/outerWidth` 未实现**（上游 API 缺失）：bot.sannysoft 上表现为 `Chrome (New) missing (failed)`、`HEADCHR_CHROME_OBJ FAIL`、`CHR_BATTERY FAIL`，creepjs 上报 `ReferenceError: outerHeight is not defined`。要补需你决策（`window.chrome` 是个很小的空对象桩，getBattery 要额外实现 Promise 对象）。
10. **文档请求缺 `Sec-Fetch-Dest/Mode/Site/User`、`Upgrade-Insecure-Requests`、`Priority`**（全仓 grep 零命中，上游从未发过），且 `Accept-Encoding` 为 `deflate, gzip, br`（真实 Edge 是 `gzip, deflate, br, zstd`）——现代反爬很常用这一组做交叉校验。
11. **换 UA 人设时 client hints 不跟着变**：`--user-agent` 只改 UA（实测：UA 变 Chrome/120 而 `Sec-Ch-Ua` 仍是 151），`--http-header` 又拦住 `Sec-Ch-Ua`（本分支不改，见上面第 6 条）→ 真要换浏览器版本人设，要么走 CDP（`Network.enable` + `setExtraHTTPHeaders`，已实测可覆盖），要么同时改代码里的 `user_agent_base` + `brands`。
12. 小噪声（非问题）：httpbin 把请求头名字按首字母大写重新格式化，回显成 `Sec-Ch-Ua-Wow64`；实际发出的是 `Sec-Ch-Ua-WoW64`（`baselineHeaders` 可证）。

---

## 7. 环境与编译

全新机器 / 换新服务器：按 7.1 → 7.8 顺序完整走一遍（或直接跑 7.8 的一键脚本）。日常同步上游：只需复查 7.4（V8 tag）与 7.6（依赖）。

### 7.1 系统依赖（Ubuntu 22.04+ / Debian 12+）

```bash
apt-get update && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config \
    libglib2.0-dev clang make curl git
```

### 7.2 Zig（版本取自 `build.zig.zon` 的 `minimum_zig_version`）

```bash
ZIG_VERSION="$(grep -oP '(?<=minimum_zig_version = ")[^"]+' build.zig.zon)"
curl -LO https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz
tar xf zig-x86_64-linux-${ZIG_VERSION}.tar.xz
rm -rf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}
mv zig-x86_64-linux-${ZIG_VERSION} /usr/local/lib
ln -sf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}/zig /usr/local/bin/zig
rm zig-x86_64-linux-${ZIG_VERSION}.tar.xz
zig version
```

### 7.3 Rust（国内镜像 rsproxy.cn）

```bash
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y
source $HOME/.cargo/env

mkdir -p ~/.cargo
cat > ~/.cargo/config.toml << 'CARGO_EOF'
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

把 `RUSTUP_DIST_SERVER` / `RUSTUP_UPDATE_ROOT` 一并写进 `~/.bashrc`。

### 7.4 预编译 V8 库（约 127 MB）

```bash
make download-v8
```

版本与路径都由仓库派生，不要手抄：`Makefile` 从 `.github/actions/install/action.yml` 读 `zig-v8`（release tag）与 `v8`（版本号），组成
`V8_CACHE := .lp-cache/prebuilt-v8/<tag>/libc_v8_<v8>_<os>_<arch>.a`；`build.zig` 里的 `findPrebuiltV8()` 用同一套规则自动发现它，所以**正常情况不需要传 `-Dprebuilt_v8_path`**。

两个必须知道的坑：

1. **tag 升级时资源文件名可能不变但字节不同**（文件名只编 V8 版本），所以缓存路径里的 `<tag>` 一层不能省；链接期报 undefined `v8__*` 符号基本就是缓存里躺着旧 tag 的 `.a`。每次同步上游后要重跑 `make download-v8`。
2. **`download-v8` 有 `test -f` 守卫**：中途被杀掉的半截文件会被当成已就绪。用镜像手动下载时先落 `.part` 再 `mv -f` 覆盖：

```bash
# GitHub 直连超时时走镜像（tag / v8 版本从 action.yml 取，不写死）
TAG=$(awk -F\' '/^  zig-v8:/{f=1} f&&/default:/{print $2; exit}' .github/actions/install/action.yml)
V8=$(awk -F\' '/^  v8:/{f=1} f&&/default:/{print $2; exit}' .github/actions/install/action.yml)
DIR=.lp-cache/prebuilt-v8/$TAG; F=libc_v8_${V8}_linux_x86_64.a
mkdir -p $DIR
curl -fL --retry 3 --no-progress-meter -o $DIR/$F.part \
  "https://ghfast.top/https://github.com/lightpanda-io/zig-v8-fork/releases/download/$TAG/$F"
ls -l $DIR/$F.part          # 字节数要与 curl -sIL 返回的 content-length 一致
mv -f $DIR/$F.part $DIR/$F
```

> 没有缓存且没传 `-Dprebuilt_v8_path` 时，`build.zig` 会打印 `No prebuilt V8 at ...; using the V8 source-build path` 并从源码编 V8（10+ 分钟）。

### 7.5 环境变量

```bash
export PATH="/usr/local/bin:$HOME/.cargo/bin:$HOME/.local/bin:$PATH"
export LIGHTPANDA_DISABLE_TELEMETRY=1
# Agent 模式按需：export GEMINI_API_KEY / GOOGLE_API_KEY
```

### 7.6 依赖离线抓取（大陆网络：全新机器与每次同步上游后都要做）

本机直连 GitHub HTTPS 会超时（`HttpConnectionClosing` / `Timeout`），而 **SSH 到 GitHub 可用**。镜像/SSH 取回的字节与原站一致 → `zig fetch` 算出的 hash 与 `build.zig.zon` 精确匹配 → 落全局缓存 `~/.cache/zig/p/`，`build.zig.zon` 一行都不用改。

要抓哪些**不凭记忆也不照抄本文档**，直接从仓库读；同步上游后用 diff 判断哪些项变了：

```bash
# 0) 哪些依赖变了（对比上次同步的基线）
git diff <旧基线>..HEAD -- build.zig.zon

# 1) tarball 类（.url = https://github.com/...tar.gz）——逐个加镜像前缀重抓
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep -v '^git+'); do
  timeout 600 zig fetch "https://ghfast.top/$u" || echo "FAILED: $u"
done

# 2) git+https 类——用 GIT_CONFIG 临时把 https 改写成 SSH，不动全局 git 配置
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep '^git+https'); do
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0="url.git@github.com:.insteadOf" GIT_CONFIG_VALUE_0="https://github.com/" \
    timeout 600 zig fetch "$u" || echo "FAILED: $u"
done

# 3) 预编译 V8 .a（裸二进制，不是 zig 包）走镜像，见 7.4

# 4) 全部抓完后验证：EXIT=0 且无输出 = 完全离线可解析
zig build --fetch
```

> 校验：每条 `zig fetch` 输出的 hash 必须与 `build.zig.zon` 里对应的 `.hash` 完全一致，否则字节不同（镜像失效/被篡改）。
> 备选镜像：`https://gh-proxy.com/`（与 `ghfast.top` 等效，可互换）。
> **注意**：即使不传 `-Dprebuilt_v8_path`，`.v8` 依赖 tarball（Zig/C 绑定源码）仍必须能解析，预编译 `.a` 不能替代它。

### 7.7 编译命令

```bash
# 生产（release + V8 snapshot）。编译机是 AMD EPYC 9T25（Zen 5），
# 直接 make build 会带 Zen 5 特有指令，在 Intel Skylake-SP 生产机上运行会 SIGILL。
ZIGFLAGS="-Dcpu=skylake_avx512" make build

# 需要兼容更老的 CPU 时
ZIGFLAGS="-Dcpu=baseline" make build

# debug 版
make build-dev

# 清理（保留 .lp-cache 里的 V8 缓存）
make clean
```

- 预编译 V8 由 `build.zig` 自动从 `.lp-cache` 发现（见 7.4），**不需要再传 `-Dprebuilt_v8_path`**；只有缓存放在仓库外时才需要显式指定。编译输出里看到 `Using prebuilt V8: ...` 才算用上了缓存，看到 `No prebuilt V8 at ...` 说明在从源码编 V8。
- `ZIGFLAGS` 统一用**环境变量形式**（`ZIGFLAGS="..." make build`），这也是上游 `Makefile` 自己给的写法（`# ZIGFLAGS=-Ddev_fast=false make test`）。上游删掉 `MAKEOVERRIDES =` 后，命令行形式 `make build ZIGFLAGS="-Dcpu=... ..."` 在 GNU Make 4.3 上实测**同样可用**（子 make 收到的是 `MAKEFLAGS=[... -- ZIGFLAGS=-Dcpu=skylake_avx512\ -Dprebuilt_v8_path=x.a]`，空格被转义成单个赋值，不会碎成 make 选项），所以这不是兼容性问题，只是写法统一。
- `Makefile` 与 `build.zig` **本分支保持与上游零 diff**（第 1 节原则：所有线路按上游走），不要为了 ZIGFLAGS 去恢复 `MAKEOVERRIDES =`。
- C/Rust 依赖不跟随 `-Doptimize`，固定编成 ReleaseFast（`build.zig` 的 `-Ddebug_deps` 才会转 Debug），debug/release 共用一套依赖对象缓存：首次编译会重建全部依赖，之后 `make build-dev` 明显更快。
- 产物：`./zig-out/bin/lightpanda`；安装位置 `~/.local/bin/lightpanda`；V8 snapshot `src/snapshot.bin`；V8 缓存 `.lp-cache/prebuilt-v8/<tag>/`。
- 格式化必须与 CI 一致：`zig fmt --check ./*.zig ./**/*.zig`（`zig build` 依赖 fmt step，会顺带检查）。

### 7.8 一键初始化（新服务器）

```bash
#!/bin/bash
set -euo pipefail
PROJECT_DIR="${1:-$(pwd)}"

echo "==> [1/6] 系统依赖"
apt-get update -qq && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config libglib2.0-dev clang make curl git

echo "==> [2/6] Zig（版本取自 build.zig.zon）"
ZIG_VERSION="$(grep -oP '(?<=minimum_zig_version = ")[^"]+' "$PROJECT_DIR/build.zig.zon")"
cd /tmp
curl -fsSLO https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz
tar xf zig-x86_64-linux-${ZIG_VERSION}.tar.xz
rm -rf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}
mv zig-x86_64-linux-${ZIG_VERSION} /usr/local/lib
ln -sf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}/zig /usr/local/bin/zig
rm -f zig-x86_64-linux-${ZIG_VERSION}.tar.xz
zig version

echo "==> [3/6] Rust（rsproxy 镜像）"
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y
export PATH="$HOME/.cargo/bin:$PATH"
mkdir -p ~/.cargo
cat > ~/.cargo/config.toml << 'CARGO_EOF'
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

echo "==> [4/6] 依赖：V8 预编译库 + zig 包离线抓取"
cd "$PROJECT_DIR"
make download-v8 || true   # 直连失败就走 7.4 的镜像法
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep -v '^git+'); do
  zig fetch "https://ghfast.top/$u" || echo "FAILED: $u"
done
for u in $(grep -oP '(?<=\.url = ")[^"]+' build.zig.zon | grep '^git+https'); do
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0="url.git@github.com:.insteadOf" GIT_CONFIG_VALUE_0="https://github.com/" \
    zig fetch "$u" || echo "FAILED: $u"
done
zig build --fetch          # 应 EXIT=0 且无输出

echo "==> [5/6] 编译（Skylake-SP 兼容）"
export LIGHTPANDA_DISABLE_TELEMETRY=1
ZIGFLAGS="-Dcpu=skylake_avx512" make build

echo "==> [6/6] 安装"
mkdir -p "$HOME/.local/bin"
cp -f zig-out/bin/lightpanda "$HOME/.local/bin/lightpanda"
echo "完成：lightpanda serve --host 0.0.0.0 --port 9222"
```

用法：`chmod +x init.sh && sudo ./init.sh /path/to/browser`

### 7.9 常见问题

| 现象 | 处理 |
|---|---|
| html5ever / Rust 依赖编译失败 | `cargo --version`；确认 7.3 的镜像配置 |
| `zig version` 与 `build.zig.zon` 的 `minimum_zig_version` 不符 | 按 7.2 重装 |
| V8 下载失败 / 只有半截文件 | 按 7.4 的镜像法（`.part` + `mv -f`） |
| `zig build` 卡在依赖解析 / `HttpConnectionClosing` | 按 7.6 离线抓取，然后 `zig build --fetch` 验证 |
| 输出里没有 `Using prebuilt V8: ...` | 缓存不在 `action.yml` 所写 tag 的目录里：重跑 `make download-v8` |
| 链接期报 undefined `v8__*` 符号 | 缓存里是旧 tag 的 `.a`；重跑 `make download-v8`（缓存路径已按 tag 分目录） |
| 生产机运行时 `SIGILL` | 编译时没加 `-Dcpu=skylake_avx512`（或改用 `-Dcpu=baseline`） |
| 个别站点报 SSL 连接错误 | 见第 6 节 ALPN 条目，先试 `--http-version 1.1` |

---

## 8. Git 分支管理与上游同步 SOP

### 8.1 拓扑

```
官方上游 ──→ lightpanda-io/browser（SSH，只读，只拉不推）= remote `upstream`
                        │
                        ▼
                本机 main 分支（跟踪上游，保持纯净，永远快进）
                        │
                        └── chrome 分支 ─── 第 3、4 节那 7 个生产文件的改动（不碰测试代码）
                                            备份推到 remote `origin`
```

### 8.2 远程配置（一次性；换机器 / 新仓库时要重做）

```bash
git remote rename origin upstream
git remote add origin git@github.com:ewpkw/browser.git
git remote -v   # upstream + origin 都应是指向 github.com 的 SSH 地址
```

> 本机所有 GitHub 远程一律走 SSH；HTTPS 地址不认 SSH key，且大陆直连会超时。

### 8.3 从零重建 chrome 分支（换新服务器 / 换仓库时用）

**一次性但可重放的流程必须保留在本文档里**（与 8.5 的「不写一次性迁移内容」互补：建仓库、配远程、装环境、离线抓依赖这类事每次重新部署都要重做）。换机器时**不需要** cherry-pick 旧 commit：第 4 节就是完整的 fork 源码，直接在上游最新版上重贴一遍，结果等价且没有陈旧的 merge 结构。

```bash
# 1. 克隆上游（一律走 SSH），main 保持纯净
git clone git@github.com:lightpanda-io/browser.git browser
cd browser
git remote rename origin upstream
git remote add origin git@github.com:ewpkw/browser.git     # 备份仓库，见 8.2

# 2. 在当时的 upstream/main 上建分支
git checkout -b chrome

# 3. 照第 4 节 4.1 → 4.7 逐文件贴 fork 改动（共 7 个文件）
#    贴之前先过第 1 节原则 1：上游已有同类开关的，只改默认值，不照搬旧硬编码

# 4. 静态核对（本分支不跑 make test）
git diff upstream/main --stat -- src      # 必须恰好第 3 节那 7 个文件
zig fmt --check ./*.zig ./**/*.zig

# 5. 环境与依赖：按 7.1 → 7.6 装齐

# 6. 编译 + 验收：ZIGFLAGS="-Dcpu=<按生产机>" make build，再跑第 9 节六步

# 7. 首次提交 + 推备份
git add -A
git commit -m "feat(chrome): 伪装成 Chrome/Edge 浏览器

- 默认 UA 改为 Edge 151，validateUserAgent 不再拒绝 Mozilla
- Sec-Ch-Ua brands 改为 Not=A?Brand / Microsoft Edge / Chromium（低熵短版本号）
- baselineHeaders 新增 5 个 Client-Hint，并去掉 Sec-Ch-Ua 系的 .source=.fixed
- navigator.* 对齐真实 Windows Edge，plugins.length=5
- 语言人设走 default_locale=zh-CN（复用上游 --locale 机制，不硬编码）
- 详见 CHROME.md 第 3、4 节"
git push -u origin chrome
```

> 备选搬代码法（仅当旧分支基线已经很旧时方便）：`git checkout <旧chrome 分支> -- src/Config.zig src/help.zon ...` 把 7 个文件拉过来，但**必须逐个与第 4 节核对**，否则会把旧基线的上游代码一并搬进来。
>
> 无论哪条路径，提交前 CHROME.md 必须与代码一致（文档头基线、第 2、3、4 节）：文档与代码不同步比代码有 bug 更贵。

### 8.4 同步流程

```bash
# 0. 干跑一次，先看会撞哪几个文件（不动工作区）
git fetch upstream
git merge-tree --write-tree --name-only chrome upstream/main

# 1. main 快进到上游
git checkout main
git merge --ff-only upstream/main

# 2. chrome 合并 main
git checkout chrome
git merge main

# 3. 解冲突：只会出现在第 3 节那 7 个文件
#    （测试代码本分支不改动，永远与上游一致，天然不冲突、不需要关心）
#      src/Config.zig               UA / brands / default_locale / validateUserAgent / 提示文案
#      src/help.zon                 --user-agent / --user-agent-suffix / --locale
#      src/network/HttpClient.zig   baselineHeaders 去 .fixed + 5 个 Client-Hint
#      src/browser/webapi/Navigator.zig        7 个取值
#      src/browser/webapi/NavigatorUAData.zig  uaPlatform / shortBrandList / 高熵值
#      src/browser/webapi/PluginArray.zig      length=5
#      src/server/cdp/domains/emulation.zig    删 error.Reserved prong
#
#    统一手法（第 1 节原则 2）：
for f in $(git diff --name-only --diff-filter=U); do git checkout --theirs "$f"; done
#    然后照第 4 节把那 7 个文件的 fork 行逐个贴回去，再 git add <file>

# 4. 静态检查（本分支不跑 make test）
zig fmt --check ./*.zig ./**/*.zig
git diff upstream/main --stat -- src      # 必须恰好 7 个文件

# 5. 依赖与环境同步（第 7.4 / 7.6 节）
#    diff 这两个文件：变了就重抓，并按 8.5 回写本文档
git diff <旧基线>..upstream/main -- build.zig.zon .github/actions/install/action.yml
make download-v8        # 直连超时则用 7.4 的镜像法
zig build --fetch       # 确认依赖全部可离线解析

# 6. 编译验收（第 9 节），通过后提交
ZIGFLAGS="-Dcpu=skylake_avx512" make build
cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda
~/.local/bin/lightpanda version
git commit          # merge commit
git push origin chrome

# 7. 回写本文档（按 8.5 的检查项），保证它只描述当前状态，不添一次性迁移内容
```

若上游改动使某处 fork 覆盖变成多余（上游提供了同类开关），**优先删掉 fork 覆盖、改用上游开关**，并把本文档对应条目一并删掉——这是本分支增量不持续膨胀的唯一办法。

### 8.5 每次同步后必须回文档更新的地方

本文档的取舍规则：**只写当前状态与可重放的流程**。一次性迁移内容（本次要重哪个依赖、上次冲突长什么样）一律不写，当场处理完、口头汇报即可；但环境重建时每次都要重做的一次性动作（配远程、建分支、装依赖链、离线抓包）**必须保留**，如 7.1–7.8、8.2、8.3。

| 章节 | 要核什么 |
|---|---|
| 文档头 | `fork 基线` 的 commit 改成新的上游 HEAD |
| 第 2 节 | 人设取值有没有因上游新机制而变动（如 `--locale`、client hint 相关的新开关） |
| 第 3、4 节 | 逐节与实际 `git diff upstream/main -- src` 对一遍；有 fork 改动被上游机制取代，就整节删掉 |
| 第 5 节 | 新增/改名的上游测试因本分支失败的要补上，已不存在的要删 |
| 第 7 节 | 编译/依赖流程本身有没有变（`build.zig` 选项、Makefile 目标、上游目录迁移） |

核对命令（跑完全绿才算同步完成）：

```bash
git diff upstream/main..HEAD --stat -- src   # 恰好第 3 节那 7 个文件
zig fmt --check ./*.zig ./**/*.zig            # 与 CI 一致
zig build --fetch                            # 依赖可离线解析
```

### 8.6 节奏

| 频率 | 操作 |
|---|---|
| 每周 | `git fetch upstream` 看动向，`git merge-tree` 干跑评估冲突 |
| 上游有反爬相关变更（UA / client hint / navigator / locale） | 立即同步，确认 fork 原则未被削弱 |
| 上游大版本 / `build.zig.zon` 变更 | 按 8.4 第 5 步重抓依赖，核对 V8 tag 与 Zig 版本 |

---

## 9. 验收清单

不要求 `make test` 全绿（见第 5 节）。生产验收按下面七步（前六步 CLI，第七步 CDP）；**本节已经过一次完整实跑，基准值已写进 5b 与 C 备注**；第 2、3 两步是必做的锁链校验，2.1 节的三条等式任一不成立就是伪装穿帮。

> URL 是**位置参数**，不是 `--url`（上游已改成 `lightpanda fetch [flags] <url>`，写 `--url` 会报 `unknown argument`）。

```bash
# 1. 编译 + 安装 + 版本
ZIGFLAGS="-Dcpu=skylake_avx512" make build
cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda && lightpanda version

# 2. HTTP 请求头：与第 2 节表格逐项核对
lightpanda fetch --dump html "https://httpbin.org/headers"
#   User-Agent: Mozilla/5.0 ... Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0
#   Sec-Ch-Ua: "Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"
#   Sec-Ch-Ua-Full-Version-List 里是 151.0.7813.2（注意与上一行短版本号不同）
#   Sec-Ch-Ua-Platform: "Windows"  -Mobile: ?0  -Arch: "x86"  -Bitness: "64"  -WoW64: ?0
#   Accept-Language: zh-CN,zh;q=0.9,en;q=0.8

# 3. JS 侧指纹与 HTTP 头锁得住（本分支的核心一致性检查）
#    注意：脚本必须包在显式的 <body> 里。写成 data:text/html,<script>…document.body… 会在 <head>
#    里执行，此时 document.body 为 null，真实浏览器同样报 TypeError（实测踩过）
lightpanda fetch --dump markdown 'data:text/html,<body><script>document.body.textContent=[navigator.language,navigator.languages.join(","),navigator.hardwareConcurrency,navigator.deviceMemory,navigator.maxTouchPoints,navigator.vendor,navigator.platform,navigator.doNotTrack,navigator.userAgentData.brands.map(function(b){return b.brand+";v="+b.version}).join(" | ")].join("\n")</script></body>'
#   期望逐行：zh-CN / zh-CN,zh,en / 32 / 32 / 10 / Google Inc. / Win32 / 1 /
#            Not=A?Brand;v=99 | Microsoft Edge;v=151 | Chromium;v=151
#   brands 必须是短版本号 151（出现 151.0.7813.2 就说明 4.5 的 low-entropy 覆盖丢了）
#   且与第 2 步的 Sec-Ch-Ua 头逐项一致
#   高熵值（uaFullVersion / platformVersion / fullVersionList / architecture / bitness）
#   是异步 Promise，不适合在 data: URL 里抢跑，直接看第 5 步 browserscan / creepjs 的输出

# 4. 语言开关由上游机制驱动（证明没有退回硬编码）
lightpanda fetch --locale en-US --dump html "https://httpbin.org/headers"
#   Accept-Language 应变回 en-US,en;q=0.9

# 5. 反爬特征站复核
#    bot.sannysoft 是同步渲染，直接 dump 就行；browserscan / creepjs 的结果卡片由异步
#    JS + worker + iframe 算，默认参数下 dump 不到判定（实测只能拿到静态外壳），必须开资源加载并等静默
lightpanda fetch --dump html "https://bot.sannysoft.com"
lightpanda fetch --load-resources iframe,worker --wait-until networkidle --wait-ms 15000 --dump markdown "https://www.browserscan.net/bot-detection"
lightpanda fetch --load-resources iframe,worker --wait-until networkidle --wait-ms 15000 --dump markdown "https://abrahamjuliot.github.io/creepjs/"

# 5b. 已实测基准（同一套命令的正常输出长这样，偏离就是回归）
#   browserscan Bot Detection：WebDriver / WebDriver Advance / Selenium / Webdriverio /
#     NightmareJS / PhantomJS / Awesomium / Cef / CefSharp / Coaches / FMiner / Born /
#     Phantomas / Rhino / **Headless Chrome** / **CDP** / **Dev Tool** 共17 项 **全 Normal，0 Abnormal**
#   bot.sannysoft：只余 window.chrome（Chrome New / HEADCHR_CHROME_OBJ）与 getBattery（CHR_BATTERY）
#     三项 failed（上游 API 缺失，见第 6 节第 9 条），其余包括 Plugins Length=5、languages=[zh-CN,zh,en]、
#     PHANTOM_PROPERTIES/SELENIUM_DRIVER/HEADCHR_PLUGINS/HEADCHR_IFRAME 均 ok
#   creepjs：因 getExtentOfChar / outerHeight / mediaDevices 等缺失，评分算不出来（卡片大量 "0% of engine"、
#     "keys (0)"）——拿不到可判读结论，属能力限制，不作为回归依据

# 6. CLI 层 Sec-Ch-Ua 拦截范围（上游行为，本分支不改，只验它没扩大）
lightpanda fetch --http-header 'Sec-Ch-Ua: "Chromium";v="999"' --dump html "https://httpbin.org/headers"
#   预期：直接 fatal 退出，提示 "Sec-Ch-Ua is not overridable"（上游拦截，正常）
lightpanda fetch --http-header 'Sec-Ch-Ua-Platform: "macOS"' --dump html "https://httpbin.org/headers"
#   预期：正常执行，且输出里 Sec-Ch-Ua-Platform 变成 "macOS"
#   → 证明上游的拦截是全等匹配 `Sec-Ch-Ua`，没把我们的 5 个 Client-Hint 一起堵死
```

CDP 侧的验证（本分支两大核心目的只能在这里验）——零依赖，Node ≥ 22 自带 `WebSocket`，无需 npm 安装：

```bash
# 你自己启动服务（本文档不代跑进程）
lightpanda serve --host 127.0.0.1 --port 9222 2>&1 | tee /tmp/lp.log

# 另开一个终端：把下面脚本存为 /tmp/cdp_check.mjs 并执行
cat > /tmp/cdp_check.mjs <<'MJS'
const BASE = process.env.LP ?? "http://127.0.0.1:9222";
const UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
const { webSocketDebuggerUrl } = await (await fetch(`${BASE}/json/version`)).json();
const ws = new WebSocket(webSocketDebuggerUrl);
await new Promise((r) => (ws.onopen = r));
let seq = 0; const pending = new Map();
ws.onmessage = (e) => {
  const msg = JSON.parse(e.data), p = pending.get(msg.id);
  if (p) (pending.delete(msg.id), msg.error ? p.reject(msg.error) : p.resolve(msg.result));
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

// A: 含 Mozilla 的 UA 必须被接受（对应 4.1(d) + 4.7）
await send("Emulation.setUserAgentOverride", { userAgent: UA, acceptLanguage: "en-US,en;q=0.9" }, sid);
console.log("A1 navigator.userAgent =", await evalJs(sid, "navigator.userAgent"));
console.log("A2 navigator.languages =", await evalJs(sid, "JSON.stringify(navigator.languages)"));

// B: 真实请求头跟着变
await send("Page.navigate", { url: "https://httpbin.org/headers" }, sid);
await new Promise((r) => setTimeout(r, 2500));
console.log("B headers =", await evalJs(sid, "document.body.innerText"));

// C: Network.setExtraHTTPHeaders 覆盖 Sec-Ch-Ua（对应 4.3 拆 .fixed）
await send("Network.enable", {}, sid);
await send("Network.setExtraHTTPHeaders", { headers: { "Sec-Ch-Ua": '"Chromium";v="999"', "x-cdp-probe": "42" } }, sid);
await send("Page.navigate", { url: "https://httpbin.org/headers" }, sid);
await new Promise((r) => setTimeout(r, 2500));
console.log("C headers =", await evalJs(sid, "document.body.innerText"));
ws.close();
MJS
node /tmp/cdp_check.mjs
```

期望与判定：

| 输出 | 必须是 | 否则说明 |
|---|---|---|
| `A1` | 完整的 Edge 151 Mozilla UA | 4.7 的 `error.Reserved` prong 没删干净，被静默忽略了 |
| `A2` | `["en-US","en"]` | `acceptLanguage` 没接入 `setAcceptLanguageOverride`（上游新机制未收下） |
| `B` | `User-Agent` 为 A1 那个值、`Accept-Language: en-US,en;q=0.9` | UA/locale 覆盖只作用于 JS、没进真实请求头 |
| `C` | `Sec-Ch-Ua` 变成 `"Chromium";v="999"`，且出现 `X-Cdp-Probe: 42` | 见下方备注 |
| `/tmp/lp.log` | **不得出现** `User agent must not contain Mozilla`、`ignore overriding fixed header` | 同上两条，本分支的核心目的失效 |

**C 的必要条件（实测结论）**：`Network.enable` 必须在 `setExtraHTTPHeaders` 之前发。

上游把 `extra_headers` 的应用放在 `Network.httpRequestStart`（network.zig）里，而这个回调是在 `Network.enable → bc.networkEnable()` 里通过
`notification.register(.http_request_start, …)` 注册的。**没 enable 就没人订阅该通知，`setExtraHTTPHeaders` 静默无效（不报错、不刷 warn）**
——踩过一次，别再删脚本里的 `Network.enable`。实测通过结果：`Sec-Ch-Ua` 变成 `"Chromium";v="999"`、`X-Cdp-Probe: 42` 出现在请求里、`/tmp/lp.log` 零 warn。
