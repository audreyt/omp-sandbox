# omp-sandbox

A macOS [Seatbelt](https://www.chromium.org/developers/design-documents/sandbox/osx-sandboxing-design/) wrapper for [Oh My Pi (omp)](https://omp.dev) that applies Seatbelt filesystem and process-metadata restrictions to the AI agent — confining file writes to a workspace and explicit exceptions, denying most `$HOME` reads, and denying host process enumeration — without patching omp or adding any runtime dependencies.

> **Isolation model — read this first.**
> This is **host macOS Seatbelt**, not a VM or container. OMP and all child commands run at the **same UID and in the same OS session** as the invoking shell. Seatbelt restricts file I/O and process metadata (see below), but does **not** provide process separation, network egress filtering, or credential isolation from same-UID children. **For strong isolation, use a separate VM or OS user account.**
>
> - **OMP-injected environment variables** (`PI_SESSION_FILE`, `PI_TOOL_BRIDGE_URL`, `PI_TOOL_BRIDGE_TOKEN`, `PI_TOOL_BRIDGE_SESSION`) are injected by OMP itself into OMP-managed command/tool children's environment. This wrapper cannot suppress them without breaking the tool bridge. `PI_SESSION_FILE` was observed mode `0644`; OMP should create it `0600` for cross-user defense, but that would not isolate same-UID sandbox children either way.
> - **Process metadata is now denied** except for the process itself (see table and security model below). Same-UID signals and IPC remain unrestricted.
> - **Executable denylists are not a security boundary** in Seatbelt.
> - **Network egress is fully open** — all hosts are reachable. An external proxy or firewall is required for network control.

## What it does

| Boundary | Policy |
|---|---|
| **Write** | Allowed only in your workspace + OMP's own state dirs + macOS runtime scratch |
| **Read ($HOME)** | Allowlist: `~/.omp` and your workspace only — everything else in `$HOME` is denied |
| **Read (system)** | Open via `allow default` — OMP's runtimes need `/usr`, `/System`, `/Library`, `/opt/homebrew`, etc. |
| **Network** | Open — Seatbelt cannot do per-domain filtering; OMP needs model APIs |
| **Process metadata** | Denied for host processes; allowed for self only (`process-info* (target self)`) |
| **YOLO** | Opt-in (`OMP_SANDBOX_YOLO=1`) — auto-approves all tool calls |

Personal data that is read-denied by default: Keychain, Messages, Mail, Calendars, Contacts, iCloud Drive, Safari, Health, Wallet, Passes, and all other `$HOME` paths not in the allowlist.

## Requirements

- macOS (any version shipping `/usr/bin/sandbox-exec`)
- [Oh My Pi](https://omp.dev) installed and on `PATH` (e.g. `brew install omp`)
Compatible with Apple's stock `/bin/bash` (3.2.57) — array expansions use the `${arr[@]+"${arr[@]}"}` idiom to avoid the bash-3.2 empty-array `unbound variable` crash.

## Quick start

```bash
# 1. Clone
git clone https://github.com/bnivanov/omp-sandbox ~/.omp/sandbox

# 2. Make executable
chmod +x ~/.omp/sandbox/omp-sandboxed

# 3. Verify it works from your project directory
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed --self-test

# 4. Run omp inside the sandbox
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed

# 5. Optional: add a shell alias
echo 'alias omp-sandbox="~/.omp/sandbox/omp-sandboxed"' >> ~/.zshrc
```

## Configuration

All configuration is via environment variables — no config files.

| Variable | Default | Purpose |
|---|---|---|
| `OMP_SANDBOX_WORKSPACE` | `$PWD` | Directory OMP can read and write. Defaults to wherever you run the script from. |
| `OMP_SANDBOX_YOLO` | `0` | Set to `1` to pass `--auto-approve` to omp (no tool-approval prompts). Use only in sandboxed sessions. |
| `OMP_SANDBOX_EXTRA_READ` | _(empty)_ | Colon-separated extra `$HOME`-subtree paths to make readable. Leading `~` expands to `$HOME`. Mirrors `EXTRA_WRITE`: same per-path canonicalization and same sensitive-path denylist. |
| `OMP_SANDBOX_CONFIRM_SENSITIVE_READ` | `0` | Set to `1` to allow sensitive paths (`.ssh`, `.aws`, Keychains, etc.) in `OMP_SANDBOX_EXTRA_READ`. Reading secrets is as dangerous as writing them — insecure. |
| `OMP_SANDBOX_EXTRA_WRITE` | _(empty)_ | Colon-separated extra writable paths. Leading `~` expands to `$HOME`. Each path is canonicalized and checked against a denylist (`.ssh`, `.aws`, Keychains, etc.). |
| `OMP_SANDBOX_PASS_ENV` | _(empty)_ | **Colon-separated env var names to forward into the clean sandbox env. Required for provider API keys** — e.g. `OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY`. Without this, OMP cannot reach its model API. |
| `OMP_SANDBOX_INHERIT_ENV` | `0` | Set to `1` to pass the full parent environment instead of the minimal allowlist. Insecure — exposes all shell secrets to the sandboxed process and any host it reaches. |
| `OMP_SANDBOX_CONFIRM_WIDE` | `0` | Set to `1` to allow `$HOME` or `/` as workspace (otherwise rejected). Insecure. |
| `OMP_SANDBOX_SKIP_HOME_VERIFY` | `0` | Set to `1` to skip the `dscl` home-directory verification. Insecure — allows HOME-poisoning attacks. |
| `OMP_SANDBOX_CONFIRM_SENSITIVE_WRITE` | `0` | Set to `1` to allow sensitive paths (`.ssh`, `.aws`, Keychains, etc.) in `OMP_SANDBOX_EXTRA_WRITE`. Insecure. |

Examples:

```bash
# Fix the workspace to a specific path regardless of cwd.
# Unquoted ~ is expanded by the shell; the script also handles a literal ~ if you
# export the variable (export OMP_SANDBOX_WORKSPACE='~/proj' works too).
OMP_SANDBOX_WORKSPACE=~/projects/myproject ~/.omp/sandbox/omp-sandboxed

# YOLO mode — auto-approve all tool calls
OMP_SANDBOX_WORKSPACE=~/projects/myproject OMP_SANDBOX_YOLO=1 ~/.omp/sandbox/omp-sandboxed

# Allow omp to write screenshots to ~/Downloads in addition to the workspace
OMP_SANDBOX_EXTRA_WRITE=~/Downloads ~/.omp/sandbox/omp-sandboxed

# Allow omp's mnemon memory store: recall + remember (read+write)
# ~/.mnemon must be passed to BOTH EXTRA_WRITE (for remember) and EXTRA_READ.
OMP_SANDBOX_EXTRA_WRITE=~/.mnemon OMP_SANDBOX_EXTRA_READ=~/.mnemon ~/.omp/sandbox/omp-sandboxed

# Pass provider API key into the minimal clean environment (required by default)
OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY OMP_SANDBOX_WORKSPACE=~/projects/myproject \
  ~/.omp/sandbox/omp-sandboxed

# Multiple keys, colon-separated
OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY:OPENAI_API_KEY ~/.omp/sandbox/omp-sandboxed

# Legacy: pass full parent environment — exposes all shell secrets (insecure)
OMP_SANDBOX_INHERIT_ENV=1 ~/.omp/sandbox/omp-sandboxed
```

## Security model

### Write allowlist

Only these locations are writable inside the sandbox:

| Path | Reason |
|---|---|
| `$OMP_SANDBOX_WORKSPACE` | Your project — OMP's working directory |
| `~/.omp` | OMP's sessions, memories, and logs |
| `/private/tmp`, `$TMPDIR` | macOS temp files (Bun, Python) |
| `/private/var/folders` | Bun/Python bytecode caches |
| `/dev` | Device nodes (`/dev/null`, `/dev/urandom`, pipes) |
| `$OMP_SANDBOX_EXTRA_WRITE` | Opt-in extra paths |
| `~/.mnemon` (via `EXTRA_WRITE`+`EXTRA_READ`) | OMP's persistent memory store (mnemon). **Opt-in**: set `OMP_SANDBOX_EXTRA_WRITE=~/.mnemon` and `OMP_SANDBOX_EXTRA_READ=~/.mnemon` to enable recall+remember under the sandbox. |

### Read allowlist (within `$HOME`)

Only these `$HOME` paths are readable:

- `~/.omp` — OMP configuration, memories, session state
- `$OMP_SANDBOX_WORKSPACE` — your project
- `$OMP_SANDBOX_EXTRA_READ` — opt-in extra `$HOME`-subtree paths; canonicalized and sensitive-path-checked the same way as `EXTRA_WRITE` (so `EXTRA_READ=~/.ssh` requires `OMP_SANDBOX_CONFIRM_SENSITIVE_READ=1`). Use it when a tool needs to read a `$HOME` path that is neither `~/.omp` nor the workspace (e.g. a notes dir, a model cache).
- `~/.mnemon` (via `EXTRA_READ`+`EXTRA_WRITE`) — OMP's persistent memory store (mnemon). **Opt-in**: pass `~/.mnemon` in both `OMP_SANDBOX_EXTRA_READ` and `OMP_SANDBOX_EXTRA_WRITE` to enable `mnemon recall`/`mnemon remember` inside the sandbox. No `MNEMON_DATA_DIR` env forward is needed: with the store at its default `~/.mnemon` (a real directory, not a symlink), mnemon resolves the path itself; its only `$HOME` ancestor is `$HOME`, already covered by the home-node metadata rule.

All other `$HOME` paths are denied: `~/.ssh`, `~/.aws`, `~/.zsh_history`, `~/.docker`, `~/.kube`, `~/Library/Messages`, `~/Library/Mail`, `~/Library/Keychains`, `~/Library/Mobile Documents` (iCloud), `~/Library/Health`, `~/Library/Passes`, Safari, Contacts, and everything else.

System paths outside `$HOME` remain readable — OMP's runtimes require them.

### Home-node metadata (stat of `$HOME` itself)

The `$HOME` read-confinement rule uses `(subpath ...)`, which matches the node **and** its descendants. The read re-allows above, however, use `(subpath ...)` on **specific subtrees** (`~/.omp`, the workspace) — they cover descendants of those subtrees but not the `$HOME` node itself.

That gap breaks **Go programs** that canonicalize paths by walking from the root: `filepath.EvalSymlinks` and `os.MkdirAll` issue an explicit `stat()` on `/Users/<you>` during traversal. Seatbelt denies that stat, so Go aborts `MkdirAll` with `mkdir /Users/<user>: file exists` or surfaces SQLite `CANTOPEN (14)` — even when the leaf file lives in an allowlisted subtree. C binaries that just `open()` the leaf rely on kernel path traversal (which routes outside the literal `$HOME` rule) and never expose the gap, which is why the symptom looks language-specific.

The profile therefore adds one rule, immediately after the read re-allow block:

```scheme
(allow file-read-metadata (literal "$HOME"))
```

Notes:
- `file-read-metadata` is the narrowest read class — `stat`/`lstat`/`fstat`/`getattrlist` only. It does **not** include `file-read-data` (so `readdir` of `$HOME` — listing the top-level names — remains denied) or `file-write*`. Self-tests #11 (stat is allowed) and #12 (readdir is denied) lock this distinction in; widening the rule to `file-read*` to make #11 pass a different way immediately flips #12.
- `(literal "$HOME")` matches the node exactly, **not** a subpath, so descendants outside the explicit allowlists stay denied (`~/.ssh`, `~/Library/Keychains`, etc.).
- The only information disclosed is the `$HOME` directory inode's metadata (mode, owner, timestamps) — already world-readable on macOS via `ls -ld $HOME` outside any sandbox policy.

### Environment isolation

By default the launcher uses `env -i` to give the sandboxed process only a minimal allowlist (PATH, HOME, TMPDIR, locale, terminal variables). All other shell variables — AWS credentials, GitHub tokens, and any other secrets in your shell — are **stripped automatically**.

**OTel trace export is intentionally disabled in clean mode.** The wrapper hardcodes `OTEL_SDK_DISABLED=true` and `OTEL_TRACES_EXPORTER=none` in the clean env. Without these the user's own OTel kill-switches (set in `.zshrc`/`.bashrc`) are stripped by `env -i`, which exposed a Bun `node:http2` crash on first launch in a telemetry path launched or bundled by OMP (`TypeError: The "authority" argument must be of type string — received number 825110816`, where `825110816` = `0x312e3120` = ASCII `"1.1 "` — a protocol-line byte sequence). The exact downstream component is unidentified [INFERENCE]; public reproductions matching the number and stack are Claude Code OTel gRPC issues. OTEL endpoint and header variables are not forwarded; only the two standard disable flags are hardcoded.

OMP needs its model provider key to operate. Pass it explicitly:

```bash
OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY ~/.omp/sandbox/omp-sandboxed
```

`OMP_SANDBOX_INHERIT_ENV=1` passes the full parent environment instead of the minimal allowlist — **insecure**: all shell secrets are exposed to the sandboxed process and every host it can reach. Use only for debugging.

### HOME verification

The launcher resolves the real user home via `/usr/bin/dscl` and **rejects launch** if the inherited `$HOME` differs — this is the exact vector for read-confinement bypass (a malicious wrapper could set `HOME=/tmp` before launch). Set `OMP_SANDBOX_SKIP_HOME_VERIFY=1` to override.

### Process metadata isolation

The profile immediately after `(allow default)` adds:

```scheme
(deny process-info*)
(allow process-info* (target self))
```

`process-info*` covers all process metadata operations: listing host PIDs (`proc_listpids`, `KERN_PROC_ALL`), reading argv/environment of another process (`proc_pidinfo`), and path lookup (`proc_pidpath`). Without this rule any sandboxed child could enumerate all running processes and read their metadata.

`(allow process-info* (target self))` re-allows the process to query its own metadata, which is required by normal startup (dyld, crash reporters, signal handling).

**Residual:** Same-UID `kill` and SHM/IPC are **not** covered by `process-info*`. A sandboxed child at the same UID can still signal the parent shell. There is no expressible Seatbelt rule to restrict signal direction between same-UID processes without also blocking OMP's own child management. Separate VM or OS user is required for full process separation.

Proven on this configuration: `/bin/ps -axo pid=` exits rc71 (denied); `proc_listpids` count = 0; `proc_pidpath` returns 0 for arbitrary PIDs while succeeding for self. Self-test 5b asserts this behavior on every run.

### Known remaining surface

- **`/Volumes`** — mounted external/network drives are readable. Add `(deny file-read* (subpath "/Volumes"))` inside `gen_profile` if needed.
- **Other `/Users/*` accounts** — readable on a multi-user Mac. Acceptable for a single-user personal machine.
- **Network** — all hosts reachable. Use an external proxy/filter for egress control.
- **`~/.omp`** — writable by the sandboxed process. A compromised agent could modify OMP's own memories and sessions.
- **Same-UID PIDs/signals** — process-info* prevents metadata enumeration, but same-UID `kill` and SHM/IPC remain unrestricted. A separate OS user or VM is required for full process separation.
- **OMP-injected bridge/session env vars** — `PI_SESSION_FILE`, `PI_TOOL_BRIDGE_URL`, `PI_TOOL_BRIDGE_TOKEN`, `PI_TOOL_BRIDGE_SESSION` are injected by OMP into OMP-managed command/tool children's environment. This wrapper cannot suppress them without breaking the tool bridge. `PI_SESSION_FILE` was observed mode `0644`; OMP should create it `0600` for cross-user defense, but that would not isolate same-UID sandbox children (they share the Seatbelt profile that allows `~/.omp`).
- **Local service endpoints** — the tool bridge runs on a private loopback port reachable by any sandboxed child. Seatbelt is not a per-host/domain egress firewall; blocking local loopback would break the bridge.

### What this does NOT protect against — non-VM threat model

This wrapper applies a macOS Seatbelt (TrustedBSD MAC) policy to the OMP process tree. It is **not** a VM, container, or separate kernel namespace. That distinction produces unavoidable residual risks:

| Risk | Reason | Mitigation outside this wrapper |
|---|---|---|
| **OMP-injected bridge/session credentials** | `PI_TOOL_BRIDGE_TOKEN`, `PI_TOOL_BRIDGE_URL`, `PI_TOOL_BRIDGE_SESSION`, `PI_SESSION_FILE` are injected by OMP into tool-child processes at runtime, inside the sandbox. They do not originate in the launching shell; `env -i` never sees them. No supported OMP config knob suppresses this injection. | Separate UID or VM/container boundary |
| **Same-UID child file access** | OMP and all its child commands run as the same OS user. The Seatbelt profile allows `~/.omp`; any sandboxed child process therefore has the same read/write access to session files, history, and logs as OMP itself. `chmod 0600` on individual files protects against *other local users* only, not same-UID children. | Separate UID or VM/container boundary |
| **Tool bridge loopback reachability** | The OMP tool bridge listens on a private loopback port. Seatbelt has no per-host/port egress filtering; any sandboxed child can connect to it. Blocking loopback would break the bridge. | VM/container with private network namespace |
| **Network exfiltration** | OMP and all child processes can reach any network host. Seatbelt cannot do per-domain filtering. | External proxy/firewall for egress control |
| **Same-UID signals and IPC** | `process-info*` blocks metadata enumeration, but same-UID `kill` and SHM/IPC are unrestricted. No expressible Seatbelt rule restricts signal direction between same-UID processes without also breaking OMP's own child management. | Separate UID or VM/container boundary |
| **`~/.omp` store integrity** | `~/.omp` is intentionally writable; a compromised agent can modify OMP memories, sessions, and config. | Out of scope for a read/write sandbox |
| **Executable controls** | Path-based execution controls are ineffective; any binary can be copied into the writable workspace. Seatbelt has no code-signing enforcement in this profile. | OS-level mandatory access control or VM |

**Recommendation for high-risk workloads:** run OMP inside a VM or container with a private network namespace, a dedicated low-privilege UID, and an egress-filtering firewall. This wrapper materially narrows the *filesystem* attack surface on a shared machine and blocks host process enumeration, but it cannot substitute for OS-level UID or namespace isolation.

## Verifying the sandbox

```bash
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed --self-test
```

All lines should read `PASS:` and the script exits 0. The self-test checks:

- Workspace writes succeed
- `$HOME` writes outside the workspace are denied (genuine Seatbelt denial, not just a permissions error)
- `~/.omp` is both readable and writable
- Workspace reads work (proves the `$HOME` deny + re-allow rule resolves correctly)
- Keychain, Messages, and iCloud reads are denied
- Host process enumeration via `/bin/ps -axo` is **denied** by the `process-info*` rule; ordinary child execution (`/bin/bash -c 'echo ok'`) succeeds (`process-info* (target self)` allows self-inspection)
- `omp --version` boots cleanly under the minimal clean environment (same as real launches)
- `--auto-approve` flag is accepted by the installed omp version
- `OMP_SANDBOX` and `PI_SANDBOX` env markers are present inside the clean env
- `TMPDIR` canonicalizes to a valid macOS scratch root, not an attacker-controlled path
- A secret canary is absent from the `env -i` sandbox — env isolation is verified end-to-end
- `.ssh` is detected as a sensitive `EXTRA_WRITE` target and rejected by the denylist
- `stat()` of the `$HOME` node itself is allowed (Go path-canonicalization regression — without it, any Go binary opening an allowlisted `$HOME` subtree fails with `mkdir /Users/<user>: file exists` or SQLite `CANTOPEN`)
- `readdir` of the `$HOME` node itself stays DENIED — companion to the above; proves the carve-out is metadata-only (passes under `file-read-metadata`, fails under a widened `file-read*`), so a future change that broadens the rule and starts disclosing top-level `$HOME` entry names flips this assertion
- `OMP_SANDBOX_EXTRA_READ`'s sensitive-path gate fires end-to-end — `EXTRA_READ=~/.ssh` aborts profile generation with the sensitive-deny error unless `OMP_SANDBOX_CONFIRM_SENSITIVE_READ=1` is set, in which case `~/.ssh` appears in the read allowlist as a `(subpath ...)` rule. Mirrors the `.ssh`-as-`EXTRA_WRITE` test; proves the new read-side opt-in path is gated the same way as `EXTRA_WRITE` (reading secrets is as bad as writing them)
- an `OMP_SANDBOX_EXTRA_READ` path is actually **readable inside the sandbox** — with `EXTRA_READ` unset, `cat` of a canary file under a `$HOME` subdir that no baseline allowlist covers is denied by the `$HOME` read-deny rule; with `EXTRA_READ=that subdir`, `cat` succeeds (the `(subpath ...)` rule opened the `$HOME`-denied path). Placing the canary inside `$HOME` is what makes this meaningful — outside `$HOME` everything is already readable via `(allow default)`, so an outside-`$HOME` canary would pass either way and prove nothing. Mirrors self-tests #3/#4 (direct `sandbox-exec -f "$PROFILE" /bin/bash -c 'cat ...'`, not through omp).
- the omp memory store is **reachable inside the sandbox** — when `mnemon` is installed and `~/.mnemon` exists, `mnemon recall` opens the database under a profile generated with `~/.mnemon` passed via `OMP_SANDBOX_EXTRA_WRITE`+`EXTRA_READ` (proving the `EXTRA_*` mechanism and the `$HOME`-node stat rule compose to cover the mnemon case the same way any user opt-in does). The self-test folds `~/.mnemon` into `EXTRA_*` automatically; in production you set them yourself. Skipped if `mnemon` is absent or the store dir does not exist.
- `OTEL_SDK_DISABLED=true` and `OTEL_TRACES_EXPORTER=none` are present in the clean env seen by sandbox children — regression guard for the Bun `node:http2` first-launch crash (authority-as-integer `825110816`)

## Printing the active profile

```bash
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed --print-profile
```

The profile is plain [SBPL](https://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf) (~30 lines). Review it to understand exactly what is and is not permitted.

## How it works

macOS ships `/usr/bin/sandbox-exec` on every Mac (backed by the TrustedBSD MAC framework, known internally as Seatbelt). It applies a profile written in SBPL (Sandbox Profile Language) to a process and all its children — no root required, no VM, no containers.

This wrapper:
1. Computes a per-launch SBPL profile from the current workspace path and environment
2. Writes it to a temp file
3. Invokes `sandbox-exec -f <profile> omp [args]`

The profile uses `(allow default)` as the base, then applies targeted write denials + re-allows, and a `$HOME` read deny + minimal re-allow. Rule evaluation is last-match-wins within each operation class.

`sandbox-exec` is officially deprecated by Apple but remains present and functional on all current macOS releases. There is no replacement API for unprivileged sandboxing of arbitrary binaries.

## License

MIT — see [LICENSE](LICENSE).
