# BeeNet Windows 构建系统分析

> 分析日期：2026-06-04  
> 分析目录：`builder/windows_uncrossed/`  
> 共 11 个 bat 文件，约 37KB

---

## 构建体系总览

```
builder/windows_uncrossed/
│
├── build_beenet_vcpkg.bat          ★ 一键构建入口
│
├── build_beenet/
│   ├── build.bat                    BeeNet 主项目编译 (make + MSVC/clang-cl)
│   └── pkg.bat                      SDK 打包 + 部署到 windows\BeeNet\
│
├── third_party_bat/                 WebRTC 构建相关
│   ├── b.bat                        一键连锁脚本
│   ├── depot_tools_setting.bat      depot_tools 安装配置
│   ├── common_compile_tools.bat     ★ 编译工具链配置 (LLVM-MinGW)
│   ├── common_path_setting.bat      PATH 路径追加工具
│   ├── common_vpn_setting.bat       代理设置 (127.0.0.1:7890)
│   ├── webrtc_clone.bat             WebRTC 源码克隆 (fetch + gclient)
│   └── webrtc_build.bat             WebRTC 编译 (gn + ninja)
│
└── third_party_vcpkg/
    └── build.bat                    第三方库构建 (vcpkg)
```

---

## 1. 工具链依赖清单

### 1.1 编译工具链 (`common_compile_tools.bat`)

```
C:\develop\tools\
├── llvm-mingw-20.1.0\       ★ LLVM-MinGW 20.1.0 (clang-cl + lld-link)
│   ├── bin\clang-cl.exe             C/C++ 编译器
│   ├── bin\lld-link.exe            链接器
│   ├── bin\llvm-lib.exe            静态库管理器 (AR)
│   ├── bin\llvm-rc.exe             资源编译器 (RC)
│   └── bin\llvm-mt.exe             清单工具 (MT)
│
├── make-3.81-bin\            GNU Make 3.81
├── python3\                  Python 3 (gn/ninja 依赖)
├── curl-8.15.0\              curl (下载依赖)
│
├── crt_14.16.27023\           ★ MSVC CRT (vcruntime)
│   ├── include\                    头文件
│   └── lib\x64\                    库文件
│
└── sdk_10.0.26100.0\          ★ Windows SDK 10.0.26100.0
    ├── include\ucrt\               UCRT 头文件
    ├── include\um\                 Win32 API 头文件
    ├── include\shared\             共享头文件
    ├── lib\um\x64\                 Win32 API 库
    └── lib\ucrt\x64\              UCRT 库
```

### 1.2 WebRTC 工具链 (`webrtc_build.bat`)

```
C:\Program Files\Microsoft Visual Studio\2022\Community\
├── VC\Tools\Llvm\            VS2022 自带 Clang 工具链
└── depot_tools\                ★ Google depot_tools (gn + ninja)
```

### 1.3 vcpkg (`third_party_vcpkg/build.bat`)

```
C:\develop\tools\vcpkg\       ★ Microsoft vcpkg 包管理器
├── vcpkg.exe                      主程序
├── installed\x64-windows-beenet-debug\    安装的库(debug)
└── installed\x64-windows-beenet-release\  安装的库(release)
```

### 1.4 代理

所有脚本强制使用代理 `http://127.0.0.1:7890`（Clash/V2Ray 等本地代理），用于：
- WebRTC 源码 `fetch`/`gclient sync`
- vcpkg 下载预编译包

---

## 2. 一键构建入口 (`build_beenet_vcpkg.bat`)

```
build_beenet_vcpkg.bat [--skip-webrtc]
```

**执行流程**（4 步）：

```
Step 1/4: third_party_vcpkg\build.bat       ← 构建 16 个第三方库
Step 2/4: third_party\webrtc\winos\step3_build_webrtc.bat  ← 构建 WebRTC (可选)
Step 3/4: build_beenet\build.bat x86_64     ← 构建 BeeNet 主项目 (debug+release)
Step 4/4: build_beenet\pkg.bat             ← 打包 SDK + 部署到 windows\BeeNet\
```

`--skip-webrtc` 跳过 WebRTC 构建（适合仅修改 HTTP/WS 协议的开发者）。

---

## 3. 第三方库构建 (`third_party_vcpkg/build.bat`)

### 3.1 构建的 16 个库

| 库 | vcpkg 包名 | 功能 | 输出 .lib |
|---|---|---|---|
| zlib | `zlib` | 数据压缩 | `zlib.lib` / `zlibd.lib` |
| c-ares | `c-ares` | 异步DNS | `cares.lib` |
| brotli | `brotli` | Brotli 压缩 | `brotlicommon.lib` `brotlidec.lib` `brotlienc.lib` |
| OpenSSL | `openssl` | TLS/SSL | `libcrypto.lib` `libssl.lib` |
| nghttp3 | `nghttp3` | HTTP/3 | `nghttp3.lib` |
| ngtcp2 | `ngtcp2[openssl]` | QUIC | `ngtcp2.lib` `ngtcp2_crypto_ossl.lib` |
| nghttp2 | `nghttp2` | HTTP/2 | `nghttp2.lib` |
| curl | `curl[websockets]` | HTTP 客户端 | `libcurl.lib` / `libcurl-d.lib` |
| opus | `opus` | Opus 音频 | `opus.lib` |
| fdk-aac | `fdk-aac` | AAC 音频 | `fdk-aac.lib` |
| jsoncpp | `jsoncpp` | JSON 解析 | `jsoncpp.lib` |
| getopt | `getopt-win32` | 命令行参数 | `getopt.lib` |
| xxtea | `xxtea` | XXTEA 加密 | `xxtea.lib` |
| lua | `lua` | Lua 5.x | `lua.lib` |
| x264 | `x264` | H.264 编码 | `libx264.lib` |
| librtmp | `librtmp` | RTMP 协议 | `rtmp.lib` |
| FFmpeg | `ffmpeg[opus,fdk-aac,x264]` | 多媒体框架 | `avcodec.lib` `avformat.lib` 等 7 个 |

### 3.2 构建流程

```
vcpkg install <package> --triplet=x64-windows-beenet-debug
  │
  ├─ 下载源码 (HTTP_PROXY=http://127.0.0.1:7890)
  ├─ CMake 配置
  ├─ MSVC 编译 (debug + release 两版本)
  │    ├─ debug:    installed\x64-windows-beenet-debug\lib\
  │    └─ release:  installed\x64-windows-beenet-release\lib\
  │
  └─ copy_all_libs_to_third_party()
       │
       ├─ 从 vcpkg\installed\ 复制
       ├─ 到 third_party\<lib>\windows\output\<debug|release>\x86_64\
       │
       ├─ 特殊重命名:
       │    libcurl-d.lib → curl.lib (debug)
       │    libcurl.lib   → curl.lib (release)
       │    zlibd.lib     → zlib.lib
       │
       └─ 目录名映射:
            c-ares → cares/
            getopt-win32 → getopt/
```

**输出目录结构**：
```
third_party/curl/windows/output/
├── debug/x86_64/
│   ├── lib/curl.lib
│   └── include/curl/*.h
└── release/x86_64/
    ├── lib/curl.lib
    └── include/curl/*.h
```

### 3.3 自定义 vcpkg triplet

BeeNet 使用自定义 triplet `x64-windows-beenet-debug`（定义在 `third_party_vcpkg/vcpkg/triplets/`）：

- 同时构建 debug 和 release 版本
- 配置 CRT 链接方式（静态/动态）
- 配置运行时库选择

---

## 4. WebRTC 源码克隆与构建

### 4.1 WebRTC 克隆 (`webrtc_clone.bat`)

```
depot_tools_setting.bat  ← 安装/配置 depot_tools
  │
  └─ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
       └─ bootstrap\win_tools.bat  (下载 gn + ninja)

webrtc_clone.bat
  │
  ├─ fetch --nohooks --no-history webrtc  ← 下载 WebRTC 源码 (约 15GB)
  │
  ├─ git checkout branch-heads/6943      ← 切换到指定 M133 分支
  │
  └─ gclient sync --force -D -v            ← 同步依赖 (abseil, libyuv, perfetto...)
```

**关键配置**：
- WebRTC 分支：`branch-heads/6943` (M133)
- 工作目录：`c:\develop\webrtc_m133\`
- 代理：`http://127.0.0.1:7890`（必须，Google 源不可直连）

### 4.2 WebRTC 构建 (`webrtc_build.bat`)

```
gn gen output_example/<build>/<arch> \
  --args="is_debug=true/false \
          symbol_level=2/0 \
          target_os=\"win\" \
          target_cpu=\"x64\" \
          rtc_use_perfetto=false \
          rtc_include_tests=true \
          rtc_build_examples=true"

ninja -C output_example/<build>/<arch> -v \
  peerconnection_client peerconnection_server \
  async_echo_client async_echo_server
```

**编译配置**：
- 工具链：VS2022 内置 LLVM（`C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\Llvm`）
- 构建系统：`gn` (生成) + `ninja` (编译)
- 产物：`peerconnection_client` 等示例程序

**注意**：WebRTC 的 `.lib` 静态库输出路径为 `third_party/webrtc/winos/src/output/<build>/<arch>/obj/`，而非独立的 `output/` 目录。这与 `pkg.bat` 中的复制路径一致。

---

## 5. BeeNet 主项目构建 (`build_beenet/build.bat`)

```
build.bat x86_64       ← 构建 x64 (debug + release)
build.bat x86          ← 构建 x86 (debug + release)
build.bat clean        ← 清理构建产物
```

**执行流程**：

```
make.exe clean                              ← 清理
make.exe all ARCH=x86_64 BUILD=debug         ← Debug 构建 (SANITIZE=1)
make.exe all ARCH=x86_64 BUILD=release       ← Release 构建
pkg.bat                                     ← 打包部署
```

**工具**：GNU Make 3.81 (`c:\develop\tools\make-3.81-bin\bin\make.exe`)

**Makefile 变量**：
- `ARCH`：x86 / x86_64
- `BUILD`：debug / release
- `SANITIZE=1`：Debug 构建启用 AddressSanitizer

**产物**：
- `build_beenet/build/debug/x86_64/beenet.lib`
- `build_beenet/build/release/x86_64/beenet.lib`
- `build_beenet/build/debug/x86_64/luacjson.lib`
- `build_beenet/build/release/x86_64/luazlib.lib`

---

## 6. SDK 打包 (`build_beenet/pkg.bat`)

```
pkg.bat        ← 仅打包到 output\ 和 windows\BeeNet\
pkg.bat zip    ← 打包 + 创建 ZIP 归档
pkg.bat clean  ← 清理 output\
```

### 6.1 收集的文件

**BeeNet 自有产物**：
```
output/
├── include/beenet/
│   ├── interface.h
│   ├── interface.hpp
│   ├── asyncworker.hpp
│   ├── logger.h
│   ├── iobuffer.h
│   ├── tokenizer.h
│   ├── external_video.h
│   └── external_audio.h
│
└── lib/
    ├── debug/
    │   ├── x86_64/beenet.lib luacjson.lib luazlib.lib
    │   └── x86/beenet.lib luacjson.lib luazlib.lib
    └── release/
        ├── x86_64/beenet.lib luacjson.lib luazlib.lib
        └── x86/beenet.lib luacjson.lib luazlib.lib
```

**第三方库**（从 `third_party/<lib>/windows/output/<BUILD>/<ARCH>/lib/` 复制）：

| 库 | 重命名规则 |
|---|---|
| curl | `libcurl-d.lib` → `curl.lib` (debug) |
| openssl | `libcrypto.lib` `libssl.lib` (保持) |
| zlib | `zlibd.lib` → `zlib.lib` (debug) |
| x264 | `libx264.lib` (保持) |
| 其他 | 按原名复制 |

**WebRTC 库**（从 `third_party/webrtc/winos/src/output/<BUILD>/<ARCH>/obj/` 复制所有 `*.lib`）

**头文件**：
- `third_party/webrtc/winos/src/third_party/libyuv/include/` → libyuv
- `third_party/ffmpeg/windows/output/release/x86_64/include/` → FFmpeg (用 release 版)
- `third_party/jsoncpp/windows/output/release/x86_64/include/` → jsoncpp (用 release 版)

### 6.2 部署目标

```
output\  → 同步到 → windows\BeeNet\
                        ├── include\  (覆盖)
                        └── lib\      (覆盖)
```

`windows\BeeNet\` 是发布 Demo 项目的 include/lib 目录。

### 6.3 ZIP 打包

当 `pkg.bat zip` 时：
- 文件名格式：`<YYMMDD>-<commit_hash>.zip`
- 内容：`output/include\` + `output/lib\`

---

## 7. WebRTC 一键脚本 (`third_party_bat/b.bat`)

简单的连锁调用：
```
depot_tools_setting.bat → webrtc_clone.bat → webrtc_build.bat
```

---

## 8. 构建工具链对比

| 组件 | 工具 | 路径 |
|---|---|---|
| **BeeNet 主项目** | GNU Make 3.81 + clang-cl | `c:\develop\tools\` |
| **第三方库** | vcpkg + MSVC | `c:\develop\tools\vcpkg\` |
| **WebRTC** | gn + ninja + VS2022 LLVM | `c:\develop\webrtc_m133\depot_tools\` |

**注意**：BeeNet 主项目使用 `llvm-mingw-20.1.0` 的 `clang-cl.exe`，而 WebRTC 使用 VS2022 内置的 LLVM 工具链。两者是不同的 Clang 发行版，各自有独立的 CRT/SDK 路径配置。

---

## 9. 首次构建完整步骤

```bash
# 1. 准备工具链目录结构
mkdir c:\develop\tools
# 下载并解压到对应目录:
#   llvm-mingw-20.1.0, make-3.81-bin, python3, curl-8.15.0
#   crt_14.16.27023, sdk_10.0.26100.0
#   vcpkg (从 github.com/microsoft/vcpkg clone)

# 2. 配置环境变量 (可选，脚本自动处理)
# VCPKG_ROOT = c:\develop\tools\vcpkg

# 3. 构建第三方库
cd builder\windows_uncrossed\third_party_vcpkg
build.bat          # 构建全部 16 个库 (约 30-60 分钟)

# 4. 克隆并构建 WebRTC (可选)
cd builder\windows_uncrossed\third_party_bat
depot_tools_setting.bat   # 安装 depot_tools
webrtc_clone.bat          # 克隆 WebRTC (约 15GB, 需代理)
webrtc_build.bat          # 编译 WebRTC (约 2-4 小时)

# 5. 一键构建 BeeNet
cd builder\windows_uncrossed
build_beenet_vcpkg.bat    # 或 build_beenet_vcpkg.bat --skip-webrtc

# 6. 产物位置
build_beenet\output\                      # 打包的 SDK
windows\BeeNet\include\                   # 部署的头文件
windows\BeeNet\lib\                       # 部署的库文件
```

---

## 10. 设计要点

| 要点 | 说明 |
|---|---|
| **三套构建系统** | Make + vcpkg/CMake + gn/ninja，分别管理 BeeNet、第三方库、WebRTC |
| **LLVM 工具链** | BeeNet 主项目使用 llvm-mingw clang-cl，WebRTC 使用 VS2022 LLVM |
| **手动 CRT/SDK** | BeeNet 不使用 VS2022 自动检测，手动指定 CRT 14.16 和 SDK 10.0.26100 |
| **vcpkg triplet** | 自定义 `x64-windows-beenet-debug` 控制编译选项 |
| **库重命名** | pkg.bat 将 vcpkg 输出名（如 libcurl-d.lib）统一为 BeeNet 命名（curl.lib） |
| **代理依赖** | WebRTC 和 vcpkg 下载均需代理（127.0.0.1:7890） |
