# Pumpkin Migration AI Context

Updated: 2026-09-02

This file gives future development sessions the current technical context for the PlayAsia Pumpkin migration. It contains no credentials, API keys, passwords, private keys, or login details.

## Goal

Move the disposable DEV PlayAsia evaluation from the Java ShreddedPaper server to the isolated Pumpkin DEV server, prove that Pumpkin and PatchBukkit can run the copied world, then test plugins in increasing order of complexity. Production must remain untouched.

The only test allocation is Pumpkin DEV on port `25553`. The production servers, proxy, and the existing DEV server are out of scope for writes, restarts, file copies, and configuration changes.

## Repository Workflow

- Pumpkin work belongs in the `Aevienne/Pumpkin` fork.
- PatchBukkit bridge work belongs in the `Aevienne/PatchBukkit` fork.
- Use feature branches named `feat/<name>` or a diagnostic branch with a clear name.
- Pull before pushing. The fork history is shared by multiple sessions.
- Add `https://github.com/Pumpkin-MC/Pumpkin.git` as the Pumpkin `upstream` remote when absent.
- Keep PatchBukkit disabled on the target after failed tests.
- Do not commit or print secrets. Redact all operational output before placing it in issues, pull requests, logs, or this file.

## Migration State

- The copied world and configuration data are present on Pumpkin DEV.
- The world copy was transferred server-side and is not to be edited during bridge tests.
- The Pumpkin binary is installed and runs without PatchBukkit on `25553`.
- The PatchBukkit native library and Java asset JAR are installed but disabled after failed tests.
- The copied Bukkit plugin set is quarantined. It must not be enabled as a group.
- The PatchBukkit test plugin has reproduced the same failure as an empty plugin directory.

## PatchBukkit Failure

Observed output ends after the embedded JVM begins loading the bridge:

```text
Starting PatchBukkit
PatchBukkit loaded successfully
Initializing JVM in background...
Initializing JVM with assets path: ".../jassets"
Found 1 JAR entries in jassets: [".../patchbukkit.jar"]
WARNING: A terminally deprecated method in sun.misc.Unsafe has been called
```

The process exits with code `139`, which is `SIGSEGV`. The failure occurs before Java plugin discovery. It reproduces with an empty Bukkit plugin directory and the official PatchBukkit test plugin. Base Pumpkin uses the same server binary and remains running when the bridge is disabled.

This rules out PlayAsiaCore, the copied world, and third-party Bukkit plugins as the cause of the first crash. The fault is in the PatchBukkit native/JVM/bootstrap path.

### Evidence and Hypotheses

The evidence proves a native crash but does not yet identify the crashing instruction. `RUST_BACKTRACE` cannot provide a native JVM backtrace. No usable `hs_err` file or native `gdb` backtrace has been retained.

The main suspects are:

1. Java class initialization in `PatchBukkitServer`, especially Minecraft `SharedConstants` and `Bootstrap` initialization.
2. JNA or Linux NUMA initialization reached through Minecraft bootstrap and `Unsafe`.
3. A JNI callback, object lifetime, or function-signature error in the Rust-to-Java bridge.
4. A build/runtime mismatch caused by building the Java bridge with Java 21 while the project and target use Java 25.

The duplicate-bootstrap theory is plausible but unproven. The diagnostic branch attempted to guard `Bootstrap.bootStrap()`, but its current reflection logic ignores a normal `false` result from `isBootstrapped()` and only checks that result inside an exception path. The guard must read the boolean result on the normal path, bootstrap only when false, and report failures instead of swallowing every `Throwable`.

`JNIVersion::V21` is not equivalent to requiring a Java 21 runtime. It requests a JNI interface version supported by Java 25. The Rust `jni` crate in use has no `V25` enum value, so changing it to `V25` is not a valid fix.

### Required Fix Workflow

1. Build the Java bridge with Java 25, matching the target runtime and project toolchain.
2. Use an empty PatchBukkit plugin directory.
3. Capture complete Pumpkin output without truncating it through `head` or a pipeline that hides the real exit status.
4. Save `/tmp/hs_err_pid*.log` and any core dump as CI artifacts.
5. Run the isolated test under `gdb` or inspect the core with `coredumpctl` where the host permits it.
6. Record the first native frame from `libjvm.so`, `libpatchbukkit.so`, JNA, or another native dependency.
7. Start with minimal JVM arguments and add one option at a time.
8. Fix bootstrap and error reporting before testing Bukkit plugins.
9. Prove the official PatchBukkit test plugin loads and enables.
10. Run BotMark only after the bridge reaches `JVM initialized successfully`.

The latest known GitHub Actions run for the diagnostic PatchBukkit branch built the Java and native artifacts successfully, including the cached Rust build. Its isolated deployment job failed before the target test completed, so that run is not evidence that the bridge fix works.

## PatchBukkit Acceptance Criteria

- PatchBukkit reaches `JVM initialized successfully`.
- Pumpkin remains running for at least five minutes with the bridge enabled.
- The empty-plugin test passes.
- The official test plugin loads and enables.
- A plugin exception is reported as a controlled failure rather than a process crash.
- Failed runs retain complete logs, JVM error files, and native backtraces.
- BotMark joins `25553`, completes login and world loading, moves, sends chat, and stays connected.
- Cleanup disables the bridge and test plugin after failed runs.
- Production receives no changes.

## Rust Build Speed

Pumpkin's slow release build comes from the large Wasmtime/Cranelift dependency graph and the workspace release profile:

```toml
[profile.release]
lto = true
codegen-units = 1
```

Fat LTO and one codegen unit are appropriate for a final artifact but slow for diagnostics. Build only the server package during development:

```powershell
cargo check -p pumpkin
cargo build -p pumpkin
```

For a runnable optimized diagnostic build, use environment overrides rather than changing the final release profile:

```powershell
$env:CARGO_PROFILE_RELEASE_LTO = "thin"
$env:CARGO_PROFILE_RELEASE_CODEGEN_UNITS = "16"
$env:CARGO_INCREMENTAL = "1"
cargo build -p pumpkin --release --locked
```

Use `sccache` for repeated builds. Use `lld` on Windows or `mold`/`lld` on Linux. Build WSL copies inside the Linux filesystem, not under `/mnt/c`. Keep Rust and Gradle caches separate in CI. Reserve fat LTO and `codegen-units=1` for tagged releases.

The PatchBukkit bridge workflow already uses disabled LTO, 16 codegen units, and Rust caching. It should build with Java 25, reuse cached artifacts, and avoid rebuilding Pumpkin when only PatchBukkit changed. Do not build Rust on the game-server VM.

## Pumpkin Plugin Architecture

Pumpkin's preferred plugin model is a WebAssembly component loaded from `plugins/*.wasm`. The source implementation is in:

- `crates/pumpkin/src/plugin/mod.rs`
- `crates/pumpkin/src/plugin/loader/wasm/mod.rs`
- `crates/pumpkin/src/plugin/loader/wasm/wasm_host/`
- `crates/pumpkin-plugin-api/src/lib.rs`
- `crates/pumpkin-plugin-api/src/commands.rs`
- `crates/pumpkin-plugin-api/src/scheduler.rs`
- `crates/pumpkin-plugin-api/src/events/`
- `crates/pumpkin-config/src/plugins.rs`

The host scans `./plugins`, skips directories and `.deactivated` files, loads WASM files through Wasmtime, obtains metadata, checks dependencies and permissions, then calls `on_load`. Unload calls `on_unload`. The current plugin API version is checked strictly for native plugins.

### Minimal Rust Plugin Shape

```rust
use pumpkin_plugin_api::{Context, Plugin, PluginMetadata};

struct ExamplePlugin;

impl Plugin for ExamplePlugin {
    fn new() -> Self {
        Self
    }

    fn metadata(&self) -> PluginMetadata {
        PluginMetadata {
            name: "example-plugin".into(),
            version: env!("CARGO_PKG_VERSION").into(),
            authors: vec!["Developer".into()],
            description: "Small Pumpkin plugin".into(),
            dependencies: vec![],
            permissions: vec![],
        }
    }

    fn on_load(&mut self, context: Context) -> pumpkin_plugin_api::Result<()> {
        context.log("Example plugin loaded");
        Ok(())
    }
}

pumpkin_plugin_api::register_plugin!(ExamplePlugin);
```

The current source API requires `dependencies` and `permissions`, even though some public documentation examples omit them.

### Plugin Project

Use Rust edition 2024, `crate-type = ["cdylib"]`, the `pumpkin-plugin-api` dependency, and a `.cargo/config.toml` containing:

```toml
[build]
target = "wasm32-wasip2"
```

Install the target with:

```bash
rustup target add wasm32-wasip2
```

Build with `cargo build --release`. Copy the resulting WASM component from `target/wasm32-wasip2/release/` into Pumpkin's `plugins/` directory.

The local `hello-pumpkin` project is a WASM API example, not a native PatchBukkit plugin. It needs a valid `.cargo/config.toml` selecting `wasm32-wasip2`, and its manifest should be checked for literal escaped quotes before it is used as a template.

### API Capabilities

The source exposes:

- Lifecycle hooks: `on_load`, `on_unload`.
- Metadata, dependency ordering, and permission declarations.
- Blocking and non-blocking events.
- Cancellable blocking events.
- Brigadier-style commands and suggestions.
- Delayed and repeating tick tasks.
- Player and server handles.
- Plugin services and IPC.
- Persistent data and plugin-specific data folders.
- World-generation and AI-goal hooks.
- WASI HTTP and socket access controlled by permissions.

Plugin data is stored under `plugins/data/<plugin-name>`. WASM filesystem access is sandboxed. Network, HTTP, environment, system-information, and data-directory access require declared permissions and server approval.

Blocking event handlers can modify or cancel events but run in sequence. Non-blocking handlers run after blocking handlers and cannot cancel events. Slow database or HTTP work must not run inside a blocking handler.

### Compatibility Limits

PatchBukkit is an experimental compatibility layer, not a complete Paper or ShreddedPaper runtime. Its scheduler does not reproduce Folia region ownership. Some Bukkit methods return defaults, `null`, empty values, or throw. NMS, reflection, Paper internals, ProtocolLib, entity schedulers, and region-specific behavior require separate validation.

For new PlayAsia code, prefer native Pumpkin WASM APIs where available. Use PatchBukkit only for plugins whose required Bukkit behavior has passed focused tests.

## Test Order

1. Base Pumpkin without PatchBukkit.
2. PatchBukkit with an empty Java plugin directory.
3. Official PatchBukkit test plugin.
4. A minimal command/listener/scheduler plugin.
5. Small PlayAsia plugins.
6. PlayAsiaTab and LobbyAlias.
7. PlayAsiaCore without ClansCore or ClansExtended.
8. ClansCore, ClansExtended, and other high-risk plugins one at a time.
9. BotMark after each stable bridge/plugin milestone.

Never infer plugin compatibility from a successful server boot alone. Verify login, world loading, commands, events, scheduler behavior, persistence, and controlled shutdown.
