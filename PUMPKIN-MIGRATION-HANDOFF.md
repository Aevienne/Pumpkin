# PlayAsia Pumpkin Migration Handoff

Date: 2026-09-01

## Current Objective

Evaluate the existing PlayAsia DEV server on PumpkinMC with PatchBukkit. Keep production on ShreddedPaper. Copy DEV data into the disposable Pumpkin DEV server, install the Pumpkin native binary and PatchBukkit bridge, start the server, test plugins and server behavior, then document working and failing features.

Do not modify production or the upstream ShreddedPaper/MultiPaper repositories. Production changes require explicit approval and a backup.

## Servers

Panel: `https://panel.playasia.online`

Existing DEV server:

- UUID: `71b0701f-09cc-4c8d-8ae4-8ae3a46953db`
- Name: existing DEV
- Runtime: ShreddedPaper/Paper-style Java server
- Approximate disk usage: 12.6 GB
- Plugin directory contains about 163 entries
- Last known state: stopped

Pumpkin DEV server:

- UUID: `0663eb80-d3b8-45bd-86b1-579098d1f653`
- Name: `[Pumpkin DEV] PlayAsia`
- Allocation: `10.10.10.2:25553`, public alias `139.99.121.15:25553`
- Node: `SGP-01-Node-1-Snapwing`
- Quota: 30 GB disk, 1 GB memory, 100% CPU
- Image: `ghcr.io/ptero-eggs/yolks:java_25`
- Initial egg: Paper
- Initial startup: `java -Xms128M -XX:MaxRAMPercentage=95.0 -Dterminal.jline=false -Dterminal.ansi=true -jar {{SERVER_JARFILE}}`
- Last known state: stopped

Production was not touched.

## Network Context

- Velocity proxy public port: `25565`
- Geyser public port: `19132`
- Live SMP: `25564`
- Existing DEV SMP: `25550`
- Pumpkin DEV allocation: `25553`
- The Pumpkin server must remain isolated from the production proxy until testing is complete.

## Local Files

Primary context:

- `E:\cnczh\SHREDDEDPAPER-PUMPKIN-CONTEXT.md`
- `E:\cnczh\MINECRAFT-DEVELOPMENT-CONTEXT.md`

ShreddedPaper source:

- `E:\snapwing\compiled-shreddedpaper\shreddedpaper-build\`
- Branch: `ver/26.2`
- HEAD: `60adb06`
- This is a local fork with working changes. Do not reset or discard them.

Plugin source:

- `E:\snapwing\plugin-dev\`
- Existing plugin source was clean at commit `27b2949` when checked.

Pumpkin source checkout:

- `E:\snapwing\pumpkin-eval-source\`
- Shallow master checkout at commit `b75e601997c254548bbb657db2234bf8b9bf0eec`

PatchBukkit source checkout:

- `E:\snapwing\patchbukkit-eval-source\`
- Shallow master checkout at commit `9ec7a3d08ec9f04ee5029e126d28eb6bcb9b9239`

Downloaded artifacts:

- `E:\snapwing\pumpkin-eval\pumpkin-X64-Linux`
- `E:\snapwing\patchbukkit-eval\libpatchbukkit-X64-Linux.so`
- `E:\snapwing\patchbukkit-eval\RELEASE.md`
- `RELEASE.md`: generated from PatchBukkit commit `9ec7a3d`, generated `2026-08-27 20:52 UTC`

Artifact hashes:

- Pumpkin: `357F53E30C0D63A147A1EC9C893B04A4E10F07DDE0F22729B35F2D13AF74CD35`
- PatchBukkit: `B4C5D4AA131B6721E8C79040BA45F80E81FFD47D0CDD2311183F9FB574EB7681`
- Release file: `AD002CE2CF0BB8E89D8927F2C843E26E4550A46DFBA1E018E2029F4E54A76D1C`

Migration archive currently on the PC:

- `E:\snapwing\pumpkin-eval\dev-plugins.tar.gz`
- It was created from source server `/plugins` with Wings compression using root `/plugins` and files `["."]`.
- Local tar listing showed top-level plugin directories and JARs without a `plugins/` prefix.
- This archive was downloaded before the server-side transfer approach was found. Avoid downloading further server data to the PC.

## Migration Progress

Completed:

1. Identified the existing DEV and new Pumpkin DEV server UUIDs.
2. Stopped both DEV servers through the panel API.
3. Confirmed production remained untouched.
4. Confirmed individual file download, archive compression, upload, directory creation, and decompression API behavior.
5. Created the source plugin archive on the existing DEV server.
6. Uploaded `dev-plugins.tar.gz` to the Pumpkin DEV server. The upload endpoint ignored the requested directory and placed the archive at target root `/dev-plugins.tar.gz`.
7. Decompressed that archive on the target with root `/patchbukkit/patchbukkit-plugins`.
8. Verified target `/patchbukkit/patchbukkit-plugins` contains 69 top-level plugin data entries, including `PlayAsiaCore`, `PlayAsiaBazaar`, `PlayAsiaEnchants`, `Citizens`, `BetterModel`, `PlaceholderAPI`, and others.

Not completed:

- World data has not been copied.
- Root server configuration has not been copied.
- Pumpkin binary has not been confirmed on the target.
- PatchBukkit native library has not been confirmed on the target.
- Target startup command has not been changed.
- Pumpkin DEV has not been started.
- No compatibility test has run.

Target cleanup still needed after the migration is staged:

- Remove `/dev-plugins.tar.gz` from target root after verifying extraction.
- Remove `/migration-test.properties` from target root.
- Check for partial `/pumpkin` or `/libpatchbukkit.so` files after the failed artifact upload.

## Transfer API Findings

The panel is a Calagopus/Wings deployment. JSON requests sent through Windows PowerShell to `curl.exe` must escape internal JSON quotes. Without escaping, PowerShell strips the quotes and Wings returns errors such as:

`invalid payload: key must be a string at line 1 column 2`

Working pattern:

```powershell
$body = '{\"root\":\"/\",\"files\":[\"server.properties\"]}'
curl.exe --data-raw $body ...
```

Wings-compatible payloads confirmed from `E:\snapwing\wings-source\router\router_server_files.go`:

- Compress: `{ "root": "/", "files": ["name"] }`
- Decompress: `{ "root": "/target", "file": "/archive.tar.gz" }`
- Create directory: `{ "name": "dirname", "path": "/parent" }`
- Pull: `{ "root": "/", "url": "https://...", "file_name": "name", "foreground": false }`

Important behavior:

- Source `files/compress` works.
- Target `files/upload` works for small files and accepted the large plugin archive upload.
- Target `files/decompress` works.
- Source signed download URLs use `https://panel.playasia.online/wings-proxy/<node>/download/file?...`.
- Target `files/pull` rejected the source signed URL with HTTP 417: `failed to create pull: failed to send download request`.
- The likely cause is that the target node cannot reach the panel’s Wings proxy URL from inside the VM/container.
- SSH was unavailable: no usable PEM key was found and the configured `OPENCODE_SERVER_PASSWORD` was rejected by `plink.exe`.
- No fresh backup was created. Existing DEV backups were present, including backup UUID `c97bf0f3-8b7b-45d1-a6e6-4d4eb51bf45a` with size `12,606,755,680` bytes.

Last failed action:

- Multipart upload of Pumpkin and PatchBukkit artifacts to target root reset at about 0.4% with curl error 55: `Send failure: Connection was reset`.
- Verify whether partial files exist before retrying.

## Recommended Transfer Sequence

1. Verify and clean partial target files and test files.
2. Use source Wings compression for `world`, with root `/` and files `["world"]`.
3. Prefer node-side transfer through a reachable internal node URL or server-transfer endpoint. Do not download the world archive to the PC.
4. If node-side transfer cannot reach the panel proxy, obtain SSH access to the Calagopus node or use a panel upload only for the remaining archives.
5. Compress root configuration files separately, excluding `server.jar`, logs, caches, JFR files, backups, and old runtime artifacts.
6. Keep the source plugin archive extraction at `/patchbukkit/patchbukkit-plugins` because PatchBukkit creates each plugin data folder beside its JAR.
7. Upload Pumpkin as `/pumpkin` and PatchBukkit as `/libpatchbukkit.so`; confirm file sizes and executable permissions through the panel.
8. Inspect PatchBukkit startup documentation and source for the exact native plugin configuration before editing the startup command.
9. Start the target only after all copied data and runtime files are present.
10. Test in this order: PatchBukkit boot, minimal command/listener/scheduler plugin, PlayAsiaTab, LobbyAlias, PlayAsiaCore without ClansCore/ClansExtended, then the ClansCore/ClansExtended chain.

## Pumpkin and PatchBukkit Findings

Pumpkin:

- Native Rust Minecraft server.
- Native plugin model uses WebAssembly components, WIT, WASI, Wasmtime, and native platform plugins.
- Bukkit, Spigot, and Paper JARs do not run directly.
- Existing Anvil world data is intended to be readable, but world behavior, entities, authentication, plugins, and proxy forwarding require tests.

PatchBukkit:

- Pumpkin native plugin with a Java 25 compatibility implementation.
- Loads Java plugins from `patchbukkit/patchbukkit-plugins/`.
- Reimplements substantial Bukkit and Paper APIs but does not provide the ShreddedPaper region ownership model.
- Bukkit sync tasks and Paper region/global schedulers use a scheduled executor.
- Region scheduler ignores world and chunk coordinates.
- `Entity#getScheduler()` currently throws `UnsupportedOperationException`.
- Some entity, world, persistence, serialization, unsafe, and internal bridge methods return defaults, null, or throw.
- Lifecycle registration does not retain and invoke registered handlers.
- Native event handling crosses the Rust/Java bridge and may block on Java handlers.
- Nightly artifacts are moving prereleases.
- PatchBukkit’s working-plugin tracker listed no working plugins when checked.

References:

- Pumpkin repository: `https://github.com/Pumpkin-MC/Pumpkin`
- Pumpkin Bukkit migration: `https://docs.pumpkinmc.org/admin/migrating-from-bukkit`
- Pumpkin plugin migration: `https://docs.pumpkinmc.org/plugin-dev/migrating-from-bukkit/`
- PatchBukkit repository: `https://github.com/Pumpkin-MC/PatchBukkit`
- PatchBukkit architecture: `https://github.com/Pumpkin-MC/PatchBukkit/blob/master/ARCHITECTURE.md`
- PatchBukkit README: `https://github.com/Pumpkin-MC/PatchBukkit/blob/master/README.md`
- Local PatchBukkit loader source: `E:\snapwing\patchbukkit-eval-source\java\patchbukkit\src\main\java\org\patchbukkit\PatchBukkitPluginClassLoader.java`
- Local PatchBukkit server source: `E:\snapwing\patchbukkit-eval-source\java\patchbukkit\src\main\java\org\patchbukkit\PatchBukkitServer.java`
- Local Wings file API source: `E:\snapwing\wings-source\router\router_server_files.go`
- Local Wings transfer API source: `E:\snapwing\wings-source\router\router_server_transfer.go`

## Plugin Risk

Critical:

- `ClansCoreHotfixes`
- `ClansExtended`
- `PlayAsiaKothHotfix`
- `ClanChatBridge`

High:

- `PlayAsiaStacker`
- `PlayAsiaCore`
- `PlayAsiaEnchants`
- `FlyToggle`

Medium:

- `PlayAsiaTab`
- `LobbyAlias`
- `BetterBounty`
- `PlayAsiaBossBalance`
- fast-leaf-decay Spigot module

The recent Folia/ShreddedPaper migration depends on real region and entity scheduling. PatchBukkit cannot be assumed to preserve those semantics. This evaluation is a compatibility proof of concept, not a production migration.

## Existing Backup Context

Existing DEV backups:

- `95722518-16da-4fe7-ae39-1aaf5bfa4ab5`
- `c97bf0f3-8b7b-45d1-a6e6-4d4eb51bf45a`, approximately 12.6 GB
- `2106a987-c191-4143-8a71-8f2dd22b8f10`

No new snapshot was created during this work. Treat the existing backups as fallback copies, not proof that the current DEV filesystem is fully captured.

## OpenCode Command Code Configuration

Global files:

- `C:\Users\Vincent\.config\opencode\opencode.json`
- `C:\Users\Vincent\.config\opencode\opencode.jsonc`

Current state:

- Default model: `commandcode/gpt-5.6-luna`
- OpenAI-compatible provider: `commandcode`
- Anthropic provider: `commandcode-anthropic`
- Base URL: `https://api.commandcode.ai/provider/v1`
- Exact API catalog was imported from `GET /provider/v1/models`.
- `opencode models commandcode` showed 54 models.
- `opencode models commandcode-anthropic` showed 7 Claude models.
- Both config files validated as JSON-compatible.

The Command Code API catalog IDs are:

```text
claude-sonnet-5
claude-sonnet-4-6
claude-fable-5
claude-opus-5
claude-opus-4-8
claude-opus-4-7
claude-haiku-4-5-20251001

Do not store the Command Code API key in this handoff. The pasted key in the earlier terminal example was not passed correctly: the correct syntax is `-H "Authorization: Bearer YOUR_KEY"`. Rotate that key if it was a real active secret.

## Prior Performance Work Still Open

JFR recordings:

- `jfr-20260823-004350-120s.jfr`
- `jfr-20260823-170233-125s.jfr`

The panel root cause investigation and JFR analysis were not completed. Keep those separate from the Pumpkin compatibility experiment.

## First Commands for the Next Session

From `C:\Users\Vincent\Desktop\snapwing`:

```powershell
$key = $env:calagopus
if ([string]::IsNullOrWhiteSpace($key)) { $key = $env:CALAGOPUS }
$target = '0663eb80-d3b8-45bd-86b1-579098d1f653'
curl.exe -sS -H "Authorization: Bearer $key" -H 'Accept: application/json' "https://panel.playasia.online/api/client/servers/$target/files/list?directory=%2F&page=1"
curl.exe -sS -H "Authorization: Bearer $key" -H 'Accept: application/json' "https://panel.playasia.online/api/client/servers/$target/files/list?directory=%2Fpatchbukkit%2Fpatchbukkit-plugins&page=1"
```

Before any new upload, verify whether `/pumpkin`, `/libpatchbukkit.so`, `/dev-plugins.tar.gz`, and `/migration-test.properties` exist. Remove only the known temporary files. Do not delete copied plugin data or world data.

## 2026-09-01 Session Update

- Language: all future reports and handoffs must use plain English.
- `DEVPlayAsia` (`71b0701f...`) and all production servers must remain untouched. Read-only resource checks are allowed; do not copy or edit its world.
- `Pumpkin DEV` (`0663eb80...`) remains the only test target. Its world copy is complete and must not be edited during bridge tests.
- Base Pumpkin starts on port `25553` without PatchBukkit. The PatchBukkit bridge crashes during embedded JVM startup, before any Java plugin loads.
- The crash reproduces with an empty plugin directory and the official PatchBukkit test plugin. Current bridge and test plugin remain disabled on the target.
- PatchBukkit checkout: `E:\snapwing\patchbukkit-eval-source`, branch `diag/playasia-jvm-core-dump`.
- PatchBukkit commits added this session: `9ef6cc6` adds JVM diagnostics, JOML no-unsafe settings, and required Java module opens; `f89057a` adds the Linux GitHub Actions build; `672673c` adds automated Pumpkin DEV deployment and test.
- Workflow: `https://github.com/Aevienne/PatchBukkit/actions/workflows/build-linux.yml`.
- GitHub environment `pumpkin-dev` contains the SSH key and host key. `VM_USER` and `VM_SUDO_PASSWORD` were added by the owner.
- The workflow builds on GitHub Actions, uploads the Linux bridge, deploys only to Pumpkin DEV, runs an isolated 30-second Docker test, then restores `.disabled` files. It never starts the panel server and never touches production.
- Do not build Rust on the Calagopus VM or the laptop. The VM build caused host memory pressure and production container restarts. The laptop build caused WSL failures even with resource limits.
- The manually triggered workflow run `33498881654` was still in progress when this handoff was updated. Check its status before triggering another run.
- No PAT, panel key, VM password, private key, or other login detail belongs in this file or any repository file.
