# Chrome 浏览器伪装修改方案

## 目标

将 Lightpanda 伪装成正常的 Chrome/Edge 浏览器，绕过常见的反爬检测（Cloudflare、Akamai、DataDome 等）。

**本文档只描述生产代码（`make build` 编译进二进制的部分）相对上游的改动。** 本分支不修改任何测试文件、不改任何 `test {}` 块内的代码——上游的测试逻辑（包括上游自己写的"拒绝 Mozilla UA"类测试断言）一律保持原样，不参与本分支的适配。同步上游时，只需要关注下面列出的文件，其余文件（包括所有测试代码）都不需要为本分支的目的做适配。

## 目标 UA 字符串

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0
```

---

## 一、修改文件总览（8 个文件）

| # | 文件 | 作用域 |
|---|------|--------|
| 1 | `src/Config.zig` | 默认 UA、Client-Hint brands / 常量、`validateUserAgent` 行为、CLI 错误提示文案 |
| 2 | `src/help.zon` | `--user-agent` / `--user-agent-suffix` 帮助文本 |
| 3 | `src/network/HttpClient.zig` | `baselineHeaders()`：新增 Client Hints，且不再把 `Sec-Ch-Ua` 系头标为不可覆盖 |
| 4 | `src/browser/webapi/Navigator.zig` | `navigator.*` 各属性对齐真实 Edge/Windows 取值 |
| 5 | `src/browser/webapi/NavigatorUAData.zig` | `navigator.userAgentData` 的 `platform` / 高熵值，以及 `brands` 版本号取短版本 |
| 6 | `src/browser/webapi/PluginArray.zig` | `navigator.plugins.length` 非零 |
| 7 | `src/browser/webapi/WorkerNavigator.zig` | `getLanguages` 返回类型跟随 `Navigator.getLanguages` 联动 |
| 8 | `src/server/cdp/domains/emulation.zig` | `Emulation.setUserAgentOverride` 不再因 UA 含 Mozilla 而忽略 |

以下改动全部是生产行为改动，没有一处是测试代码改动。

---

## 二、逐文件改动详情

### 2.1 `src/Config.zig`

#### (a) 默认 UA

```zig
// 上游
const user_agent_base: [:0]const u8 = "Lightpanda/1.0";

// 本分支
const user_agent_base: [:0]const u8 = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0";
```

#### (b) Client-Hint brands（顺序、品牌名、版本号均对齐真实 Edge 151）

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

> - 品牌名是 `Not=A?Brand`（等号 + 问号），不是 `Not-A.Brand`，顺序也不能变。
> - `.version`（短版本号，如 `"151"`）用于 HTTP 头 `Sec-Ch-Ua` 和 `navigator.userAgentData.brands`。
> - `.full_version`（完整构建号，如 `"151.0.7813.2"`）用于 HTTP 头 `Sec-Ch-Ua-Full-Version-List` 和 `navigator.userAgentData.getHighEntropyValues().fullVersionList`。
> - 两者必须分别对应，不能混用，详见 2.5。

#### (c) 新增 Client-Hint 字符串常量（定义在 `accept_language` 之后、`navigation_accept` 之前）

```zig
pub const sec_ch_ua_platform: [:0]const u8 = "Sec-Ch-Ua-Platform: \"Windows\"";
pub const sec_ch_ua_mobile: [:0]const u8 = "Sec-Ch-Ua-Mobile: ?0";
pub const sec_ch_ua_arch: [:0]const u8 = "Sec-Ch-Ua-Arch: \"x86\"";
pub const sec_ch_ua_bitness: [:0]const u8 = "Sec-Ch-Ua-Bitness: \"64\"";
pub const sec_ch_ua_wow64: [:0]const u8 = "Sec-Ch-Ua-WoW64: ?0";
```

> 所有 Client-Hint 值统一定义在此处，供 `HttpClient.zig` 的 `baselineHeaders()` 引用（见 2.3），避免字符串散落在多个文件。

#### (d) `validateUserAgent`：去掉 Mozilla 拒绝逻辑，保留非打印字符检测

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

// 本分支
pub fn validateUserAgent(ua: []const u8) !void {
    for (ua) |c| {
        if (!std.ascii.isPrint(c)) {
            return error.NonPrintable;
        }
    }
}
```

> `error.Reserved` 在本分支不再被 `validateUserAgent` 返回；所有调用方（见 2.2/2.8/2.3 关联点）都不应再假设这个错误会发生。

#### (e) `userAgentValidator` 里 CLI 报错提示文案

```zig
// 上游
log.fatal(.app, "invalid user-agent", .{ .err = err, .hint = "must be printable ASCII and can't contain Mozilla" });

// 本分支
log.fatal(.app, "invalid user-agent", .{ .err = err, .hint = "must be printable ASCII" });
```

### 2.2 `src/help.zon`

```
// 上游
--user-agent <STRING>
    Override the User-Agent header entirely. Must not impersonate other
    browsers; any value containing "Mozilla" is forbidden. The browser
    still sends Sec-Ch-Ua. Incompatible with --user-agent-suffix.
--user-agent-suffix <STRING>
    Suffix appended to the Lightpanda/X.Y User-Agent.

// 本分支
--user-agent <STRING>
    Override the User-Agent header entirely. The browser still sends
    Sec-Ch-Ua. Incompatible with --user-agent-suffix.
--user-agent-suffix <STRING>
    Suffix appended to the default User-Agent.
```

### 2.3 `src/network/HttpClient.zig` — `baselineHeaders()`

```zig
// 上游
pub fn baselineHeaders(self: *const Client) [4]Transfer.RequestHeader {
    return .{
        .{ .name = "User-Agent", .value = self.getUserAgent() },
        .{ .name = "Sec-Ch-Ua", .value = lp.Config.HttpHeaders.sec_ch_ua, .source = .fixed },
        .{ .name = "Sec-Ch-Ua-Full-Version-List", .value = lp.Config.HttpHeaders.sec_ch_ua_full_version_list, .source = .fixed },
        // Omitting Accept-Language triggers bot-protection on some CDNs
        // (Akamai) when Accept-Encoding is present.
        .{ .name = "Accept-Language", .value = lp.Config.HttpHeaders.accept_language },
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

**要点：**

- 数组长度 `[4]` → `[9]`，新增 5 个 Client-Hint 头（值对应 2.1(c) 里的常量，这里直接写字面量字符串）。
- `Sec-Ch-Ua` / `Sec-Ch-Ua-Full-Version-List` **不带** `.source = .fixed`。这是本分支刻意与上游不一致的一处：上游给这两个头加了 `.fixed`，是为了保护它自己"永不接受 Mozilla UA"的设定不被 CDP 绕过；但对本分支来说效果正好相反——CDP 客户端（比如 `Network.setExtraHTTPHeaders`）如果带着 `Sec-Ch-Ua` 一起下发，`Transfer.putHeader` 会命中 `.fixed` 分支，**每个 HTTP 请求都打一条 `ignore overriding fixed header` warn 日志，同时客户端设置的值被静默丢弃**。去掉 `.fixed` 后，日志刷屏消失，且行为跟真实浏览器一致：CDP 层设置应该能覆盖 baseline，跟设置其他普通请求头没区别。
- 如果未来上游再次给这两个头加回 `.fixed`，同步时注意手工去掉。

### 2.4 `src/browser/webapi/Navigator.zig`

| 函数 | 上游返回 | 本分支返回 |
|------|---------|-----------|
| `getLanguages` | `[2][]const u8`：`.{ "en-US", "en" }` | `[3][]const u8`：`.{ "zh-CN", "en-US", "en" }`（返回类型也要一起改） |
| `getDoNotTrack` | `null` | `"1"` |
| `getAppVersion` | `"1.0"` | `"5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0"`（= UA 去掉 `Mozilla/` 前缀，需与 `navigator.userAgent` 保持一致，反爬会交叉比对） |
| `getLanguage` | `"en-US"` | `"zh-CN"` |
| `getHardwareConcurrency` | `4` | `32` |
| `getDeviceMemory` | `8.0` | `32`（Chrome 对该值返回整数形式的 GB 数） |
| `getMaxTouchPoints` | `0` | `10`（0 是明显的无触屏 / headless 特征） |
| `getVendor` | `""` | `"Google Inc."`（真实 Chrome 固定返回这个值） |
| `getPlatform` | 按 `builtin.os.tag` 分支返回 `MacIntel`/`Win32`/`Linux x86_64`/... | 固定返回 `"Win32"`（对应 UA 里的 `Windows NT 10.0; Win64; x64`；注意 Chrome 即使在 64 位系统也返回 `Win32`，不是 `Win64`） |

`getAppName`（`"Netscape"`）、`getProduct`（`"Gecko"`）、`getGlobalPrivacyControl`（`false`）等其余 getter 保持上游原样，不需要改。

### 2.5 `src/browser/webapi/NavigatorUAData.zig`

#### `uaPlatform()`：固定 `"Windows"`

```zig
// 上游
fn uaPlatform() []const u8 {
    return switch (builtin.os.tag) {
        .macos => "macOS",
        .windows => "Windows",
        .linux => "Linux",
        .freebsd => "FreeBSD",
        else => "Unknown",
    };
}

// 本分支
fn uaPlatform() []const u8 {
    return "Windows";
}
```

#### `brandList()` 拆成 `shortBrandList()` / `fullBrandList()`

**这是保真度关键点，务必理解后再合并/同步：** 上游原本只有一个 `brandList()`，内部固定取 `b.full_version`。直接照抄的后果是：`navigator.userAgentData.brands`（低熵，真实 Chrome/Edge 应该返回**主版本号**，如 `"151"`）跟 HTTP 头 `Sec-Ch-Ua`（用的是 `.version`，即 `"151"`）对不上——因为 `brandList()` 会把 `full_version`（如 `"151.0.7813.2"`）也塞进 `.brands`。任何一个把 JS 侧 `navigator.userAgentData.brands` 跟 HTTP 请求头 `Sec-Ch-Ua` 做交叉比对的反爬检测，都会立刻发现这个不一致，直接判定为伪造客户端。

本分支把 `brandList()` 拆成两个函数，按语义分别给低熵/高熵两处用：

```zig
// 低熵：navigator.userAgentData.brands / toJSON().brands / getHighEntropyValues().brands
// 用 .version（短版本号，如 "151"），必须与 Config.HttpHeaders.sec_ch_ua 一致
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

// 高熵：getHighEntropyValues().fullVersionList
// 用 .full_version（完整构建号，如 "151.0.7813.2"），必须与 Config.HttpHeaders.sec_ch_ua_full_version_list 一致
fn fullBrandList() []const Brand {
    const out = comptime blk: {
        const src = &Config.HttpHeaders.brands;
        var arr: [src.len]Brand = undefined;
        for (src, 0..) |b, i| {
            arr[i] = .{ .brand = b.brand, .version = b.full_version };
        }
        const final = arr;
        break :blk final;
    };
    return &out;
}
```

调用关系：

```zig
pub fn getBrands(_: *const NavigatorUAData) []const Brand {
    return shortBrandList();
}

pub fn toJSON(_: *const NavigatorUAData) struct { ... } {
    return .{
        .mobile = false,
        .brands = shortBrandList(),
        .platform = uaPlatform(),
    };
}

pub fn getHighEntropyValues(_: *const NavigatorUAData, hints: []const []const u8, exec: *const Execution) !js.Promise {
    _ = hints;
    return exec.js.local.?.resolvePromise(.{
        .brands = shortBrandList(),
        .mobile = false,
        .platform = uaPlatform(),
        .architecture = "x86",
        .bitness = "64",
        .model = "",
        .platformVersion = "15.0.0",
        .uaFullVersion = "151.0.7813.2",
        .fullVersionList = fullBrandList(),
        .wow64 = false,
        .formFactor = [_][]const u8{"Desktop"},
    });
}
```

> 高熵值里的 `architecture`/`bitness`/`platformVersion`/`uaFullVersion` 从上游的"跟随编译机器 CPU / 留空"改成固定 Windows/x86_64 对应的真实值，与 `navigator.platform = "Win32"`、UA 字符串保持一致。

### 2.6 `src/browser/webapi/PluginArray.zig`

```zig
// 上游
pub const length = bridge.property(0, .{ .template = false });

// 本分支
pub const length = bridge.property(5, .{ .template = false });
```

> 真实 Chrome 默认有 5 个内置插件（PDF Viewer 等）。长度非 0 就能过掉大部分只看 `navigator.plugins.length` 的检测；`[...navigator.plugins]` 迭代仍返回空数组（未改动底层数据源），一般检测不会走到这一步。

### 2.7 `src/browser/webapi/WorkerNavigator.zig`

上游新增文件，内部 `getLanguages` 直接委托 `Navigator.getLanguages`，返回类型也写死为 `[2][]const u8`：

```zig
// 上游
pub fn getLanguages(_: *const WorkerNavigator) [2][]const u8 {
    return Navigator.getLanguages(&Navigator.init);
}

// 本分支（跟随 2.4 里 `Navigator.getLanguages` 的返回类型改动，必须同步改，否则 `snapshot_creator` 编译失败）
pub fn getLanguages(_: *const WorkerNavigator) [3][]const u8 {
    return Navigator.getLanguages(&Navigator.init);
}
```

> 这是本分支 2.4 改动引起的必要联动，无法绕开：只要改了 `Navigator.getLanguages` 的返回类型，就必须同步改这个委托调用的返回类型标注。

### 2.8 `src/server/cdp/domains/emulation.zig` — `setUserAgentOverride`

去掉 `error.Reserved` 分支（对应 2.1(d) 里 `validateUserAgent` 不再返回 `error.Reserved`）：

```zig
// 上游
const ua = params.userAgent;
Config.validateUserAgent(ua) catch |err| switch (err) {
    error.NonPrintable => return cmd.sendError(-32602, "User agent contains non-printable characters", .{}),
    error.Reserved => {
        log.warn(.not_implemented, "Emulation.setUserAgentOverride", .{ .param = "userAgent", .value = ua, .info = "User agent must not contain Mozilla" });
        return cmd.sendResult(null, .{});
    },
};

// 本分支
const ua = params.userAgent;
Config.validateUserAgent(ua) catch |err| switch (err) {
    error.NonPrintable => return cmd.sendError(-32602, "User agent contains non-printable characters", .{}),
};
```

> 这是本分支的核心目的之一：CDP 侧 `Emulation.setUserAgentOverride` 传入含 Mozilla 的 UA 必须被接受并生效，不能再静默忽略。
> **注意**：因为 2.1(d) 已经从 `validateUserAgent` 里移除了 `error.Reserved` 这个分支路径，如果以后上游重新引入对 `error.Reserved` 的 switch 分支处理（或者上游改了 `validateUserAgent` 的错误集），这里会出现「switch 未穷尽」或「分支不可达」的编译错误，属于预期内的合并冲突点，看到这种报错就直接按本节内容重新对齐这两处。

---

## 三、关于测试代码

**本分支不维护任何测试代码的适配。** 上游自带的测试（包括形如「拒绝 Mozilla UA」的断言、以及依赖 `navigator.*` 上游默认值的用例）在本分支下会因为上面的生产改动而失败，这是预期行为，不需要修复，也不需要为了让测试变绿去改动测试文件或 `test {}` 块里的断言。

`make test` 目前存在的失败（如 `WebApi: Navigator`、`cdp.Emulation: setUserAgentOverride ignores mozilla` 等），根因全部是"上游测试断言的是上游默认行为，而本分支故意改了生产行为"，不影响生产二进制（`test {}` 块不参与 `make build`）。

---

## 四、未修改但需注意的限制（TLS 指纹等）

以下问题无法通过代码层面完全解决：

1. **TLS 指纹 (JA3/JA4)**：libcurl 使用 OpenSSL，其 TLS 握手指纹与真实 Chrome/Edge 不同。Cloudflare 等高级反爬系统可通过 TLS 指纹识别非浏览器客户端。这需要修改 libcurl 的 TLS 配置或使用 curl-impersonate 才能模拟 Chrome 的 TLS 指纹。
2. **HTTP/2 指纹**：libcurl 的 HTTP/2 设置帧、WINDOW_UPDATE 等参数与真实浏览器不同。
3. **Canvas/WebGL 指纹**：headless 无 GPU，Canvas 返回空白或软件渲染特征。
4. **WebRTC**：headless 下可能无 WebRTC 或返回异常 IP。

---

## 五、编译方案

### 5.1 新服务器环境搭建（完整清单）

#### 5.1.1 系统依赖包（Ubuntu 22.04+ / Debian 12+）

```bash
apt-get update && apt-get install -y --no-install-recommends \
    xz-utils ca-certificates pkg-config \
    libglib2.0-dev clang make curl git
```

#### 5.1.2 Zig 编译器（v0.16.0）

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

#### 5.1.3 Rust 工具链（使用国内镜像）

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

#### 5.1.4 V8 引擎预编译库（127 MB）

```bash
make download-v8
# 或手动下载 v0.5.4 版本（缓存路径带 tag 版本号，不能省掉这一层目录，
# 详见 Makefile 里 V8_CACHE 的定义；同步上游后要重新核对
# .github/actions/install/action.yml 里 zig-v8 / v8 两个默认值是否变了）
curl -fL -o .lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a \
  https://github.com/lightpanda-io/zig-v8-fork/releases/download/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a
```

#### 5.1.5 环境变量

```bash
# PATH（写入 ~/.bashrc 持久化）
echo 'export PATH="/usr/local/bin:$HOME/.cargo/bin:$HOME/.local/bin:$PATH"' >> ~/.bashrc
echo 'export LIGHTPANDA_DISABLE_TELEMETRY=1' >> ~/.bashrc
source ~/.bashrc

# Agent 模式（按需）
# export GOOGLE_API_KEY="your-key"
# export GEMINI_API_KEY="your-key"
```

#### 5.1.6 一键初始化脚本

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

### 5.2 首次编译

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

# 启动服务
lightpanda serve --host 0.0.0.0 --port 9222
```

> 本节不包含 `make test`：本分支不维护测试代码（见第三节），测试用例因生产改动而失败属于预期，不作为验收步骤。

### 5.3 增量编译

```bash
# make build    # 默认针对编译机 CPU 优化
# 针对 Intel Skylake-SP (Xeon Platinum) 编译
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"
make clean    # 完全清理（保留 V8 缓存）
```

> **CPU 指令集说明**：编译机为 AMD EPYC 9T25（Zen 5），默认编译会启用 Zen 5 特有指令（如 AVX-512 VNNI/VBMI），这些指令在 Intel Xeon Platinum（Skylake-SP）上不存在，导致运行时 `SIGILL`（非法指令）崩溃。使用 `-Dcpu=skylake_avx512` 确保只生成 Skylake-SP 支持的指令集（AVX-512 F/DQ/CD/BW/VL）。如需兼容更老的 CPU，改用 `-Dcpu=baseline`（任何 x86_64 都能跑，性能损失约 5-10%）。

### 5.4 编译产物

- 可执行文件：`./zig-out/bin/lightpanda`（编译输出）
- 安装位置：`~/.local/bin/lightpanda`（安装后）
- V8 snapshot：`src/snapshot.bin`
- V8 缓存：`.lp-cache/prebuilt-v8/v0.5.4/`

### 5.5 常见问题

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

### 5.6 大陆网络下离线抓取 build.zig.zon 依赖（同步上游后必做）

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

# 3) 预编译 V8 .a（裸二进制，非 zig 包）——走镜像，见 5.1.4 / 5.5

# 全部抓完后验证：应 EXIT=0 且无输出，代表完全离线可解析
zig build --fetch
```

> 校验方法：`zig fetch` 输出的 hash 必须与 `build.zig.zon` 中对应依赖的 `.hash` 完全一致，否则说明字节不同（镜像失效或被篡改）。
> 备选镜像：`https://gh-proxy.com/`（与 `ghfast.top` 等效，可互换）。

---

## 六、Git 分支管理与上游同步方案

### 6.1 仓库来源

当前仓库直接克隆自 `lightpanda-io/browser`（官方上游），所有自定义修改在本地 `chrome` 分支上维护。

```
官方上游  ──→  lightpanda-io/browser（只读，只拉不推）
                                │
                                ▼
                        你本机的 main 分支（跟踪上游，保持纯净）
                                │
                                ├── chrome 分支 ─── 本文档「一、」列出的 8 个生产文件的改动（不碰测试代码）
```

### 6.2 初始化远程（仅首次）

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

### 6.3 创建自定义分支（一次性）

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

### 6.4 日常同步上游流程

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
#    冲突只会出现在本文档「一、修改文件总览」列出的 8 个文件里，不会涉及任何测试文件
#    （测试代码本分支不改动，永远与上游一致，因此天然不会冲突，也不需要关心）：
#    - src/Config.zig              (UA 默认值、brands、sec_ch_ua_* 常量、validateUserAgent、userAgentValidator 提示文案)
#    - src/help.zon                (--user-agent / --user-agent-suffix 帮助文本)
#    - src/network/HttpClient.zig  (baselineHeaders: Sec-Ch-Ua 系 header 不能标 .source=.fixed，见 2.3)
#    - src/browser/webapi/Navigator.zig
#    - src/browser/webapi/NavigatorUAData.zig (uaPlatform / shortBrandList / fullBrandList)
#    - src/browser/webapi/PluginArray.zig
#    - src/browser/webapi/WorkerNavigator.zig (getLanguages 返回类型需跟 Navigator.zig 联动)
#    - src/server/cdp/domains/emulation.zig (setUserAgentOverride 不处理 error.Reserved)

# 5. 编译验证（只验证生产代码，不要求 make test 全绿）
# make build    # 默认针对编译机 CPU
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"

cp -f zig-out/bin/lightpanda ~/.local/bin/lightpanda
~/.local/bin/lightpanda version

# 6. 备份到自己的远程仓库
#    如果用 rebase 了需要 force push
git push origin chrome
```

### 6.5 冲突预防：最小 diff 原则

为了降低每次合并上游时的冲突概率：

1. **只改必要行**：每次修改尽量集中在一两行内，不要大面积重构。
2. **不碰测试代码**：测试文件、`test {}` 块一律不改动、不同步；遇到测试因生产改动而失败也不修（见第三节）。这确保冲突面只有生产代码这 8 个文件，测试相关永远是上游原版，不会成为同步障碍。
3. **记录冲突文件**：对冲突热点了如指掌（就是「一、」里的 8 个文件），合并时直奔目标。

### 6.6 同步节奏建议

| 频率 | 操作 |
|------|------|
| 每周 | `git fetch upstream` 查看上游变动 |
| 有冲突时 | 执行完整 merge/rebase 流程 |
| 上游大版本更新 | 特别注意 build.zig.zon 依赖版本变化 |

### 6.7 如何快速核对当前全部分歧

任何时候都可以用这一条命令拿到当前 `chrome` 分支相对上游最新的完整改动清单，不用翻历史 commit：

```bash
git diff upstream/main..HEAD --stat
```

预期结果应该只有本文档「一」里列出的 8 个文件（外加本文档自己 `CHROME.md`）。如果哪天跑出来多了别的文件（尤其是测试文件），说明有人在测试相关的地方做了不该做的改动，需要按「三、关于测试代码」的原则回退掉。

---

## 七、验证方法

修改完成后，通过以下方式验证（不要求 `make test` 全绿，见第三节，下面只列生产相关验证）：

```bash
# 1. 编译
# make build  # 默认针对编译机 CPU 优化
# 针对 Intel Skylake-SP (Xeon Platinum) 编译
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.4/libc_v8_14.9.207.35_linux_x86_64.a"

# 2. 访问指纹检测网站
lightpanda fetch --url "https://bot.sannysoft.com" --dump html
lightpanda fetch --url "https://abrahamjuliot.github.io/creepjs/" --dump html
lightpanda fetch --url "https://www.browserscan.net/bot-detection" --dump html

# 3. 验证 HTTP 请求头
lightpanda fetch --url "https://httpbin.org/headers" --dump html
# 确认返回的 headers 中包含：
# - User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
# - Sec-Ch-Ua: "Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"
# - Sec-Ch-Ua-Platform: "Windows"
# - Accept-Language: en-US,en;q=0.9
```
