# 移植技术说明（PORT-NOTES）

面向维护者与贡献者的技术文档：改了什么、为什么这么改、如何与上游同步。

## 移植架构总览

Windows 移植遵循一条铁律：**所有平台差异走 `comptime` 门控**（`if (builtin.os.tag == .windows)`），非 Windows 编译路径逐字节不变。这使得同一个树同时服务 Windows 构建与上游 Linux CI（posix-compat 工作流的回归保护）。

### 改动分层（27 个提交，`git diff 219f290..main --stat` 为准）

```
build.zig                    构建系统：动态 MSVC/SDK 发现、Windows 系统库、
                             rust gnu target、cargo 依赖接线、V8 链接路径映射
src/sys/net.zig              socket I/O 的 Winsock 分支（send/recv/accept/
                             closesocket + WSA 错误映射）
src/sys/libcurl.zig          CURL_GLOBAL_WIN32（WSAStartup 根因修复）
src/server/Server.zig        WindowsIO：select() 事件循环 + loopback 唤醒通道
                             （填上游预留的 WindowsIO 占位）
src/server/Link.zig          非阻塞语义注释（生产加固跟进项）
src/server/{http,websocket}  混合分隔符清理
src/crash_handler.zig        fork 上传的 Windows 早返回
src/agent/*                  终端输入（GetConsoleMode）、process args（initAllocator）、
                             environ（Win32 API）、时间（桥接）
src/browser/{tools,js/bridge} 环境变量与 localtime 的平台分支
src/lightpanda.zig, cli.zig, Config.zig, core_dump.zig  同类
win-port/shims.c             链接期垫片（__imp_wcstold、TLS guard、localtime_s）
win-port/filtered-libs/      msvcrt.lib 符号手术版（见下）
win-port/windows-v8-compat.py V8 源码树 post-sync 补丁（随 zig-v8-fork 分发）
.gitattributes               强制 LF（Windows checkout 破坏 zig fmt --check）
.github/workflows/windows-portable.yml  全流程构建工作流
```

### 关键设计决策

**1. rust 强制 gnu 目标（msvc host 的坑）**
CI runner 的 rustup 默认 x86_64-pc-windows-msvc——其 compiler_builtins 发出
`.weak.__netf2.default` 形态的弱符号，与 zig 的 compiler_rt 在链接期冲突
（14 个重复符号）。强制 `--target x86_64-pc-windows-gnu` 后与本地（gnu host）
完全同构，zig 的 mingw 模式链接器自动去重弱符号。

**2. cargo 显式执行依赖**
`addPrefixedOutputDirectoryArg` 声明的目录输出在 zig 0.16 中不保证 cargo 步骤
先于消费它的链接步骤执行（全新克隆首次链接时 file not found；有缓存的树永远
不触发）。修复：linkRust 返回 cargo 的 Run step，所有消费方（exe/check_lib/
tests）显式 `dependOn`。产物路径弃用 `--target-dir` 花活，回归 cargo 默认的
workspace target 目录（`src/rust/target/<triple>/release/`）。

**3. msvcrt.lib 符号手术**
zig（mingw CRT）与 MSVC 的 msvcrt 导入库混链时，后者的 TLS/CFG 支持对象
（`__dyn_tls_init*`、`__guard_check_icall_fptr`…）与前者的同名定义冲突。
解法：解析导入库格式，按符号名剔除冲突成员后重建归档。手术版随仓库分发
（`win-port/filtered-libs/msvcrt.lib`），可用 `LP_FILTERED_LIBS` 覆盖。
合规说明：msvcrt.dll 是 Windows 系统组件，手术版仅去除重复符号定义、
不改变任何导入语义；如维护者偏好，可切换为 `.def` + lib.exe 生成流程。

**4. CURL_GLOBAL_WIN32**
lightpanda 调 `curl_global_init(CURL_GLOBAL_SSL)`——该标志组合抑制了 libcurl
的 WSAStartup 兜底，导致整个进程的 Winsock 未初始化（所有 socket/getaddrinfo
返回 WSAENOTINITIALISED）。加 `CURL_GLOBAL_WIN32` 后一切网络操作恢复。
这是"所有网络操作全挂"这类系统性症状的单点根因。

**5. WindowsIO select 引擎**
上游在 Server.zig 预留的 WindowsIO 占位（注释："the epoll/kqueue engines
have no equivalent there yet"）。实现：select() + 注册表（1024 连接 + 唤醒
通道）+ loopback TCP socketpair 唤醒通道（替代 eventfd/signal）+ owner
哨兵编码（LISTENER/SHUTDOWN/SIGNAL）。语义与 epoll 引擎镜像——但注意：
病态客户端理论上可阻塞 select 循环（epoll 不会），生产加固跟进项见
BUILD-WINDOWS.md 的"已知限制"。

## 依赖 fork 策略

三个上游依赖在 Windows 上编不过，补丁以 fork 分发（`build.zig.zon` 指向
fork 的 archive URL + 内容哈希校验）：

| 依赖 | 上游 pin | fork 补丁 |
|---|---|---|
| [boringssl-zig](https://github.com/lightpanda-io/boringssl-zig) | 5b5ad66f | wincrypt.h 宏污染（NOCRYPT/NOGDI）+ OPENSSL_NO_ASM（无 NASM） |
| [zig-v8-fork](https://github.com/lightpanda-io/zig-v8-fork) | 200123d4 | Windows 构建编排全量化（+259 行）+ V8 树 post-sync 补丁 |
| [zenai](https://github.com/lightpanda-io/zenai) | b3386fde | std.process.Environ 的 Win32 API（上游调用在 zig 0.16 Windows std 上触发编译错误） |

上游同步时：重新 rebase 三个 fork 的 windows-port 分支 → zig fetch 取新哈希 →
更新 zon。**向上游提 PR 时**保持上游 pin 并在描述中披露 fork 依赖（范式见
lightpanda-io/browser#3753）。

## 上游同步流程

```bash
# 1. 拉上游
git fetch --depth=200 upstream main

# 2. 检查漂移（重点：我们改过的文件）
git diff --stat 219f290 FETCH_HEAD -- src/ build.zig

# 3. 合并上游到 main（冲突集中在 src/server/Server.zig 等）
git merge FETCH_HEAD

# 4. 全量回归（Windows + 依赖 fork 的 windows-port 分支需同步 rebase）
zig build -Doptimize=ReleaseFast -Dcpu=x86_64

# 5. 发布
git tag portable-vX.Y.Z && git push origin portable-vX.Y.Z
```

漂移监控经验：上游日均 10+ commit；我们改过的核心文件
（Server.zig/net.zig/libcurl.zig/target.zig/CDP.zig）历史上单周漂移 <10 行，
rebase 成本可控。

## 上游 PR 状态

| PR | 内容 | 状态 |
|---|---|---|
| [browser#3753](https://github.com/lightpanda-io/browser/pull/3753) | Windows 原生支持（本仓库 main 的移植内容） | open，CLA 已签 |
| [browser#3752](https://github.com/lightpanda-io/browser/pull/3752) | CDP setAutoAttach 尊重客户端 filter（Puppeteer pages() 挂死根因） | open，CLA 已签 |
| [zig-v8-fork#219](https://github.com/lightpanda-io/zig-v8-fork/pull/219) | Windows 构建编排（#3753 的使能器） | open，mergeable clean |

## 已知限制与后续路线

1. **WindowsIO 生产加固**：accept 后 FIONBIO + WouldBlock 破循环（当前阻塞
   accept 与读路径）；WSAStartup 显式顺序化。面向公网长时服务前必修。
2. **CIPD latest 浮动**：gn/ninja 从 chrome-infra-packages 拉 latest（无版本
   锁定）——可复现性改进项。
3. **msvcrt.lib 生成流程**：当前为手术版二进制入库；候选改进为 .def 文件 +
   lib.exe 生成脚本。
4. **win-port/patches/orig/**：30 个补丁前备份（fork 维护产物），可归档清理。
