<div align="center">

<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://media.x.ai/v1/website/spacexai-symbol-white-transparent-0c31957f.png">
    <source media="(prefers-color-scheme: light)" srcset="https://media.x.ai/v1/website/spacexai-symbol-black-transparent-6435cf42.png">
    <img alt="SpaceXAI logo" src="https://media.x.ai/v1/website/spacexai-symbol-black-transparent-6435cf42.png" width="96">
  </picture>
  <br>
  Grok Build (<code>grok</code>)
</h1>

**Grok Build** is SpaceXAI's terminal-based AI coding agent. It runs as a
full-screen TUI that understands your codebase, edits files, executes shell
commands, searches the web, and manages long-running tasks — interactively,
headlessly for scripting/CI, or embedded in editors via the Agent Client
Protocol (ACP).

[Installing the released binary](#installing-the-released-binary) ·
[Building from source](#building-from-source) ·
[Documentation](#documentation) ·
[Repository layout](#repository-layout) ·
[项目导读与阅读路径](#project-reading-guide) ·
[Development](#development) ·
[Contributing](#contributing) ·
[License](#license)

![Grok Build TUI](https://media.x.ai/v1/website/universe-tui-screenshot-6f7a0837.png)

**Learn more about Grok Build at [x.ai/cli](https://x.ai/cli)**

This repository contains the Rust source for the `grok` CLI/TUI and its agent
runtime. It is synced periodically from the SpaceXAI monorepo.

A small `SOURCE_REV` file at the root records the full monorepo commit SHA
for the version of the code present in this tree.

</div>

---

## Installing the released binary

Prebuilt binaries are published for macOS, Linux, and Windows:

```sh
curl -fsSL https://x.ai/cli/install.sh | bash   # macOS / Linux / Git Bash
irm https://x.ai/cli/install.ps1 | iex          # Windows PowerShell
grok --version
```

See the [changelog](https://x.ai/build/changelog) for the latest fixes,
features, and improvements in each release.

## Building from source

Requirements:

- **Rust** — the toolchain is pinned by [`rust-toolchain.toml`](rust-toolchain.toml);
  `rustup` installs it automatically on first build.
- **[DotSlash](https://dotslash-cli.com)** — required so hermetic tools under
  [`bin/`](bin/) (notably [`bin/protoc`](bin/protoc)) can download and run.
  Install it and ensure `dotslash` is on your `PATH` **before** building:

  ```sh
  cargo install dotslash
  # or: prebuilt packages — https://dotslash-cli.com/docs/installation/
  /usr/bin/env dotslash --help   # sanity check
  ```

- **protoc** — proto codegen resolves [`bin/protoc`](bin/protoc) via DotSlash,
  or falls back to a `protoc` on `PATH` / `$PROTOC`.
- macOS and Linux are supported build hosts; Windows builds are best-effort
  and not currently tested from this tree.

```sh
cargo run -p xai-grok-pager-bin              # build + launch the TUI
cargo build -p xai-grok-pager-bin --release  # release binary: target/release/xai-grok-pager
cargo check -p xai-grok-pager-bin            # fast validation
```

The binary artifact is named `xai-grok-pager`; official installs ship it as
`grok`. On first launch it opens your browser to authenticate — see the
[authentication guide](crates/codegen/xai-grok-pager/docs/user-guide/02-authentication.md).

## Documentation

Full online documentation is available at
[docs.x.ai/build/overview](https://docs.x.ai/build/overview).

The user guide ships with the pager crate:
[`crates/codegen/xai-grok-pager/docs/user-guide/`](crates/codegen/xai-grok-pager/docs/user-guide/)
— getting started, keyboard shortcuts, slash commands, configuration, theming,
MCP servers, skills, plugins, hooks, headless mode, sandboxing, and more.

## Repository layout

| Path | Contents |
|------|----------|
| `crates/codegen/xai-grok-pager-bin` | Composition-root package; builds the `xai-grok-pager` binary |
| `crates/codegen/xai-grok-pager` | The TUI: scrollback, prompt, modals, rendering |
| `crates/codegen/xai-grok-shell` | Agent runtime + leader/stdio/headless entry points |
| `crates/codegen/xai-grok-tools` | Tool implementations (terminal, file edit, search, ...) |
| `crates/codegen/xai-grok-workspace` | Host filesystem, VCS, execution, checkpoints |
| `crates/codegen/...` | The rest of the CLI crate closure (config, MCP, markdown, sandbox, ...) |
| `crates/common/`, `crates/build/`, `prod/mc/` | Small shared leaf crates pulled in by the closure |
| `third_party/` | Vendored upstream source (Mermaid diagram stack) — see below |

> [!IMPORTANT]
> The root `Cargo.toml` (workspace members, dependency versions, lints,
> profiles) is **generated** — treat it as read-only. Prefer editing per-crate
> `Cargo.toml` files.

<a id="project-reading-guide"></a>

## 项目导读与阅读路径

如果你第一次阅读这个项目，不建议从整个 `workspace` 逐个 crate 浏览。最有效的方式是先从最终可执行文件的组合根开始，再沿着一次 prompt 的运行路径逐层展开。

### 先记住这条主链

```text
grok
  -> xai-grok-pager-bin/src/main.rs       # 进程初始化与模式分发
  -> xai-grok-pager/src/app/cli.rs        # CLI 参数和子命令
  -> xai-grok-pager/src/app/mod.rs        # TUI 启动与应用组装
  -> xai-grok-pager/src/app/event_loop.rs # 输入、消息和渲染事件循环
  -> xai-grok-shell/src/agent/app.rs      # agent runtime 入口
  -> session / turn loop / tool dispatch
  -> xai-grok-tools + xai-grok-workspace  # 工具执行和工作区操作
```

`xai-grok-pager-bin` 是 composition root：它把 pager、shell、workspace、更新器和可选的 minimal render mode 组合成最终二进制。Cargo package 名是 `xai-grok-pager-bin`，生成的 binary artifact 名是 `xai-grok-pager`，发布时对用户暴露为 `grok`。

从 `main.rs` 继续阅读时，注意运行模式会在这里分叉：

- **交互式 TUI** 进入 `xai-grok-pager::app::run`，随后进入 `app/mod.rs` 和 `event_loop.rs`。
- **Headless、ACP stdio 和 leader** 通过 `xai-grok-shell` 的 agent 入口运行，之后与 session、turn loop 和工具分发逻辑汇合。
- **文件、搜索、终端、Git/worktree 等操作** 最终由 `xai-grok-tools` 调用 `xai-grok-workspace` 提供的能力完成。

### 推荐阅读顺序

#### 1. 了解仓库边界和构建约定

先读根目录的 [`README.md`](README.md)、[`Cargo.toml`](Cargo.toml)、[`rust-toolchain.toml`](rust-toolchain.toml) 和 [CI 检查 workflow](.github/workflows/check.yml)：

- 这是一个 Rust Cargo monorepo，主要代码位于 `crates/codegen/`，共享叶子 crate 位于 `crates/common/`、`crates/build/` 和 `prod/mc/`。
- Rust toolchain 由 `rust-toolchain.toml` 固定。构建前还要留意 `protoc` 的查找规则和 `.cargo/config.toml` 中的目标平台配置。
- 根 `Cargo.toml` 是生成文件；修改依赖或 crate 配置时，应编辑对应 crate 的 `Cargo.toml`，不要直接改根 manifest。
- 默认优先执行 package-specific 命令，例如 `cargo check -p xai-grok-pager-bin`，不要一开始就构建整个 workspace。

#### 2. 从组合根和 CLI 开始

1. 阅读 [`xai-grok-pager-bin/Cargo.toml`](crates/codegen/xai-grok-pager-bin/Cargo.toml)，了解最终 binary 依赖哪些运行时 crate，以及为什么 minimal mode 在 binary 层接入。
2. 阅读 [`xai-grok-pager-bin/src/main.rs`](crates/codegen/xai-grok-pager-bin/src/main.rs)，跟踪启动初始化、认证/配置准备、崩溃处理和运行模式分发。
3. 阅读 [`xai-grok-pager/src/app/cli.rs`](crates/codegen/xai-grok-pager/src/app/cli.rs)，建立 `Command`、`AgentArgs`、TUI、headless、ACP、MCP、workspace 等命令之间的映射。

#### 3. 理解 TUI 生命周期

- [`xai-grok-pager/src/app/mod.rs`](crates/codegen/xai-grok-pager/src/app/mod.rs) 负责应用启动、终端初始化、session startup 和根视图组装。
- [`xai-grok-pager/src/app/event_loop.rs`](crates/codegen/xai-grok-pager/src/app/event_loop.rs) 是事件循环，处理终端输入、ACP 消息、后台任务和渲染刷新。
- 需要理解状态更新时，再看 `app/actions.rs`、`app/dispatch/mod.rs`、`app/effects/mod.rs`；需要理解视图时，看 `app/app_view.rs` 和 `app/agent_view/mod.rs`。
- 渲染、主题、Markdown 和终端原语集中在 [`xai-grok-pager-render`](crates/codegen/xai-grok-pager-render/src/lib.rs)；pager crate 会重新导出其中的一部分能力。
- pager 局部架构可参考 [`xai-grok-pager/README.md`](crates/codegen/xai-grok-pager/README.md)，用户行为则看 [pager user guide](crates/codegen/xai-grok-pager/docs/user-guide/README.md)。

#### 4. 理解 agent、session 和 turn loop

从 [`xai-grok-shell/src/agent/app.rs`](crates/codegen/xai-grok-shell/src/agent/app.rs) 开始，先看不同 agent 运行方式的入口。然后按下面的顺序深入：

1. [`xai-grok-shell/src/session/mod.rs`](crates/codegen/xai-grok-shell/src/session/mod.rs)：session API、持久化、通知和 prompt 来源。
2. [`xai-grok-shell/src/session/acp_session.rs`](crates/codegen/xai-grok-shell/src/session/acp_session.rs)：ACP session 的边界。
3. [`xai-grok-shell/src/session/acp_session_impl/run_loop.rs`](crates/codegen/xai-grok-shell/src/session/acp_session_impl/run_loop.rs)：一次 turn 如何推进。
4. [`xai-grok-shell/src/session/acp_session_impl/tool_dispatch.rs`](crates/codegen/xai-grok-shell/src/session/acp_session_impl/tool_dispatch.rs)：模型产生工具调用后如何分发和回传结果。

当你需要研究多进程协调时，再阅读 `xai-grok-shell/src/leader/`；不要把 leader 细节当成普通 TUI 启动路径的一部分。

#### 5. 理解 tools、workspace 和安全边界

- 工具 crate 的整体导出从 [`xai-grok-tools/src/lib.rs`](crates/codegen/xai-grok-tools/src/lib.rs) 开始。
- 工具注册和分类见 [`xai-grok-tools/src/registry/mod.rs`](crates/codegen/xai-grok-tools/src/registry/mod.rs)，具体实现见 [`xai-grok-tools/src/implementations/mod.rs`](crates/codegen/xai-grok-tools/src/implementations/mod.rs)。
- workspace crate 的总体边界见 [`xai-grok-workspace/src/lib.rs`](crates/codegen/xai-grok-workspace/src/lib.rs)，操作协议和能力边界见 [`workspace_ops.rs`](crates/codegen/xai-grok-workspace/src/workspace_ops.rs)。
- 文件系统、权限、session 状态和 worktree 分别从 `xai-grok-workspace/src/file_system/`、`permission/`、`session/` 和 `worktree/` 展开。
- 阅读或修改会触及文件写入、命令执行、权限确认或 worktree 的代码时，应同时参考 [`SECURITY.md`](SECURITY.md) 和对应模块的测试。

#### 6. 最后补充配置、扩展和测试

- 配置加载从 [`xai-grok-config/src/lib.rs`](crates/codegen/xai-grok-config/src/lib.rs) 开始，再按需阅读 `loader.rs` 和 `paths.rs`。
- shell 的扩展入口位于 `xai-grok-shell/src/extensions/`，包括 `mcp.rs`、`hooks.rs`、`plugins.rs` 和 `skills.rs`。
- MCP 的 transport、OAuth 和 server lifecycle 由 [`xai-grok-mcp/src/lib.rs`](crates/codegen/xai-grok-mcp/src/lib.rs)、`servers.rs`、`oauth.rs` 和 `mcp_http_client.rs` 组成。
- 需要了解实现行为时，优先找离改动最近的定向测试，而不是直接运行整个 workspace 测试集。

### 按任务快速跳转

| 想了解或修改的内容 | 首个入口 | 下一步 |
|---|---|---|
| CLI 参数和子命令 | [`app/cli.rs`](crates/codegen/xai-grok-pager/src/app/cli.rs) | [`main.rs`](crates/codegen/xai-grok-pager-bin/src/main.rs) |
| TUI 启动和事件循环 | [`app/mod.rs`](crates/codegen/xai-grok-pager/src/app/mod.rs) | [`event_loop.rs`](crates/codegen/xai-grok-pager/src/app/event_loop.rs) |
| TUI 状态和交互 | [`app/app_view.rs`](crates/codegen/xai-grok-pager/src/app/app_view.rs) | `app/actions.rs`、`app/dispatch.rs` |
| 渲染、主题和终端原语 | [`xai-grok-pager-render/src/lib.rs`](crates/codegen/xai-grok-pager-render/src/lib.rs) | 该 crate 导出的具体模块 |
| Headless、ACP 和 leader | [`agent/app.rs`](crates/codegen/xai-grok-shell/src/agent/app.rs) | `xai-grok-shell/src/session/`、`leader/` |
| Session 和 turn loop | [`session/acp_session.rs`](crates/codegen/xai-grok-shell/src/session/acp_session.rs) | `session/acp_session_impl/` |
| 工具注册和内置工具 | [`registry/mod.rs`](crates/codegen/xai-grok-tools/src/registry/mod.rs) | [`implementations/mod.rs`](crates/codegen/xai-grok-tools/src/implementations/mod.rs) |
| 文件、VCS、权限和 worktree | [`workspace_ops.rs`](crates/codegen/xai-grok-workspace/src/workspace_ops.rs) | `file_system/`、`permission/`、`worktree/` |
| 配置加载和合并 | [`xai-grok-config/src/lib.rs`](crates/codegen/xai-grok-config/src/lib.rs) | `loader.rs`、`paths.rs` |
| MCP 和扩展 | [`xai-grok-mcp/src/lib.rs`](crates/codegen/xai-grok-mcp/src/lib.rs) | shell 的 `extensions/` |
| 用户可见行为 | [pager user guide](crates/codegen/xai-grok-pager/docs/user-guide/README.md) | 按功能进入对应用户文档 |
| vendored 第三方代码 | [`third_party/README.md`](third_party/README.md) | 仅在需要审计或升级时阅读 |

### 测试入口

根据改动范围选择定向测试：

```sh
cargo test -p xai-grok-pager --test settings_e2e
cargo test -p xai-grok-pager --test scripted_scenarios
cargo test -p xai-grok-tools --test path_suggestions_production
cargo test -p xai-grok-mcp --test repro_sse_flood -- --nocapture
```

涉及真实终端或多进程行为时，再查看并显式运行 [`pty_e2e`](crates/codegen/xai-grok-pager/tests/pty_e2e/mod.rs) 和 [`leader_pty_e2e`](crates/codegen/xai-grok-pager/tests/leader_pty_e2e/mod.rs)。这些测试通常更慢、需要额外环境，不适合作为第一次阅读项目时的默认验证步骤。

### 文档分工

- 根 `README.md`：项目入口、构建边界和跨 crate 的阅读路径。
- [pager user guide](crates/codegen/xai-grok-pager/docs/user-guide/README.md)：认证、快捷键、slash commands、配置和功能使用方式。
- [`xai-grok-pager/README.md`](crates/codegen/xai-grok-pager/README.md)：pager 局部架构和交互实现概览。
- 各 crate 的源码、测试和局部 README：具体实现细节。
- [`third_party/README.md`](third_party/README.md)：vendored 上游代码及其许可证边界。

当组合根、CLI 分发、crate 重命名或目录结构发生变化时，应同步检查本节的主链和链接；新功能的使用说明应优先放进 user guide，避免让根 README 变成第二份完整设计文档。

## Development

```sh
cargo check -p <crate>        # always target specific crates; full-workspace builds are slow
cargo test -p xai-grok-config # per-crate tests
cargo clippy -p <crate>       # lint config: clippy.toml at the repo root
cargo fmt --all               # rustfmt.toml at the repo root
```

## Contributing

> [!NOTE]
> External contributions are not accepted. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

First-party code in this repository is licensed under the **Apache License,
Version 2.0** — see [`LICENSE`](LICENSE).

Third-party and vendored code remains under its original licenses. See:

- [`THIRD-PARTY-NOTICES`](THIRD-PARTY-NOTICES) — crates.io / git dependencies,
  bundled UI themes, and **in-tree source ports** (including openai/codex and
  sst/opencode tool implementations)
- [`crates/codegen/xai-grok-tools/THIRD_PARTY_NOTICES.md`](crates/codegen/xai-grok-tools/THIRD_PARTY_NOTICES.md)
  — crate-local notice for the codex and opencode ports (license texts +
  Apache §4(b) change notice)
- [`third_party/NOTICE`](third_party/NOTICE) — vendored Mermaid-stack index
