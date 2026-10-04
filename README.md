# Lightpanda Browser — Windows 原生移植

<p align="center">
<strong>非 Chromium 分支 · 非 WebKit 补丁 · 用 Zig 从零写的新一代无头浏览器</strong><br>
官方支持 Linux/macOS —— 本仓库提供 <b>Windows x64 原生构建</b>（社区移植，AGPL-3.0）
</p>

> 上游项目：[lightpanda-io/browser](https://github.com/lightpanda-io/browser)（上游说明见 [UPSTREAM-README.md](UPSTREAM-README.md)）

## 这是什么

Lightpanda 是专为 AI Agent 与自动化打造的无头浏览器：**16 倍省内存、9 倍快于 Chromium**，单二进制零依赖。本仓库把它的**原生 Windows 支持**从零实现并维护：

- ✅ `fetch`（DNS+TLS+DOM+Markdown/HTML/PNG/PDF 导出）
- ✅ `serve`（CDP 服务，Puppeteer/Playwright 直连）
- ✅ Puppeteer 自动化全链路实测（导航/JS/表单/搜索/截图）
- ✅ 20 并发压测 / 错误路径 / 全新克隆构建 全部通过
- ✅ GitHub Actions 一键产包（打 `portable-v*` 标签自动构建并发布 Release）

## 快速开始

### 下载现成包
[Releases](https://github.com/qidiai/lightpanda-windows-port/releases) → 下载 `lightpanda-portable-windows-x64.zip` → 解压即用（免安装，含 VC 运行库）。

```cmd
lightpanda.exe version
lightpanda.exe fetch https://www.baidu.com --dump markdown
lightpanda.exe serve --port 9222        :: CDP 服务，Puppeteer 连 ws://127.0.0.1:9222
lightpanda.exe fetch --help             :: 完整参数
```

访问外网需代理时先设 `HTTP_PROXY` / `HTTPS_PROXY` 环境变量。

### 从源码构建
见 [docs/BUILD-WINDOWS.md](docs/BUILD-WINDOWS.md)（前置条件 + 一条命令 + 15 轮 CI 踩坑全记录）。

## 文档

| 文档 | 内容 |
|---|---|
| [docs/BUILD-WINDOWS.md](docs/BUILD-WINDOWS.md) | Windows 构建完全指南（本机/CI + 故障排查手册） |
| [docs/PORT-NOTES.md](docs/PORT-NOTES.md) | 移植技术架构、逐文件改动说明、依赖 fork 策略、上游同步流程 |
| [UPSTREAM-README.md](UPSTREAM-README.md) | 上游项目原始说明 |

## 状态

| 工作流 | 状态 |
|---|---|
| windows-portable（Windows 全流程构建+发布） | [![windows-portable](https://github.com/qidiai/lightpanda-windows-port/actions/workflows/windows-portable.yml/badge.svg)](https://github.com/qidiai/lightpanda-windows-port/actions/workflows/windows-portable.yml) |
| posix-compat（Linux 回归保护） | [![posix-compat](https://github.com/qidiai/lightpanda-windows-port/actions/workflows/posix-compat.yml/badge.svg)](https://github.com/qidiai/lightpanda-windows-port/actions/workflows/posix-compat.yml) |
| zig-test | [![zig-test](https://github.com/qidiai/lightpanda-windows-port/actions/workflows/zig-test.yml/badge.svg)](https://github.com/qidiai/lightpanda-windows-port/actions/workflows/zig-test.yml) |

## 上游贡献

Windows 支持已向上游提交评审：
- [browser#3753](https://github.com/lightpanda-io/browser/pull/3753) — feat(build): native Windows support
- [browser#3752](https://github.com/lightpanda-io/browser/pull/3752) — fix(cdp): honor client target filters in setAutoAttach
- [zig-v8-fork#219](https://github.com/lightpanda-io/zig-v8-fork/pull/219) — feat(build): Windows 构建编排（浏览器 PR 的使能器）

## 协议

AGPL-3.0（与上游一致）。Windows 移植补丁同协议开源于本仓库。
