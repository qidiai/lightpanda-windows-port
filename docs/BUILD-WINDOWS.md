# Windows 构建完全指南

从零在 Windows x64 上构建 Lightpanda 原生版。本文档是 15 轮 CI 迭代的完整沉淀——每一节标题都是真实踩过的坑。

## 前置条件

| 工具 | 版本 | 说明 |
|---|---|---|
| [Zig](https://ziglang.org/download/) | **0.16.0**（精确版本） | 编译器 + 构建系统 |
| Visual Studio | 2022 BuildTools 或 2026 Enterprise（含 C++ 工作负载） | 仅需头文件/导入库；`winget install Microsoft.VisualStudio.2022.BuildTools --override "--add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"` |
| Windows SDK | 10.0.26100.0 或更高 | 随 VS BuildTools 安装 |
| Rust（gnu 工具链） | stable + `rustup target add x86_64-pc-windows-gnu` | src/rust/ffi 静态库 |
| Git for Windows | 任意 | **`usr\bin` 里的 cp/touch/sh 会在 bootstrap 用到** |
| Python 3.10+ | 任意 | depot_tools 脚本 |

> 不需要 NASM（boringssl 走 OPENSSL_NO_ASM）、不需要手动 CIPD（脚本自动处理 gn/ninja）。

## 一条命令构建

```powershell
$env:PATH = "<zig目录>;C:\Program Files\Git\usr\bin;$env:PATH"
$env:DEPOT_TOOLS_UPDATE = '0'
$env:DEPOT_TOOLS_WIN_TOOLCHAIN = '0'
$env:GYP_MSVS_VERSION = '2022'
zig build -Doptimize=ReleaseFast -Dcpu=x86_64
```

首次构建约 1.5-2.5 小时（V8 从源码全编：depot_tools 引导 → gclient sync → chromium clang 下载 → gn gen → ninja 2301 目标 → c_v8.lib → 链接）。之后增量构建 2-10 分钟。

产物：`zig-out\bin\lightpanda.exe`（约 78-82MB，基线 x86_64 指令集，任意 x64 机器可运行）。

## 环境变量

| 变量 | 作用 | 默认 |
|---|---|---|
| `LP_MSVC_LIB_X64` | 显式指定 MSVC lib\x64 目录（绕过自动发现） | 自动遍历 VS 安装 |
| `LP_WINSDK_DIR` | 显式指定 Windows Kits\10\Lib 根 | 自动遍历取最新 |
| `LP_FILTERED_LIBS` | 手术版 msvcrt.lib 所在目录 | `win-port/filtered-libs`（随仓库分发） |

## 可选构建参数

| 参数 | 说明 |
|---|---|
| `-Dcpu=x86_64` | 基线指令集（**分发必加**——默认按本机 CPU 生成指令，新机器构建的产物旧机器会崩溃 0xC000001D） |
| `-Doptimize=ReleaseFast` | 发布优化（Debug 模式惰性分析会漏掉部分编译错误，分发一律 ReleaseFast） |
| `-Dprebuilt_v8_path=<path>` | 使用预编译 V8 静态库（Linux 有官方预编译；Windows 无，必须源码构建） |

## CI 构建

打 `portable-v*` 标签即可：

```bash
git tag portable-v0.2.4 && git push origin portable-v0.2.4
```

GitHub Actions 自动执行全流程并发布 Release（约 2-2.5 小时，windows-latest runner）。

## 故障排查手册（15 轮 CI 实战全记录）

按构建阶段排列，每条都是真实撞过的墙：

### 依赖获取阶段
| 症状 | 根因 | 解法 |
|---|---|---|
| `HttpConnectionClosing` 拉不上 GitHub | zig 0.16 HTTP 客户端不走代理且 TLS 指纹被干扰 | 用 curl/系统工具下载 tar.gz，`zig fetch <本地文件>` 灌包；**必须在工程根目录内跑 fetch** |
| `failed to create temporary zip file: FileNotFound` | zig 0.16 处理 .zip 依赖有 bug（sqlite amalgamation 等 zip 源） | curl 下载 → tar 解包 → `zig fetch <解包目录>` |
| `Failed to lock handle` / gsutil 锁错误 | 上次中断的 .locked 残留 | 删除 external_bin/gsutil 重跑 |
| `Invalid command "GSUtil:software_update_check_period"` | gsutil 5.x 移除了该选项，depot_tools 包装器还在传 | 补丁 gsutil.py 清空 args_opt |
| `'vpython3.bat' 不是内部或外部命令` | depot_tools 子进程需要自身目录在 PATH | PATH 前置 depot_tools 目录 |

### 工具链阶段
| 症状 | 根因 | 解法 |
|---|---|---|
| `MsvcLibX64NotFound` | build.zig 硬编码 MSVC 版本路径，而 runner/本机版本不同 | 已改为动态遍历所有 VS 版本根（build.zig L386-432） |
| `Path "...10.0.28000.0\um" does not exist` | V8 树 `vs_toolchain.py` 硬编码 SDK 28000（VS2026 时代），本机装 26100 | `windows-v8-compat.py` post-sync 补丁自动替换 |
| `dbghelp.dll not found` | Debugging Tools for Windows 未装 | vs_toolchain 已补丁为可选 |
| `error: unknown argument '-fvisibility=default'`（clang-cl） | BUILD.gn 给 clang-cl 传 GCC 风格参数 | `windows-v8-compat.py` 已补 `/clang:` 前缀 |
| `NTDDI_WIN11_BR` 未定义 | 26100 SDK 无此宏（28000 才有） | `windows-v8-compat.py` 已降级为 NTDDI_WIN10_NI |
| `Invalid token 锘?`（gn gen 崩） | PowerShell `Set-Content -Encoding UTF8` 给 BUILD.gn 写了 BOM | 用 `[IO.File]::WriteAllText + UTF8Encoding($false)` 写 GN 文件 |

### 链接阶段
| 症状 | 根因 | 解法 |
|---|---|---|
| `could not open 'liblibcmt.a'` | V8 默认 /MT 静态 CRT，与 zig mingw 的动态 msvcrt 冲突 | gn args 强制 V8 动态 CRT（dynamic_crt） |
| `duplicate symbol: __guard_check_icall_fptr` | V8 的 /guard:cf 与 zig CRT 都定义 CFG 符号 | gn args 禁用 /guard:cf |
| `duplicate symbol: __dyn_tls_init*` | msvcrt.lib 导入库的 TLS 支持对象与 zig CRT 冲突 | `win-port/filtered-libs/msvcrt.lib`（符号过滤手术版，随仓库分发） |
| `liblightpanda_ffi.a: file not found` | cargo 步骤与链接之间无执行依赖；msvc host rust 产出 `lightpanda_ffi.lib` 而非 `.a` | build.zig 已接显式 dependOn + rust 强制 gnu target + normalize 步 |
| `duplicate symbol: .weak.__netf2.default` | msvc host rust 的 compiler_builtins 与 zig compiler_rt 冲突 | rust 强制 gnu target 后消失 |

### 运行阶段
| 症状 | 根因 | 解法 |
|---|---|---|
| 所有网络操作 `CouldntResolveHost`/`CouldntConnect` | `CURL_GLOBAL_SSL` 抑制了 curl 的 WSAStartup 兜底 → Winsock 未初始化 | libcurl.zig 加 `CURL_GLOBAL_WIN32` |
| 退出码 `-1073741795`（0xC000001D） | 新 CPU 机器构建的 exe 在旧 CPU 上跑 | `-Dcpu=x86_64` 基线构建 |
| exe 在 CI 构建后本机崩溃 | 同上（runner CPU 更新） | 同上 |

## 依赖说明

Windows 构建依赖三个**补丁版依赖 fork**（标准上游依赖在 Windows 上编不过）：

| 依赖 | fork | 补丁 |
|---|---|---|
| boringssl-zig | [qidiai/boringssl-zig](https://github.com/qidiai/boringssl-zig) | `-DNOCRYPT -DNOGDI`（wincrypt.h 宏污染）+ `-DOPENSSL_NO_ASM`（Windows 无 NASM 时走纯 C） |
| zig-v8-fork | [qidiai/zig-v8-fork](https://github.com/qidiai/zig-v8-fork) | Windows 构建编排全量化（见 build.zig +259 行）+ V8 树 post-sync 补丁脚本 |
| zenai | [qidiai/zenai](https://github.com/qidiai/zenai) | `std.process.Environ` 的 Windows API 修复 |

`build.zig.zon` 在本仓库 main 分支已指向这些 fork（带内容哈希校验）。**向上游提 PR 时**请保持上游 pin 并在 PR 描述中披露 fork 依赖（参见 lightpanda-io/browser#3753 的处理方式）。

## 已知限制

- serve 模式事件循环为 select() 实现（FD 上限 1024 连接 + 唤醒通道），高并发场景待上游原生支持
- 慢客户端理论上可阻塞事件循环（读路径短读启发式 + 写路径大响应）——生产公网暴露前建议加 FIONBIO
- 测试数据 hooks 未跑（wasm-spec-tests 等纯测试资产，不影响构建产物）
