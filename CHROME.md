# Chrome 浏览器伪装修改方案

## 目标

将 Lightpanda 从透明 bot 伪装成正常的 Chrome/Edge 浏览器，绕过常见的反爬检测（Cloudflare、Akamai、DataDome 等）。

## 目标 UA 字符串

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0
```

---

## 一、User-Agent 相关修改（5个文件）

### 1.1 修改默认 User-Agent

**文件**: `src/Config.zig` 第 656 行

```zig
// 修改前
const user_agent_base: [:0]const u8 = "Lightpanda/1.0";

// 修改后
const user_agent_base: [:0]const u8 = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
```

### 1.2 移除 Mozilla 校验

**文件**: `src/Config.zig` 第 841-851 行

`validateUserAgent` 函数中删除 Mozilla 检测逻辑：

```zig
// 修改前
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

// 修改后
pub fn validateUserAgent(ua: []const u8) !void {
    for (ua) |c| {
        if (!std.ascii.isPrint(c)) {
            return error.NonPrintable;
        }
    }
}
```

> 保留非打印字符检测，删除 Mozilla 检测。

### 1.3 修改 CDP Emulation.setUserAgentOverride 中的 Mozilla 拒绝逻辑

**文件**: `src/server/cdp/domains/emulation.zig` 第 157-164 行

```zig
// 修改前
const ua = params.userAgent;
Config.validateUserAgent(ua) catch |err| switch (err) {
    error.NonPrintable => return cmd.sendError(-32602, "User agent contains non-printable characters", .{}),
    error.Reserved => {
        log.warn(.not_implemented, "Emulation.setUserAgentOverride", .{ .param = "userAgent", .value = ua, .info = "User agent must not contain Mozilla" });
        return cmd.sendResult(null, .{});
    },
};

// 修改后
const ua = params.userAgent;
Config.validateUserAgent(ua) catch |err| switch (err) {
    error.NonPrintable => return cmd.sendError(-32602, "User agent contains non-printable characters", .{}),
};
```

### 1.4 将测试用例从「拒绝 Mozilla」反转为「接受 Mozilla」

**文件**: `src/server/cdp/domains/network.zig`

将上游的 Mozilla 拒绝测试反转为正向验证（本分支的核心意义就是允许 Mozilla UA）：
- `test "cdp.network setExtraHTTPHeaders rejects a Mozilla User-Agent"` → 改为 `accepts a Mozilla User-Agent`，期望 `extra_headers.items.len == 1`
- ~~`test "...rejects a Mozilla User-Agent smuggled via a colon in the key"` → 改为 `accepts ...`~~（**2026-09-09 修正**：该反转是错的。上游同期还有一条跟 Mozilla 无关的通用校验——header **名字**必须是合法 HTTP token，冒号不是合法字符，无论值是不是 Mozilla 都会被拒绝。已改回 `rejects a header name smuggling a colon`，期望 `extra_headers.items.len == 0`，并加注释说明拒绝原因与 Mozilla 无关）
- `test "...rejects a header that smuggles CRLF"` → **保留测试但替换载荷**，将 `Mozilla/5.0` 改为 `CustomBot/1.0`（该测试核心是 CRLF 注入防护，与 Mozilla 无关）

**文件**: `src/server/cdp/domains/emulation.zig`

将上游的 2 个 Mozilla 拒绝测试合并为 1 个正向测试：
- 删除: `test "...ignores mozilla"` 和 `test "...ignores mozilla case insensitive"`
- 新增: `test "cdp.Emulation: setUserAgentOverride accepts Mozilla user agent"`，期望 `user_agent_changed == true`

**文件**: `src/Config.zig`

将上游新增的 CLI 层 Mozilla 拒绝测试反转为正向验证：
- `test "Config: parseArgs refuses a mozilla user-agent"` → 改为 `accepts a mozilla user-agent`，期望成功解析 Chrome UA
- `test "Config: validateUserAgent"` → 删除 `error.Reserved` 断言，改为 `try validateUserAgent("mozilla/1.0")` 和 `try validateUserAgent("Mozilla/5.0")`
- `userAgentValidator` 函数中的错误提示从 `"must be printable ASCII and can't contain Mozilla"` 改为 `"must be printable ASCII"`

> **2026-09-09 补充修复**：上面这条测试最初反转时写成了 `config.user_agent.?`（`Config` 结构体没有这个顶层字段，正确访问方式是 `config.userAgent().?`，见 `userAgent()` 方法），导致 `make test` **整体编译失败**（`test` 块不参与 `make build`，所以之前只跑 `make build` 没发现）；同时这个测试直接用了裸的 `std.testing.allocator`，而 `parseArgs` 的 `userAgentValidator` 内部 `allocator.dupe` 出来的内存不会被 `Config.deinit` 释放（`deinit` 只负责 `http_headers`），触发内存泄漏检测失败。已改为跟相邻的 `--http-version`/`collects --http-header` 测试一致的 arena 模式。

**文件**: `src/network/HttpClient.zig`（**2026-09-09 新增**，上游提交 `737f69ee4 http: improve header overwrite/enforcement` 引入，之前合并时漏掉）

`test "HttpClient: Transfer header layering"` 里有一段验证「带 Mozilla 的 UA 会被 `verifyHeader`/`validateUserAgent` 拒绝、不会进入 header 列表」，与本分支核心原则冲突，已反转为：`Mozilla/5.0` 正常进入并覆盖默认 UA（同时删掉对应的 `testing.expectLog(&.{.http})`，因为不再有 "invalid header dropped" 日志）。

**文件**: `src/browser/tests/net/fetch.html` / `src/browser/tests/net/xhr.html`（**2026-09-09 新增**，同一个上游提交 `737f69ee4` 引入，对应 HTML 测试之前也漏改）

`fetch_header_layers` / `xhr_request_headers` 两个脚本分别用 `Headers.set('User-Agent', 'Mozilla/5.0 ...')` 和 `setRequestHeader('Sec-Ch-Ua', '"Chromium";v="140"')` 断言这两个 header 会被静默丢弃（因为上游：Mozilla 无效 + `Sec-Ch-Ua` 是 fixed）。本分支两者都不成立（Mozilla 有效，且见第 4.2 节，`Sec-Ch-Ua` 不再 fixed），已改为断言 author 层设置直接生效覆盖 baseline：`got['user-agent'] === 'Mozilla/5.0 (X11; Linux x86_64)'`，`got['sec-ch-ua'] === '"Chromium";v="140"'`。

对应地，`src/browser/webapi/net/Fetch.zig` 的 `test "WebApi: fetch"` 里 `testing.expectLog(&.{ .http, .http })` 已删除（不再有 2 条 header-dropped 日志），`src/browser/webapi/net/XMLHttpRequest.zig` 的 `test "WebApi: XHR"` 从 `&.{ .http, .http, .http }` 减为 `&.{.http}`（3 条里只有 2 条跟本次改动有关，剩下 1 条来自 xhr.html 里其他不相关的用例）。

### 1.5 更新 help.zon 中的说明

**文件**: `src/help.zon` 第 382-387 行

```
// 修改前
--user-agent <STRING>
    Override the User-Agent header entirely. Must not impersonate other
    browsers; any value containing "Mozilla" is forbidden. The browser
    still sends Sec-Ch-Ua. Incompatible with --user-agent-suffix.
--user-agent-suffix <STRING>
    Suffix appended to the Lightpanda/X.Y User-Agent.

// 修改后
--user-agent <STRING>
    Override the User-Agent header entirely. The browser still sends
    Sec-Ch-Ua. Incompatible with --user-agent-suffix.
--user-agent-suffix <STRING>
    Suffix appended to the default User-Agent.
```

---

## 二、Client Hints (Sec-CH-UA) 修改（1个文件）

### 2.1 修改 brands 数据为 Chrome 品牌

**文件**: `src/Config.zig` HttpHeaders.brands

```zig
// 修改前
pub const brands = [_]Brand{
    .{ .brand = "Lightpanda", .version = "1", .full_version = lp.build_config.version },
};

// 修改后（顺序、品牌名、版本号均与真实 Edge 151 一致）
pub const brands = [_]Brand{
    .{ .brand = "Not=A?Brand", .version = "99", .full_version = "99" },
    .{ .brand = "Microsoft Edge", .version = "151", .full_version = "151.0.7813.2" },
    .{ .brand = "Chromium", .version = "151", .full_version = "151.0.7813.2" },
};
```

> 真实 Edge 151 的 Sec-Ch-Ua：`"Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"`
> **注意**：品牌名是 `Not=A?Brand`（等号和问号），不是 `Not-A.Brand`。顺序也很重要。
> `full_version` 字段用于 `Sec-Ch-Ua-Full-Version-List` header 和 `getHighEntropyValues().fullVersionList`。

这会同时影响：
- HTTP 头 `Sec-Ch-Ua` 的值（通过 `sec_ch_ua` 常量自动生成，使用 `.version`）
- HTTP 头 `Sec-Ch-Ua-Full-Version-List` 的值（通过 `sec_ch_ua_full_version_list` 常量自动生成，使用 `.full_version`）
- `navigator.userAgentData.brands` JS API 返回值（通过 `NavigatorUAData.shortBrandList()` 引用 `brands`，使用 `.version`）
- `navigator.userAgentData.getHighEntropyValues().fullVersionList` 返回值（通过 `NavigatorUAData.fullBrandList()` 引用 `brands`，使用 `.full_version`）

> **2026-09-09 修复的保真度缺陷**：上游 `NavigatorUAData.zig` 里只有一个 `brandList()`，内部统一用 `b.full_version`（上游自身无所谓，因为他们只有一个 `Lightpanda` 品牌，且不关心低熵/高熵区分）。直接照抄后，`navigator.userAgentData.brands`（低熵，真实 Chrome/Edge 应该返回主版本号如 `"151"`）跟 `Sec-Ch-Ua` HTTP 头（用的是短版本号 `v="151"`）对不上，会被任何做交叉校验的反爬系统（对比 HTTP 头和 JS API 的 brand version）发现不一致，直接判定为伪造。已拆成 `shortBrandList()`（给 `getBrands()`/`toJSON().brands`/`getHighEntropyValues().brands` 用，取 `.version`）和 `fullBrandList()`（给 `getHighEntropyValues().fullVersionList` 用，取 `.full_version`），与真实 Chrome/Edge 行为一致。

---

## 三、Navigator 属性修改（2个文件）

### 3.1 修改 navigator.appVersion

**文件**: `src/browser/webapi/Navigator.zig` 第 61-63 行

```zig
// 修改前
pub fn getAppVersion(_: *const Navigator) []const u8 {
    return "1.0";
}

// 修改后（真实 Edge 151 的值，即 UA 去掉开头的 "Mozilla/" 前缀）
pub fn getAppVersion(_: *const Navigator) []const u8 {
    return "5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
}
```

> 反爬检测会对比 `navigator.userAgent` 和 `navigator.appVersion` 的一致性。

### 3.2 修改 navigator.vendor

**文件**: `src/browser/webapi/Navigator.zig` 第 89-91 行

```zig
// 修改前
pub fn getVendor(_: *const Navigator) []const u8 {
    return "";
}

// 修改后
pub fn getVendor(_: *const Navigator) []const u8 {
    return "Google Inc.";
}
```

> Chrome 返回 `"Google Inc."`，空字符串是异常/非 Chrome 浏览器的特征。

### 3.3 修改 navigator.hardwareConcurrency

**文件**: `src/browser/webapi/Navigator.zig` 第 77-79 行

```zig
// 修改前
pub fn getHardwareConcurrency(_: *const Navigator) u32 {
    return 4;
}

// 修改后
pub fn getHardwareConcurrency(_: *const Navigator) u32 {
    return 32;
}
```

### 3.4 修改 navigator.deviceMemory

**文件**: `src/browser/webapi/Navigator.zig` 第 81-83 行

```zig
// 修改前
pub fn getDeviceMemory(_: *const Navigator) f64 {
    return 8.0;
}

// 修改后
pub fn getDeviceMemory(_: *const Navigator) f64 {
    return 32;
}
```

> Chrome 返回的是整数（如 8, 16, 32），表示设备内存 GB 数。

### 3.5 修改 navigator.maxTouchPoints

**文件**: `src/browser/webapi/Navigator.zig` 第 85-87 行

```zig
// 修改前
pub fn getMaxTouchPoints(_: *const Navigator) u32 {
    return 0;
}

// 修改后
pub fn getMaxTouchPoints(_: *const Navigator) u32 {
    return 10;
}
```

> Windows 触屏设备通常返回 10。0 是明显的 headless/无触屏特征。

### 3.6 修改 navigator.doNotTrack

**文件**: `src/browser/webapi/Navigator.zig` 第 49-51 行

```zig
// 修改前
pub fn getDoNotTrack(_: *const Navigator) ?[]const u8 {
    return null;
}

// 修改后
pub fn getDoNotTrack(_: *const Navigator) ?[]const u8 {
    return "1";
}
```

### 3.7 修改 navigator.language 和 navigator.languages

**文件**: `src/browser/webapi/Navigator.zig` 第 65-67 行、第 45-47 行

```zig
// 修改前（language）
pub fn getLanguage(_: *const Navigator) []const u8 {
    return "en-US";
}

// 修改后
pub fn getLanguage(_: *const Navigator) []const u8 {
    return "zh-CN";
}

// 修改前（languages）—— 注意返回类型也需要从 [2] 改为 [3]
pub fn getLanguages(_: *const Navigator) [2][]const u8 {
    return .{ "en-US", "en" };
}

// 修改后
pub fn getLanguages(_: *const Navigator) [3][]const u8 {
    return .{ "zh-CN", "en-US", "en" };
}
```

> **重要**：返回类型从 `[2][]const u8` 变为 `[3][]const u8`，需要同步修改。

> **上游同步联动**：上游新增的 `src/browser/webapi/WorkerNavigator.zig`（WorkerNavigator，来自 #3288）里 `getLanguages` 直接 `return Navigator.getLanguages(...)` 委托，返回类型也写死为 `[2][]const u8`。我们改了 `Navigator.getLanguages` 后，此文件第 49 行必须同步改为 `[3][]const u8`，否则 `snapshot_creator` 编译报 `expected type '[2][]const u8', found '[3][]const u8'`。这是本分支改动引起的必要联动（无法绕开该官方文件，即便改我们自己的签名为 slice 也仍要同步它）。

### 3.8 修改 navigator.plugins.length 返回非零值

**文件**: `src/browser/webapi/PluginArray.zig` 第 64 行

```zig
// 修改前
pub const length = bridge.property(0, .{ .template = false });

// 修改后
pub const length = bridge.property(5, .{ .template = false });
```

> Chrome 默认有 5 个内置插件（PDF Viewer 等）。长度非零即可通过大部分检测。

### 3.9 修改 navigator.platform 固定为 Win32

**文件**: `src/browser/webapi/Navigator.zig` 第 110-118 行

```zig
// 修改前
pub fn getPlatform(_: *const Navigator) []const u8 {
    return switch (builtin.os.tag) {
        .macos => "MacIntel",
        .windows => "Win32",
        .linux => "Linux x86_64",
        .freebsd => "FreeBSD",
        else => "Unknown",
    };
}

// 修改后
pub fn getPlatform(_: *const Navigator) []const u8 {
    return "Win32";
}
```

> 与 UA 字符串中的 `Windows NT 10.0; Win64; x64` 对应。注意：`navigator.platform` 在 Chrome 中返回 `"Win32"`（即使是 64 位系统）。

### 3.10 修改 navigator.userAgentData 中的 platform

**文件**: `src/browser/webapi/NavigatorUAData.zig` 第 93-101 行

```zig
// 修改前
fn uaPlatform() []const u8 {
    return switch (builtin.os.tag) {
        .macos => "macOS",
        .windows => "Windows",
        .linux => "Linux",
        .freebsd => "FreeBSD",
        else => "Unknown",
    };
}

// 修改后
fn uaPlatform() []const u8 {
    return "Windows";
}
```

### 3.11 修改 navigator.userAgentData 高熵值

**文件**: `src/browser/webapi/NavigatorUAData.zig` 第 58-78 行

```zig
// 修改前
pub fn getHighEntropyValues(_: *const NavigatorUAData, hints: []const []const u8, exec: *const Execution) !js.Promise {
    _ = hints;
    return exec.js.local.?.resolvePromise(.{
        .brands = brandList(),
        .mobile = false,
        .platform = uaPlatform(),
        .architecture = uaArchitecture(),
        .bitness = uaBitness(),
        .model = "",
        .platformVersion = "",
        .uaFullVersion = "1.0.0.0",
        .fullVersionList = brandList(),
        .wow64 = false,
        .formFactor = [_][]const u8{"Desktop"},
    });
}

// 修改后（brands 顺序与真实浏览器一致，由 brandList() 从 Config.brands 获取）
pub fn getHighEntropyValues(_: *const NavigatorUAData, hints: []const []const u8, exec: *const Execution) !js.Promise {
    _ = hints;
    return exec.js.local.?.resolvePromise(.{
        .brands = brandList(),
        .mobile = false,
        .platform = uaPlatform(),
        .architecture = "x86",
        .bitness = "64",
        .model = "",
        .platformVersion = "15.0.0",
        .uaFullVersion = "151.0.7813.2",
        .fullVersionList = brandList(),
        .wow64 = false,
        .formFactor = [_][]const u8{"Desktop"},
    });
}
```

> **2026-09-09 修正（重要保真度缺陷，见第二节末尾说明）**：上面的 `brandList()` 实现内部统一取 `b.full_version`，导致低熵的 `.brands` 也返回了完整构建版本号（如 `"151.0.7813.2"`），与 `Sec-Ch-Ua` HTTP 头里的短版本号 `v="151"` 对不上，是会被反爬交叉校验直接识别的不一致。现已拆分为两个函数，`getHighEntropyValues` 内：
>
> ```zig
> .brands = shortBrandList(),        // 取 .version，如 "151"
> .fullVersionList = fullBrandList(), // 取 .full_version，如 "151.0.7813.2"
> ```
>
> 同时 `getBrands()`（给 `navigator.userAgentData.brands`）和 `toJSON().brands` 也都改为调用 `shortBrandList()`。

---

## 四、需要额外添加的 HTTP 头（2个文件）

### 4.1 在 Config.zig 中定义 Client Hint 常量

**文件**: `src/Config.zig` 在 `sec_ch_ua` 定义后添加：

```zig
pub const sec_ch_ua_platform: [:0]const u8 = "Sec-Ch-Ua-Platform: \"Windows\"";
pub const sec_ch_ua_mobile: [:0]const u8 = "Sec-Ch-Ua-Mobile: ?0";
pub const sec_ch_ua_arch: [:0]const u8 = "Sec-Ch-Ua-Arch: \"x86\"";
pub const sec_ch_ua_bitness: [:0]const u8 = "Sec-Ch-Ua-Bitness: \"64\"";
pub const sec_ch_ua_wow64: [:0]const u8 = "Sec-Ch-Ua-WoW64: ?0";
```

> 所有 Client Hint 值统一定义在 Config.zig，避免字符串硬编码分散在多处。

### 4.2 在 HttpClient.zig baselineHeaders 中添加 Client Hints

**文件**: `src/network/HttpClient.zig` `baselineHeaders` 函数

> **架构变更说明**：上游重构了 header 管理，删除了 `http.zig` 中的 `Headers` struct，改为在 `HttpClient.zig` 中通过 `baselineHeaders()` 返回默认 header 数组。Client Hints 现在在此处添加。上游进一步引入了 `Transfer.RequestHeader` 类型（含 `.source` 字段）和 `Sec-Ch-Ua-Full-Version-List` header。

将原来的：
```zig
pub fn baselineHeaders(self: *const Client) [4]Transfer.RequestHeader {
    return .{
        .{ .name = "User-Agent", .value = self.getUserAgent() },
        .{ .name = "Sec-Ch-Ua", .value = lp.Config.HttpHeaders.sec_ch_ua, .source = .fixed },
        .{ .name = "Sec-Ch-Ua-Full-Version-List", .value = lp.Config.HttpHeaders.sec_ch_ua_full_version_list, .source = .fixed },
        .{ .name = "Accept-Language", .value = lp.Config.HttpHeaders.accept_language },
    };
}
```

替换为：
```zig
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
        .{ .name = "Accept-Language", .value = lp.Config.HttpHeaders.accept_language },
        // Client Hints for Chrome fingerprint
        .{ .name = "Sec-Ch-Ua-Platform", .value = "\"Windows\"" },
        .{ .name = "Sec-Ch-Ua-Mobile", .value = "?0" },
        .{ .name = "Sec-Ch-Ua-Arch", .value = "\"x86\"" },
        .{ .name = "Sec-Ch-Ua-Bitness", .value = "\"64\"" },
        .{ .name = "Sec-Ch-Ua-WoW64", .value = "?0" },
    };
}
```

> **注意**：数组大小为 `[9]`。当 Chrome 版本升级时，修改对应值即可。
>
> **2026-09-09 修正**：上述“`Sec-Ch-Ua`/`Sec-Ch-Ua-Full-Version-List` 标记为 `.source = .fixed`，不可通过 CDP 覆盖”的设计不对。这个 `.fixed` 实际是上游为了保护自己“永不接受 Mozilla UA”的设计而强加的防篡改措施，对本分支而言正好相反：我们需要 CDP 客户端能完整控制 client hints，与真实浏览器行为一致。
>
> 历史代码库 `Transfer.putHeader`（`src/network/HttpClient.zig`）对 `.fixed` header 的处理：任何低优先级 source（包括 `.cdp`）尝试覆盖都会直接 `return` 不生效，并打印 `log.warn(.http, "ignore overriding fixed header", .{ .header = hdr.name })`。而 `src/server/cdp/domains/network.zig` 的 `httpRequestStart` 会对**每个 HTTP 请求**都应用一次 `bc.extra_headers`（以 `.source = .cdp` 身份）——只要客户端（如你们的 worker）通过 `Network.setExtraHTTPHeaders` 包含了 `Sec-Ch-Ua`，就会产生每请求一条的 warn 刷屏，且客户端设置的值被静默丢弃。去掉 `.fixed` 后两个问题一并解决（header 可正常覆盖，不再命中 fixed 分支，日志自然消失）。

---

## 五、需要修改的测试文件

### 5.1 修改 navigator 测试

**文件**: `src/browser/tests/navigator/navigator.html`

更新期望值：
```javascript
// 修改前
testing.expectEqual(1, navigator.userAgentData.brands.length);
testing.expectEqual({brand: 'Lightpanda', version: "1"}, navigator.userAgentData.brands[0]);

// 修改后（顺序与真实浏览器一致）
testing.expectEqual(3, navigator.userAgentData.brands.length);
testing.expectEqual({brand: 'Not=A?Brand', version: "99"}, navigator.userAgentData.brands[0]);
testing.expectEqual({brand: 'Microsoft Edge', version: "151"}, navigator.userAgentData.brands[1]);
testing.expectEqual({brand: 'Chromium', version: "151"}, navigator.userAgentData.brands[2]);
```

> **2026-09-09 重要补充**：上面这一处之前已经同步过，但这个文件里其余大部分断言长期没同步（只改到了 `userAgentData`/`getHighEntropyValues`，遗漏了第三节里其余所有改动对应的期望值），CHROME.md 之前也没记录。因为测试套件长期不能成功编译（见 1.4 节“补充修复”），这个遗漏一直没被发现。本次已全部补齐：
>
> | 断言 | 旧（上游）值 | 新（本分支）值 | 对应实现改动 |
> |------|-----------|-------------|-------------|
> | `navigator.userAgent.includes('Lightpanda')` | true | false，且需 `includes('Edg/')` | 3.1 |
> | `navigator.appVersion` | `'1.0'` | 完整 Edge UA 字符串（去掉 `Mozilla/` 前缀） | 3.1 |
> | `navigator.language` | `'en-US'` | `'zh-CN'` | 3.7 |
> | `navigator.languages` | `[2]` 长 | `[3]` 长，`['zh-CN','en-US','en']` | 3.7 |
> | `navigator.hardwareConcurrency` | `4` | `32` | 3.3 |
> | `navigator.maxTouchPoints` | `0` | `10` | 3.5 |
> | `navigator.vendor` | `''` | `'Google Inc.'` | 3.2 |
> | `navigator.doNotTrack`（`id=navigator` 和 `id=navigator_native_descriptor_walk` 两处） | `null` | `'1'` | 3.6 |
> | `navigator.deviceMemory` | `8` | `32` | 3.4 |

### 5.2 更新 userAgentData 高熵值测试

```javascript
// 修改前
testing.expectEqual('Lightpanda', v.fullVersionList[0].brand);
testing.expectEqual('1.0.0.0', v.uaFullVersion);

// 修改后（顺序与真实浏览器一致）
testing.expectEqual('Not=A?Brand', v.fullVersionList[0].brand);
testing.expectEqual('Microsoft Edge', v.fullVersionList[1].brand);
testing.expectEqual('151.0.7813.2', v.uaFullVersion);
```

### 5.3 testing.zig 中的测试配置

**文件**: `src/testing.zig`

上游测试配置中使用了 `user_agent_suffix = "internal-tester"`（会在默认 UA 后追加后缀），本分支必须移除此字段以确保测试使用纯 Chrome UA：

```zig
// 上游版本
test_config = try Config.init(test_allocator, "test", .{
    .serve = .{
        .insecure_disable_tls_host_verification = true,
        .user_agent_suffix = "internal-tester",   // ← 必须删除
        .ws_max_concurrent = 50,
        .load_resources = .{ .worker = true, .iframe = true },
        .watchdog_ms = 0,
    },
});

// 本分支版本（移除 user_agent_suffix）
test_config = try Config.init(test_allocator, "test", .{
    .serve = .{
        .insecure_disable_tls_host_verification = true,
        .ws_max_concurrent = 50,
        .load_resources = .{ .worker = true, .iframe = true },
        .watchdog_ms = 0,
    },
});
```

---

## 六、修改文件清单汇总

| # | 文件 | 修改类型 | 说明 |
|---|------|---------|------|
| 1 | `src/Config.zig` | 修改+反转测试 | 默认 UA、brands（含 full_version）、删除 validateUserAgent Mozilla 检测、添加 CH 常量、反转 CLI UA 测试；2026-09-09 又修正了该测试里的 `config.user_agent`→`config.userAgent()` 字段引用错误和内存泄漏 |
| 2 | `src/network/HttpClient.zig` | 修改 | baselineHeaders 添加 Sec-Ch-Ua-Full-Version-List + 5 个 Client Hints（[4]→[9]，Transfer.RequestHeader 类型）；2026-09-09 去掉 Sec-Ch-Ua/Full-Version-List 的 `.source=.fixed`（见 4.2）；修正 `Transfer header layering` 测试里 Mozilla UA 被拒绝的断言（见 1.4） |
| 3 | `src/browser/webapi/Navigator.zig` | 修改 | appVersion、vendor、hardwareConcurrency、deviceMemory、maxTouchPoints、doNotTrack、language/languages、platform |
| 4 | `src/browser/webapi/NavigatorUAData.zig` | 修改 | platform="Windows"、高熵值匹配 Chrome（platformVersion、uaFullVersion）；2026-09-09 拆分 shortBrandList/fullBrandList，修正低熵 brands 返回 full_version 的保真度缺陷（见 2.1、3.11） |
| 5 | `src/browser/webapi/PluginArray.zig` | 修改 | length=5 (非零) |
| 6 | `src/server/cdp/domains/emulation.zig` | 修改+反转测试 | 删除 Mozilla 拒绝逻辑；反转测试为"accepts Mozilla" |
| 7 | `src/server/cdp/domains/network.zig` | 反转测试+改载荷 | 反转 Mozilla UA 拒绝测试为接受；CRLF 测试替换载荷；2026-09-09 修正“冒号 smuggle”测试应保留“拒绝”语义（拒绝原因与 Mozilla 无关，是 header 名字非 token，见 1.4） |
| 8 | `src/help.zon` | 修改 | 更新帮助文本（user-agent + user-agent-suffix） |
| 9 | `src/browser/tests/navigator/navigator.html` | 修改 | 更新 brands 和 highEntropy 期望值为 Chrome 值；2026-09-09 补齐此前遗漏的 appVersion/language(s)/hardwareConcurrency/maxTouchPoints/vendor/doNotTrack/deviceMemory 期望值（见 5.1） |
| 10 | `src/testing.zig` | 修改 | 删除上游 `user_agent_suffix = "internal-tester"`，使用默认 Chrome UA |
| 11 | `src/browser/webapi/WorkerNavigator.zig` | 上游同步联动 | 上游新增文件，`getLanguages` 委托 `Navigator.getLanguages`，返回类型 `[2]`→`[3]` 对齐本分支（否则 snapshot_creator 编译失败） |
| 12 | `src/browser/tests/net/fetch.html` | 反转测试（2026-09-09） | `fetch_header_layers` 反转为接受 Mozilla UA + author 层可覆盖 Sec-Ch-Ua |
| 13 | `src/browser/tests/net/xhr.html` | 反转测试（2026-09-09） | `xhr_request_headers` 同上 |
| 14 | `src/browser/webapi/net/Fetch.zig` | 测试计数修正（2026-09-09） | `test "WebApi: fetch"` 删除对应的 `expectLog(&.{ .http, .http })` |
| 15 | `src/browser/webapi/net/XMLHttpRequest.zig` | 测试计数修正（2026-09-09） | `test "WebApi: XHR"` 的 `expectLog` 从 3 条减为 1 条 |

---

## 七、未修改但需注意的限制（TLS 指纹）

以下问题无法通过代码层面完全解决：

1. **TLS 指纹 (JA3/JA4)**: libcurl 使用 OpenSSL，其 TLS 握手指纹与真实 Chrome/Edge 不同。Cloudflare 等高级反爬系统可通过 TLS 指纹识别非浏览器客户端。这需要修改 libcurl 的 TLS 配置或使用 curl-impersonate 才能模拟 Chrome 的 TLS 指纹。

2. **HTTP/2 指纹**: libcurl 的 HTTP/2 设置帧、WINDOW_UPDATE 等参数与真实浏览器不同。

3. **Canvas/WebGL 指纹**: headless 无 GPU，Canvas 返回空白或软件渲染特征。

4. **WebRTC**: headless 下可能无 WebRTC 或返回异常 IP。

---

## 八、编译方案

### 8.1 新服务器环境搭建（完整清单）

#### 8.1.1 系统依赖包（Ubuntu 22.04+ / Debian 12+）

```bash
apt-get update && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config \
    libglib2.0-dev clang make curl git
```

#### 8.1.2 Zig 编译器（v0.16.0）

```bash
ZIG_VERSION="0.16.0"
curl -LO https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz
tar xf zig-x86_64-linux-${ZIG_VERSION}.tar.xz
rm -rf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}
mv zig-x86_64-linux-${ZIG_VERSION} /usr/local/lib
ln -sf /usr/local/lib/zig-x86_64-linux-${ZIG_VERSION}/zig /usr/local/bin/zig
rm zig-x86_64-linux-${ZIG_VERSION}.tar.xz
zig version  # 应输出 0.16.0
```

#### 8.1.3 Rust 工具链（使用国内镜像）

```bash
# 设置镜像环境变量（写入 ~/.bashrc）
echo 'export RUSTUP_DIST_SERVER="https://rsproxy.cn"' >> ~/.bashrc
echo 'export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"' >> ~/.bashrc
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"

# 安装
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y
source $HOME/.cargo/env

# 配置 crates.io 镜像
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

rustc --version
```

#### 8.1.4 V8 引擎预编译库（127 MB）

```bash
make download-v8
# 或手动下载 v0.5.4 版本
curl -fL -o .lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a \
  https://github.com/lightpanda-io/zig-v8-fork/releases/download/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a
```

#### 8.1.5 环境变量

```bash
# PATH（写入 ~/.bashrc 持久化）
echo 'export PATH="/usr/local/bin:$HOME/.cargo/bin:$HOME/.local/bin:$PATH"' >> ~/.bashrc
echo 'export LIGHTPANDA_DISABLE_TELEMETRY=1' >> ~/.bashrc
source ~/.bashrc

# Agent 模式（按需）
# export GOOGLE_API_KEY="your-key"
# export GEMINI_API_KEY="your-key"
```

#### 8.1.6 一键初始化脚本

```bash
#!/bin/bash
set -euo pipefail

PROJECT_DIR="${1:-$(pwd)}"
echo "==> [1/6] 安装系统依赖..."
apt-get update -qq && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config libglib2.0-dev clang make curl git

echo "==> [2/6] 安装 Zig 0.16.0..."
cd /tmp
curl -fsSLO https://ziglang.org/download/0.16.0/zig-x86_64-linux-0.16.0.tar.xz
tar xf zig-x86_64-linux-0.16.0.tar.xz
rm -rf /usr/local/lib/zig-x86_64-linux-0.16.0
mv zig-x86_64-linux-0.16.0 /usr/local/lib
ln -sf /usr/local/lib/zig-x86_64-linux-0.16.0/zig /usr/local/bin/zig
rm zig-x86_64-linux-0.16.0.tar.xz
zig version

echo "==> [3/6] 安装 Rust（国内镜像）..."
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

echo "==> [4/6] 下载 V8 预编译库..."
cd "$PROJECT_DIR"
make download-v8

echo "==> [5/6] 编译 release 版本..."
export LIGHTPANDA_DISABLE_TELEMETRY=1
# make build  # 默认针对编译机 CPU 优化（AMD EPYC 9T25），可能不兼容 Intel
# 针对 Intel Skylake-SP (Xeon Platinum) 编译，确保指令集兼容
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"

echo "==> [6/6] 安装到 ~/.local/bin..."
mkdir -p "$HOME/.local/bin"
cp -f zig-out/bin/lightpanda "$HOME/.local/bin/lightpanda"

echo ""
echo "✅ 完成！"
echo "   命令: lightpanda"
echo "   启动服务: lightpanda serve --host 0.0.0.0 --port 9222"
```

**用法**：
```bash
# 在项目根目录执行
chmod +x init.sh && sudo ./init.sh /path/to/browser
```

### 8.2 首次编译

```bash
# 下载 V8 预编译库（127 MB）
make download-v8

# 编译 release 版本（含 V8 snapshot）
# make build  # 默认针对编译机 CPU 优化，可能不兼容其他 CPU
# 针对 Intel Skylake-SP (Xeon Platinum) 编译
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"

# 安装到 ~/.local/bin（确保 ~/.local/bin 在 PATH 中）
mkdir -p ~/.local/bin
cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda

# 编译 debug 版本
make build-dev

# 运行测试
make test

# 启动服务
lightpanda serve --host 0.0.0.0 --port 9222
```

### 8.3 增量编译

```bash
# make build    # 默认针对编译机 CPU 优化
# 针对 Intel Skylake-SP (Xeon Platinum) 编译
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"
make clean    # 完全清理（保留 V8 缓存）
```

> **CPU 指令集说明**：编译机为 AMD EPYC 9T25（Zen 5），默认编译会启用 Zen 5 特有指令（如 AVX-512 VNNI/VBMI），
> 这些指令在 Intel Xeon Platinum（Skylake-SP）上不存在，导致运行时 `SIGILL`（非法指令）崩溃。
> 使用 `-Dcpu=skylake_avx512` 确保只生成 Skylake-SP 支持的指令集（AVX-512 F/DQ/CD/BW/VL）。
> 如需兼容更老的 CPU，改用 `-Dcpu=baseline`（任何 x86_64 都能跑，性能损失约 5-10%）。

### 8.4 编译产物

- 可执行文件：`./zig-out/bin/lightpanda`（编译输出）
- 安装位置：`~/.local/bin/lightpanda`（安装后）
- V8 snapshot：`src/snapshot.bin`
- V8 缓存：`.lp-cache/prebuilt-v8/v0.5.4/`

### 8.5 常见问题

```bash
# html5ever 编译失败 → 检查 Rust
cargo --version

# Zig 版本不对
zig version  # 必须 0.16.0

# V8 下载失败 → 手动下载后放入缓存
mkdir -p .lp-cache/prebuilt-v8/v0.5.4/
curl -fL -o .lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a \
  https://github.com/lightpanda-io/zig-v8-fork/releases/download/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a
```

### 8.6 大陆网络下离线抓取 build.zig.zon 依赖（同步上游后必做）

上游每次同步都可能 bump `build.zig.zon` 里的依赖版本；本机直连 GitHub 的 HTTPS 抓取会超时（`HttpConnectionClosing` / `Timeout`），而 **SSH 到 GitHub 可用**。注意：即使传了 `-Dprebuilt_v8_path`，`.v8` 依赖 tarball（Zig/C 绑定源码）仍必须解析，预编译 `.a` 无法替代它。

按依赖类型分别处理（原理：镜像/SSH 取回的字节与原 GitHub 一致 → `zig fetch` 算出的 hash 与 `build.zig.zon` 精确匹配 → 落全局缓存 `~/.cache/zig/p/`，`build.zig.zon` 一行都不用改）：

```bash
# 1) tarball 类依赖（如 v8、curl）——走 GitHub HTTP 镜像 ghfast.top
zig fetch "https://ghfast.top/https://github.com/lightpanda-io/zig-v8-fork/archive/<commit>.tar.gz"
zig fetch "https://ghfast.top/https://github.com/curl/curl/releases/download/<tag>/<file>.tar.gz"

# 2) git+https 类依赖（如 zenai、isocline）——用 GIT_CONFIG 临时把 https 改写成 SSH，不动全局 git 配置
GIT_CONFIG_COUNT=1 \
  GIT_CONFIG_KEY_0="url.git@github.com:.insteadOf" \
  GIT_CONFIG_VALUE_0="https://github.com/" \
  zig fetch "git+https://github.com/lightpanda-io/zenai.git#<commit>"

# 3) 预编译 V8 .a（裸二进制，非 zig 包）——走镜像，见 8.1.4 / 8.5

# 全部抓完后验证：应 EXIT=0 且无输出，代表完全离线可解析
zig build --fetch
```

> 校验方法：`zig fetch` 输出的 hash 必须与 `build.zig.zon` 中对应依赖的 `.hash` 完全一致，否则说明字节不同（镜像失效或被篡改）。
> 备选镜像：`https://gh-proxy.com/`（与 `ghfast.top` 等效，可互换）。

---

## 九、Git 分支管理与上游同步方案

### 9.1 仓库来源

当前仓库直接克隆自 `lightpanda-io/browser`（官方上游），所有自定义修改在本地 `chrome` 分支上维护。

```
官方上游  ──→  lightpanda-io/browser（只读，只拉不推）
                                │
                                ▼
                        你本机的 main 分支（跟踪上游，保持纯净）
                                │
                                ├── chrome 分支 ─── 所有 Chrome 伪装修改在此
```

### 9.2 初始化远程（仅首次）

```bash
# 当前 origin 指向上游
# 将 origin 重命名为 upstream（指向官方仓库）
git remote rename origin upstream

# 添加自己的远程仓库（你 fork 后的备份）
git remote add origin https://github.com/ewpkw/browser.git

# 验证
git remote -v
# 应看到:
# origin    https://github.com/ewpkw/browser.git (fetch)
# origin    https://github.com/ewpkw/browser.git (push)
# upstream  https://github.com/lightpanda-io/browser.git (fetch)
# upstream  https://github.com/lightpanda-io/browser.git (push)
```

### 9.3 创建自定义分支

```bash
# 切到 main，创建 chrome 分支（一次性的，后续同步用 merge）
git checkout main
git checkout -b chrome

# 将所有修改 commit 到 chrome 分支
git add -A
git commit -m "feat(chrome): 伪装成 Chrome/Edge 浏览器

- 默认 UA 改为 Edge 151
- Sec-Ch-Ua brands 改为 Not=A?Brand / Microsoft Edge / Chromium
- 删除 Mozilla UA 校验
- navigator 属性全部对齐真实浏览器
- 添加 Sec-Ch-Ua-Platform / Architecture / Bitness / WoW64 等 Client Hint
- PluginArray.length 改为 5"

# 推送到自己的远程仓库（可选备份）
git push origin chrome
```

### 9.4 日常同步上游流程

```bash
# 1. 切到 main，拉取上游最新代码
git checkout main
git fetch upstream

# 2. 更新 main 到最新上游
git merge upstream/main
# 此时 main 就是上游的最新代码

# 3. 切回 chrome 分支，将上游更新合并进来
git checkout chrome
git merge main
# 或者用 rebase（更清洁但需 force push）：
# git rebase main

# 4. 解决冲突（如有）
#    冲突热点文件：
#    - src/Config.zig     (UA 默认值、brands、validateUserAgent)
#    - src/network/http.zig  (Headers.init)
#    - src/browser/webapi/Navigator.zig
#    - src/browser/webapi/NavigatorUAData.zig
#    - src/server/cdp/domains/emulation.zig
#    - src/server/cdp/domains/network.zig

# 5. 编译验证（确认合并无误）
# make build && make test  # 默认针对编译机 CPU
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"

cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda
~/.local/bin/lightpanda version

# 6. 备份到自己的远程仓库
#    如果用 rebase 了需要 force push
git push origin chrome
```

### 9.5 冲突预防：最小 diff 原则

为了降低每次合并上游时的冲突概率：

1. **只改必要行**：每次修改尽量集中在一两行内，不要大面积重构。
2. **不改测试结构**：测试只是更新期望值，不要改动测试本身的逻辑。
3. **记录冲突文件**：对冲突热点了如指掌，合并时直奔目标。

### 9.6 同步节奏建议

| 频率 | 操作 |
|------|------|
| 每周 | `git fetch upstream` 查看上游变动 |
| 有冲突时 | 执行完整 merge/rebase 流程 |
| 上游大版本更新 | 特别注意 build.zig.zon 依赖版本变化 |

### 9.7 本次同步记录（2026-09-09）

- 合并 `upstream/main`（自上次同步 `c5aaafe59` 以来共 **89 个新提交**）。
- **零文本冲突**（`git merge-tree` 预演 + 实际 `git merge --no-ff` 均无冲突），无需手工解决。
- 已逐个确认本分支的所有冲突热点文件（`Config.zig`/`HttpClient.zig`/`Navigator.zig`/`NavigatorUAData.zig`/`PluginArray.zig`/`emulation.zig`/`network.zig`/`help.zon`/`testing.zig`/`WorkerNavigator.zig`/`navigator.html`）在这 89 个新提交里的真实改动内容，**均未触及本分支的 UA / Mozilla 校验 / Sec-Ch-Ua / Navigator / PluginArray 相关代码**，不需要新的策略反转。
- `build.zig`、`build.zig.zon`、`.github/actions/install/action.yml`（V8 版本/标签）、`Makefile` **均无变化**，不需要重新执行第 8.6 节的大陆网络离线抓取流程，V8 缓存路径（`v0.5.4`）不变。
- 本次新合入的上游功能：adblock 引入 request-engine（`isUrlBlocked` 改为接收 `*const Transfer`，支持 `$document`/`$subdocument`/`$third-party` 修饰符）；CLI `--dump` 拼写建议（did-you-mean）；`--adblock-lists`/`--block-cidrs`/`--block-urls` 可重复传值累加；HTTP 超时默认值调整（connect `0`→`8000ms`，transfer `5000`→`15000ms`）；`Emulation.setDeviceMetricsOverride`/`clearDeviceMetricsOverride` 改为调用 `Browser.setViewportOverride()`；isolated world 每 frame 独立 context；`Performance` 实现 `EventTarget`；等等。

**合并过程中发现并修复的历史遗留问题**（均非本次合并引入，是之前几轮 `git merge upstream/main` 遗留的，本轮一并清理）：

1. `src/Config.zig`：`test "Config: parseArgs accepts a mozilla user-agent"` 写错了字段访问方式（`config.user_agent` 应为 `config.userAgent()`），导致测试直接**编译失败**，阻塞整个 `make test`（之前几轮合并只验证了 `make build`，`test` 块不参与 `make build`，所以一直没暴露）。同时修正该测试的内存泄漏（漏用 arena）。
2. `src/network/HttpClient.zig`：`test "HttpClient: Transfer header layering"`（上游 `737f69ee4` 引入）仍按“Mozilla UA 会被拒绝”断言，与本分支核心原则冲突，已反转（见 1.4）。
3. `src/server/cdp/domains/network.zig`：“冒号走私 key”测试之前被错误地反转为“接受”，已改回“拒绝”，并注明拒绝原因跟 Mozilla 无关（见 1.4）。
4. `src/browser/tests/net/{fetch,xhr}.html`：`fetch_header_layers` / `xhr_request_headers`（同一上游提交 `737f69ee4` 引入的 HTML 版姊妹测试）同样漏改，已同步修正（见 1.4）。
5. `src/browser/tests/navigator/navigator.html`：`CHROME.md` 第 3.1~3.9 节实现改动对应的测试期望值长期未同步（见 5.1 里的对照表）。
6. `src/browser/webapi/NavigatorUAData.zig`：发现并修复一个**真实保真度缺陷**——`navigator.userAgentData.brands`（低熵）错误返回了完整构建版本号，与 `Sec-Ch-Ua` HTTP 头不一致，会被反爬交叉校验识别（见 2.1、3.11）。这个是直接影响本分支存在意义的问题，之前因测试套件整体编译不过（第 1 点）而被掩盖，从未被发现。
7. `src/network/HttpClient.zig` `baselineHeaders()`：`Sec-Ch-Ua`/`Sec-Ch-Ua-Full-Version-List` 之前被标为 `.source = .fixed`（沿用上游设计），导致生产环境 `worker` 通过 CDP `setExtraHTTPHeaders` 设置这些 header 时，每个请求都会刷一条 `ignore overriding fixed header` 的 warn 日志，且客户端设置的值被静默丢弃。已去掉 `.fixed`（见 4.2）。

**本次验证结果**：`make test` 共 **1460 个测试，1458 个通过**，失败 2 个（见下）。

**已知遗留问题：`WebApi: Frames` / `WebApi: Window` 两个测试确定性失败**

- 与本次合并、与本分支的 Chrome 模拟目标均无直接关系：失败点分别是 `cross_realm_collection.html`（iframe 跨 realm `childNodes` 缓存失效）和 `body_onload3.html`（`window.onload`/`body.onload` 类型判定）。
- 已确认**不是本次合并、也不是本轮修复引入的**：在合并前的 HEAD 上完全一样的失败，纯粹是之前几次合并遗留的问题（因为测试套件从更早版本开始就一直编译不过，被完全掩盖）。
- 已确认**不是随机波动，也不是测试端口冲突**：单独跑、清空 `.zig-cache` 重跑、确认 `127.0.0.1:9582` 空闲时单独跑，均稳定重现。
- 已确认**不是本分支相对上游的源码差异造成**：对比 `git diff upstream/main..HEAD`，`Window.zig`/`node_live.zig`/`Frame.zig`/`Node.zig` 等涉及文件均与上游完全一致（无 diff）；把两边相关源码内容直接 `diff` 对比，也是字节对字节完全相同。
- 已确认**不是编译缓存问题**：用全新 `--cache-dir` 完全冷构建，仍失败。
- 已定位到一个确定性的复现/排除步骤：在 `/tmp` 下新建一个 `git worktree add /tmp/lp-upstream-check upstream/main`，先把本分支与上游的 **16 个差异源文件**（`git diff --name-only upstream/main..HEAD` 列出的、不含 `CHROME.md`）全部拷进去覆盖 → 在这个纯净的 upstream 工作树里用同一命令跑 `WebApi: Frames`，**成功复现了失败**。然后逐个二分回退：
  - 回退 `testing.zig` → 仍失败（排除）
  - 再回退 `HttpClient.zig` + `http.zig` → 仍失败（排除）
  - 再回退 `Config.zig` + `help.zon` + `emulation.zig` + `network.zig` 四个 → **PASS**（锁定范围内）
  - 四个里单独回退 `help.zon` + `network.zig`，保留 `Config.zig` + `emulation.zig` → 仍失败
  - **已缩小到：罪魁祸首在 `src/Config.zig` 和/或 `src/server/cdp/domains/emulation.zig` 这两个文件里**（因 `error.Reserved` 的声明与 `switch` 分支必须同进同退，这两个文件不能单独拆开测试，否则直接编译失败，无法进一步二分）。
  - 令人费解的是：逐行比对这两个文件与上游的完整 diff（就是 1.3/1.4 节记录的那些 UA/Mozilla 相关改动），找不到任何看似与 DOM/iframe/collection 缓存相关的内容。具体机制需要单独开一个专题排查（建议：先固定住其他 14 个文件，只把 Config+emulation 换成上游版本确认修复，再把这两个文件里的具体改动拆成更小的语义单元——比如单独只改 `user_agent_base` 字符串、单独只删 `validateUserAgent` 里的 Mozilla 判断——逐个尝试，定位到具体哪一行造成的）。
  - 本问题不阻塞本次合并，本次任务范围内不再继续深入，待你确认下一步。

---

## 十、验证方法

修改完成后，通过以下方式验证：

```bash
# 1. 编译
# make build  # 默认针对编译机 CPU 优化
# 针对 Intel Skylake-SP (Xeon Platinum) 编译
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"

# 2. 运行测试
make test

# 3. 运行特定测试（验证 navigator 相关修改）
TEST_FILTER="Navigator" make test

# 4. 访问指纹检测网站
lightpanda fetch --url "https://bot.sannysoft.com" --dump html
lightpanda fetch --url "https://abrahamjuliot.github.io/creepjs/" --dump html
lightpanda fetch --url "https://www.browserscan.net/bot-detection" --dump html

# 5. 验证 HTTP 请求头
lightpanda fetch --url "https://httpbin.org/headers" --dump html
# 确认返回的 headers 中包含：
# - User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
# - Sec-Ch-Ua: "Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"
# - Sec-Ch-Ua-Platform: "Windows"
# - Accept-Language: en-US,en;q=0.9
```
