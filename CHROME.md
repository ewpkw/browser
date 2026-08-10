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

**文件**: `src/cdp/domains/emulation.zig` 第 157-164 行

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

### 1.4 将 CDP 测试用例从「拒绝 Mozilla」反转为「接受 Mozilla」

**文件**: `src/cdp/domains/network.zig`

将上游的 Mozilla 拒绝测试反转为正向验证（本分支的核心意义就是允许 Mozilla UA）：
- `test "cdp.network setExtraHTTPHeaders rejects a Mozilla User-Agent"` → 改为 `accepts a Mozilla User-Agent`，期望 `extra_headers.items.len == 1`
- `test "...rejects a Mozilla User-Agent smuggled via a colon in the key"` → 改为 `accepts a Mozilla User-Agent with colon in key`，期望 `extra_headers.items.len == 1`
- `test "...rejects a header that smuggles CRLF"` → **保留测试但替换载荷**，将 `Mozilla/5.0` 改为 `CustomBot/1.0`（该测试核心是 CRLF 注入防护，与 Mozilla 无关）

**文件**: `src/cdp/domains/emulation.zig`

将上游的 2 个 Mozilla 拒绝测试合并为 1 个正向测试：
- 删除: `test "...ignores mozilla"` 和 `test "...ignores mozilla case insensitive"`
- 新增: `test "cdp.Emulation: setUserAgentOverride accepts Mozilla user agent"`，期望 `user_agent_changed == true`

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

**文件**: `src/Config.zig` 第 666-668 行

```zig
// 修改前
pub const brands = [_]Brand{
    .{ .brand = "Lightpanda", .version = "1" },
};

// 修改后（顺序、品牌名、版本号均与真实 Edge 151 一致）
pub const brands = [_]Brand{
    .{ .brand = "Not=A?Brand", .version = "99" },
    .{ .brand = "Microsoft Edge", .version = "151" },
    .{ .brand = "Chromium", .version = "151" },
};
```

> 真实 Edge 151 的 Sec-Ch-Ua：`"Not=A?Brand";v="99", "Microsoft Edge";v="151", "Chromium";v="151"`
> **注意**：品牌名是 `Not=A?Brand`（等号和问号），不是 `Not-A.Brand`。顺序也很重要。

这会同时影响：
- HTTP 头 `Sec-Ch-Ua` 的值（通过 `sec_ch_ua` 常量自动生成）
- `navigator.userAgentData.brands` JS API 返回值（通过 `brandList()` 引用 `brands`）
- `navigator.userAgentData.getHighEntropyValues()` 返回值

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

> **架构变更说明**：上游重构了 header 管理，删除了 `http.zig` 中的 `Headers` struct，改为在 `HttpClient.zig` 中通过 `baselineHeaders()` 返回默认 header 数组。Client Hints 现在在此处添加。

将原来的：
```zig
pub fn baselineHeaders(self: *const Client) [3]http.Header {
    return .{
        .{ .name = "User-Agent", .value = self.getUserAgent() },
        .{ .name = "Sec-Ch-Ua", .value = lp.Config.HttpHeaders.sec_ch_ua },
        .{ .name = "Accept-Language", .value = lp.Config.HttpHeaders.accept_language },
    };
}
```

替换为：
```zig
pub fn baselineHeaders(self: *const Client) [8]http.Header {
    return .{
        .{ .name = "User-Agent", .value = self.getUserAgent() },
        .{ .name = "Sec-Ch-Ua", .value = lp.Config.HttpHeaders.sec_ch_ua },
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

> **注意**：数组大小从 `[3]` 扩展为 `[8]`。当 Chrome 版本升级时，修改对应值即可。

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

### 5.3 更新 testing.zig 中的测试配置

**文件**: `src/testing.zig` 第 527 行

```zig
// 修改前
.user_agent_suffix = "internal-tester",

// 修改后 (保持测试可识别，但基于 Chrome UA)
.user_agent_suffix = null,
// 或者直接删除该字段使用默认 UA
```

---

## 六、修改文件清单汇总

| # | 文件 | 修改类型 | 说明 |
|---|------|---------|------|
| 1 | `src/Config.zig` | 修改 | 默认 UA、brands、删除 validateUserAgent Mozilla 检测、添加 CH 常量 |
| 2 | `src/network/HttpClient.zig` | 修改 | baselineHeaders 添加 5 个 Client Hints headers（[3]→[8]） |
| 3 | `src/browser/webapi/Navigator.zig` | 修改 | appVersion、vendor、hardwareConcurrency、deviceMemory、maxTouchPoints、doNotTrack、language/languages、platform |
| 4 | `src/browser/webapi/NavigatorUAData.zig` | 修改 | platform="Windows"、高熵值匹配 Chrome |
| 5 | `src/browser/webapi/PluginArray.zig` | 修改 | length=5 (非零) |
| 6 | `src/cdp/domains/emulation.zig` | 修改+反转测试 | 删除 Mozilla 拒绝逻辑；反转测试为"accepts Mozilla" |
| 7 | `src/cdp/domains/network.zig` | 反转测试+改载荷 | 反转 Mozilla UA 拒绝测试为接受；CRLF 测试替换载荷 |
| 8 | `src/help.zon` | 修改 | 更新帮助文本（user-agent + user-agent-suffix） |
| 9 | `src/browser/tests/navigator/navigator.html` | 修改 | 更新期望值 |
| 10 | `src/testing.zig` | 修改 | 更新测试 UA 配置 |

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
# 或手动下载 v0.5.2 版本
curl -fL -o .lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a \
  https://github.com/lightpanda-io/zig-v8-fork/releases/download/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a
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
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a"

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
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a"

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
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a"
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
- V8 缓存：`.lp-cache/prebuilt-v8/v0.5.2/`

### 8.5 常见问题

```bash
# html5ever 编译失败 → 检查 Rust
cargo --version

# Zig 版本不对
zig version  # 必须 0.16.0

# V8 下载失败 → 手动下载后放入缓存
mkdir -p .lp-cache/prebuilt-v8/v0.5.2/
curl -fL -o .lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a \
  https://github.com/lightpanda-io/zig-v8-fork/releases/download/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a
```

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
#    - src/cdp/domains/emulation.zig
#    - src/cdp/domains/network.zig

# 5. 编译验证（确认合并无误）
# make build && make test  # 默认针对编译机 CPU
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a"

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

---

## 十、验证方法

修改完成后，通过以下方式验证：

```bash
# 1. 编译
# make build  # 默认针对编译机 CPU 优化
# 针对 Intel Skylake-SP (Xeon Platinum) 编译
make build ZIGFLAGS="-Dcpu=skylake_avx512 -Dprebuilt_v8_path=.lp-cache/prebuilt-v8/v0.5.2/libc_v8_14.9.207.35_linux_x86_64.a"

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
