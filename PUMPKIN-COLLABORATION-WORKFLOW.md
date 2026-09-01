# Pumpkin Migration — Centralized Collaboration Workflow

> Give this file to ANY model/chat session. It will resume correctly without re-asking.

## 0) Goal
Migrate `DEVPlayAsia` → `Pumpkin DEV` on Calagopus VM, stabilize Pumpkin + PatchBukkit, keep production untouched, collaborate via forks `Aevienne/Pumpkin` and `Aevienne/PatchBukkit`.

## 1) Prerequisites (check, never print secrets)
- `gh` installed: `C:\Users\Vincent\bin\gh.exe` (v2.98.0). Auth via keyring, not env:
  ```
  gh auth status
  # expect: Logged in to github.com account Aevienne (read:org, repo, workflow)
  ```
  If not logged in: `Get-Content token.txt -Raw | gh auth login --with-token` then `Remove-Item token.txt -Force` (never commit token.txt).
- Panel API key in env `calagopus` / `CALAGOPUS` (used as `Authorization: Bearer $key`).
- VM SSH: `ailegrabielle@139.99.121.15:2222` pass `@l03e1t3` hostkey `ssh-ed25519 255 SHA256:khTar0VCSUTUNy21qoL44GVZjDipvEQlB95vez8OpzY`
  Use `plink.exe -batch -hostkey <hk> -P 2222 -l ailegabrielle -pw <pw> 139.99.121.15 "<cmd>"` or `sudo -S -p '' bash -s` for volume ops.
- Handoff: `C:\Users\Vincent\Desktop\snapwing\PUMPKIN-MIGRATION-HANDOFF.md` — read first.

## 2) Repository Map (centralized)
- Upstream: `Pumpkin-MC/Pumpkin`, `Pumpkin-MC/PatchBukkit`
- **Forks (central, all work here):** `Aevienne/Pumpkin` https://github.com/Aevienne/Pumpkin , `Aevienne/PatchBukkit` https://github.com/Aevienne/PatchBukkit (both `isFork:true`)
- Clone for work:
  ```
  gh repo clone Aevienne/Pumpkin
  gh repo clone Aevienne/PatchBukkit
  git remote -v  # origin -> Aevienne, upstream -> Pumpkin-MC (add with: git remote add upstream https://github.com/Pumpkin-MC/Pumpkin.git)
  ```
- Sync upstream: `git fetch upstream; git merge upstream/master` (or rebase).
- Collaboration: each feature = branch `feat/<name>` pushed to `Aevienne/*`, PR against `Aevienne/*:master` (or upstream PR from fork). Both models push to same forks — pull before push.

## 3) Server Inventory (Calagopus)
- VM volumes: `/var/lib/calagopus-wings/volumes/<server-id>`
- Panel: `https://panel.playasia.online`
- Do NOT touch production:
  - `PlayAsia` `0e5793db-755a-4c3a-a730-450c0f1b43a5` :25564
  - `Lobby` `43dcbf87-4156-47f6-8eca-8ff820191348` :25551
  - `Minigames` `d927e1a7-60a1-4bd0-bd8d-b59c07af508d` :25552
  - `Proxy` `f2dcab78-9120-4043-a96c-f55f212e40a6` :25565/:19132
  - `OneBlock` `d13505b8-48c3-4a2f-9b91-e8ce4f37be62` :25560
- Migration only:
  - Source `DEVPlayAsia` `71b0701f-09cc-4c8d-8ae4-8ae3a46953db` :25550 (world 5.5GB / 3426 files, seed probe via Java 25 container needed — `/seed` broken on ShreddedPaper console)
  - Target `Pumpkin DEV` `0663eb80-d3b8-45bd-86b1-579098d1f653` :25553 (isolated)

## 4) Current Migration State (2026-09-01)
- World + configs `rsync` via VM SSH (not panel proxy) with `sudo` (source files mode 600). Both volumes 5.5GB/3426 files match.
- Seed fixed: probe `world.getSeed()` → populate `world/data/minecraft/world_gen_settings.dat` (PatchBukkit expects `/world/data/...`). Leave `world/level.dat` from source; don't invent world.
- Artifacts installed to target volume:
  - `/pumpkin` hash `357F53E30C0D63A147A1EC9C893B04A4E10F07DDE0F22729B35F2D13AF74CD35` (110MB, +x)
  - `/plugins/libpatchbukkit.so` hash `B4C5D4AA131B6721E8C79040BA45F80E81FFD47D0CDD2311183F9FB574EB7681` (131M, rwxr-xr-x)
  - `pumpkin.toml` isolated: `25553`, Bedrock/query disabled. Startup `./pumpkin` (panel power API).
  - `plugins/data/patchbukkit/jassets/patchbukkit.jar` 87M present.
- Base Pumpkin **without** PatchBukkit = stable: `Server is now running. Connect using port: 0.0.0.0:25553` (verified via `docker run --rm --entrypoint "" -v <vol>:/home/container -w /home/container ghcr.io/ptero-eggs/yolks:java_25 bash -c 'timeout 15 ./pumpkin'`)
- PatchBukkit enabled → `Initializing JVM with assets path: .../jassets/patchbukkit.jar` then `timeout: the monitored command dumped core` / exit 139 even with empty `patchbukkit/patchbukkit-plugins`. Same with official `patchbukkit-test-plugin.jar` (43K, built on VM Java 25 via `yolks:java_25 ./gradlew :patchbukkit-test-plugin:build`). Leaves `libpatchbukkit.so.disabled` + `*.jar.disabled` to keep target offline/stable.
- Preserved full plugin set in `patchbukkit/patchbukkit-plugins.quarantine`; active dir quarantined.
- Published compatibles: `AwesomePumpkin` (~9 native Rust/WASM) + `PumpkinHub` (WASM-only, empty); PatchBukkit has no Bukkit compat matrix — nightly/experimental.

## 5) Resume Checklist (every new session)
1. Read `PUMPKIN-MIGRATION-HANDOFF.md` + this file.
2. Verify: `gh auth status`, `gh repo view Aevienne/Pumpkin --json nameWithOwner,isFork`, same for PatchBukkit.
3. Panel check: `Invoke-RestMethod /api/client/servers/<id>/resources` for Pumpkin DEV (expect offline unless testing) — don't start production.
4. VM check: `plink ... "ls -lh /var/lib/calagopus-wings/volumes/0663eb80.../pumpkin; ls -lh .../plugins/libpatchbukkit.so* | head"`
5. If testing: `docker run --rm -v ... yolks:java_25 bash -c 'timeout 15 ./pumpkin 2>&1 | head -n 400'` — capture crash before panel log truncation.

## 6) Workflow (centralized)
- All code changes → branches on `Aevienne/Pumpkin` or `Aevienne/PatchBukkit`, pushed, PR for review. Upstream sync via `git fetch upstream`.
- For VM fixes (seed, toml, lib): fix locally, document commit, also note volume path if manual SSH fix needed.
- To test PatchBukkit: re-enable `sudo mv .../libpatchbukkit.so.disabled .../libpatchbukkit.so` + move one jar from quarantine, run manual docker capture with `RUST_BACKTRACE=1`, then disable again if crash.
- Never change production servers, shared Paper egg, or panel startup for other servers. Record original DB values before any `servers` table edit (if needed).

## 7) Quick Commands
```
# forks
gh repo view Aevienne/Pumpkin --json nameWithOwner,isFork
gh repo clone Aevienne/Pumpkin; cd Pumpkin; git remote add upstream https://github.com/Pumpkin-MC/Pumpkin.git

# panel start/monitor (PowerShell)
$key=$env:calagopus; $id='0663eb80-d3b8-45bd-86b1-579098d1f653'
Invoke-RestMethod -Method Post -Uri "https://panel.playasia.online/api/client/servers/$id/power" -Headers @{Authorization="Bearer $key"} -Body '{"signal":"start"}'
Invoke-RestMethod -Method Get -Uri "https://panel.playasia.online/api/client/servers/$id/resources" -Headers @{Authorization="Bearer $key"}

# VM manual test
plink -batch -hostkey ssh-ed25519\ 255\ SHA256:khTar0VCSUTUNy21qoL44GVZjDipvEQlB95vez8OpzY -P 2222 -l ailegabrielle -pw <pw> 139.99.121.15 "bash ~/crash-test.sh"
```

## 8) Safety & Secrets (MANDATORY — any model must obey)
- **NEVER post, print, echo, log, or commit ANY secret** — GitHub PAT (`ghp_...`/`github_pat_...`), `calagopus`/`CALAGOPUS` panel API key, VM SSH password (`@l03e1t3`), hostkey private material, or any token/password/login detail — not in chat, not in GitHub issues/PRs/commits, not in logs, not in screenshots, not in `git diff`.
- `token.txt` is **ephemeral only**: `Get-Content token.txt -Raw | gh auth login --with-token` then `Remove-Item token.txt -Force` immediately; it is gitignored and must never be committed. Same for any `*.env` or secret file.
- When debugging, redact: use `gh auth status` (shows `ghp_****`) and `$key.Substring(0,4) + '****'` never the full value. `gh auth token` output must never be pasted anywhere.
- If you must reference VM SSH, do not re-print the password in PR descriptions or commit messages — assume the reader has this workflow file locally.
- Read-only checks first; stop source before world copy.
- Keep `25553` isolated; Bedrock off until verified.

## 9) Prompt to Resume (paste to new model)
> Resume Pumpkin migration per `PUMPKIN-COLLABORATION-WORKFLOW.md` and `PUMPKIN-MIGRATION-HANDOFF.md`. Verify `gh auth status` (Aevienne), forks `Aevienne/Pumpkin` + `Aevienne/PatchBukkit`, VM SSH, and server states. Continue centralized work on those forks (branches/PRs), keep production untouched. Report current state then proceed with next step (PatchBukkit JVM core-dump diagnosis or native Hello-Pumpkin probe).

## 10) Full Directory (local + GitHub — any session reads here)
- **Local (this PC):**
  - `C:\Users\Vincent\Desktop\snapwing\PUMPKIN-COLLABORATION-WORKFLOW.md` (this file)
  - `C:\Users\Vincent\Desktop\snapwing\PUMPKIN-MIGRATION-HANDOFF.md` (server IDs, hashes, state)
  - `C:\Users\Vincent\Desktop\snapwing\crash-test.sh` (VM manual Pumpkin run)
- **Centralized on GitHub (readable by any other chat/model — git log):**
  - `https://github.com/Aevienne/Pumpkin/blob/master/PUMPKIN-COLLABORATION-WORKFLOW.md` — raw: `https://raw.githubusercontent.com/Aevienne/Pumpkin/master/PUMPKIN-COLLABORATION-WORKFLOW.md`
  - `https://github.com/Aevienne/Pumpkin/blob/master/PUMPKIN-MIGRATION-HANDOFF.md` — raw: `https://raw.githubusercontent.com/Aevienne/Pumpkin/master/PUMPKIN-MIGRATION-HANDOFF.md`
  - `git log --oneline` on `Aevienne/Pumpkin` master shows docs commits (2957246, 6ffb888) — any session `gh repo clone Aevienne/Pumpkin` gets full context and history.
- **Forks:**
  - `Aevienne/Pumpkin` fork of `Pumpkin-MC/Pumpkin` — all workflow/docs + future fix branches
  - `Aevienne/PatchBukkit` fork of `Pumpkin-MC/PatchBukkit` — bridge/JVM fixes
- Updated: 2026-09-01 — PatchBukkit disabled (`.disabled`) until JVM core-dump fixed; base Pumpkin on 25553 stable; use manual `docker run ... yolks:java_25 timeout 15 ./pumpkin` to avoid truncated panel logs.
