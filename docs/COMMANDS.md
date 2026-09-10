# Commands

[Siding](../README.md) · [Quick start](QUICKSTART.md) · [Recipe](CONTRACT.md) · [State](STATE.md) · [Releases](https://github.com/TheBlackhowling/siding-releases/releases)

Every command, its options, and what it actually does. Run `siding <cmd> --help` for the same flags from the binary.

Use the **host binary** for `up` / `down` / `reset` / `tail` / `remove` / `ps` / `proxy` so compose bind mounts are your real disk paths.

## Global

These apply to every command (persistent flags / env).

| Flag / env | What it does |
|------------|----------------|
| `--root` / `SIDING_ROOT` | Override the state **file** or the directory that contains `state.json` |
| `SIDING_DATA` | Override the user-data root (default `%LOCALAPPDATA%\siding` on Windows, `~/Library/Application Support/siding` on macOS, `~/.local/share/siding` on Linux) |
| `NO_COLOR` / `SIDING_NO_COLOR` | Disable ANSI color on stdout/stderr (also off when not a TTY) |
| `FORCE_COLOR` / `SIDING_FORCE_COLOR` | Force color even when piped (`NO_COLOR` still wins) |
| `SIDING_NO_UPDATE_CHECK` | Skip the after-command check for a newer GitHub Release |

After a successful command, a release binary may print one stderr line when
[siding-releases](https://github.com/TheBlackhowling/siding-releases/releases/latest)
has a newer stable tag (`Run: siding update`). `dev` builds, `--json`, and
`SIDING_NO_UPDATE_CHECK` skip that check. The result is cached as
`update-check.json` under the user-data directory for one hour.

Most repo commands also take:

| Flag | What it does |
|------|----------------|
| `--path` | Start directory (default: cwd). Walks **up** until `.siding/config.yml` (or `.siding.yml`) |
| `--stack` | Stack name. Default: folder basename minus recipe `folder_prefix`, else basename |
| `--mode` | Recipe mode (default `test`) |

Identity: `--path` → repo root → checkout key in `state.json` → stack name from the folder. There is no named roster in the recipe. See [STATE.md](STATE.md).

---

## `siding version`

Print version, commit, build date, and Go version.

No extra flags. See [Global](#global) for the optional newer-release notice.

---

## `siding install`

Copy **this** executable to a user bin directory and, on Windows, add that directory to your user PATH.

PowerShell does not search the current folder, so `siding` only works after the exe is on PATH (or you type `.\siding.exe`).

| Where | Path |
|-------|------|
| Windows | `%LOCALAPPDATA%\siding\bin\siding.exe` |
| macOS / Linux | `~/.local/bin/siding` |

Download the archive from [Releases](https://github.com/TheBlackhowling/siding-releases/releases/latest) first. Windows amd64/arm64 are zips; macOS and Linux are `.tar.gz` (macOS is Developer ID signed).

The install process cannot change **this** PowerShell sessionâ€™s PATH (a child process cannot). After `siding install`, prepend the bin dir in this session (or open a new Windows Terminal):

```powershell
.\siding.exe install
$env:Path = "$env:LOCALAPPDATA\siding\bin;" + $env:Path
siding version
```

**macOS / Linux:** unpack `siding` and copy it onto PATH yourself (install still works if you run it from that copy):

```bash
tar -xzf siding_*_darwin_arm64.tar.gz   # or linux / amd64
install -m 0755 siding ~/.local/bin/siding
siding version
```

No extra flags.

---

## `siding update`

Download the archive for **this** OS/arch from
[siding-releases](https://github.com/TheBlackhowling/siding-releases/releases/latest),
verify `checksums.txt`, and replace the binary in the same directory as
[`siding install`](#siding-install), not a copy you launched from Downloads.

```powershell
siding update          # latest stable
siding update v0.1.4   # that tag
```

Already current (same stable tag as this binary) prints that and exits 0
without downloading. Network / checksum failures exit non-zero and leave
the installed file unchanged. `SIDING_NO_UPDATE_CHECK` does not apply;
this command is explicit.

After a successful replace, siding prints the new `siding version` output
when the new file can run, plus the same Windows PATH reminder as `install`
if that directory is not already on PATH.

No extra flags.

---

## `siding init`

Create a committed org recipe, **or** map an existing one onto this machine.

- **No** `.siding/config.yml` → wizard (TTY) or flags: scan compose, own **every** published port, write `.siding/config.yml` + `.siding/.gitignore`, then `sync`.
- **Recipe already present** → runs [`sync`](#siding-sync) (does not overwrite). `--force` re-runs the wizard.

Never edits the OS hosts file. Does not start containers unless you run `up` yourself.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Repo to initialize |
| `--project` | folder prefix without the trailing hyphen, else folder name | Product id shared by every clone (`acme`, `hello`). Not the clone folder  -  that is the stack |
| `--folder-prefix` | guessed from `org-clone` folders | Stripped from folder names (`acme-shop` → stack `shop`). `acme-genie2` → `acme-` even when non-interactive. `none` / empty flag value disables |
| `--http-prefix` / `--host-prefix` | inferred or none | Product prefix without `app.` (`example` → `app/www/api.example.{stack}.{mode}.localhost`). `none` = `{stack}.{project}.{mode}` |
| `--entry KEY=LABEL` | `web=app` when a prefix is set, else `web`; `api=api` | Repeatable HTTP entry points. Clickable on `http://siding.localhost`. Extras: `--entry web=app,www` |
| `--compose` | all root files | Repeatable. Wizard lists root compose files; type `1,3` or `all`. Each file is its own stack except *override* files, which attach to the base |
| `--mode-name FILE=SUFFIX` | from filename | Repeatable. Suffix for that compose stack (`test`, `prod`, …). TTY prompts per file when omitted |
| `--compose-env-file MODE=PATH` | none | Repo `--env-file` **for that mode only** (not shared). TTY asks once per mode and lists only that mode's compose stack; type comma-separated **numbers** in override order (`2,1`), `none`, or `all`. Modes with no candidates print “none for this mode”. Non-interactive writes none unless you pass this flag; a warning lists leftover files **for that mode** |
| `--primary-branch` | current branch or `main` | Primary git branch written to the recipe; default start point for `worktree add` |
| `--port KEY=NUMBER` | (scan) | Override one default port. Repeatable. **Does not drop** other scanned publishes |
| `--prefer-proxy` | `true` | Host URLs via `siding proxy` (Caddy on host `:80`). Init stops if something else holds `:80` (not our own Caddy) |
| `--rewrite-compose` | `false` | Copy root compose files to `.siding/compose/` and adapt **the copies** (ports → `${…_PORT}`, public URL env, Compose DNS, shared network). Originals at the repo root are not modified |
| `--print-hosts` | `false` | Remind that `*.localhost` must not be added to the hosts file |
| `--force` | `false` | Overwrite an existing recipe (re-run wizard) |
| `--dry-run` | `false` | Show recipe / compose-copy plan; write nothing |

Interactive prompts only when stdin is a TTY **and** you did not pass the corresponding flag.

After a successful first-time write, init always `sync`s so this checkout has `state.json` + `.siding/env`.

**Check in** `.siding/config.yml`, `.siding/.gitignore`, and `.siding/compose/` if you used `--rewrite-compose`. See [Quick start](QUICKSTART.md#3-check-in-the-org-files).

```powershell
siding init --path my-app --project myapp --folder-prefix myapp- --prefer-proxy --http-prefix example --dry-run
siding init --path my-app --project myapp --folder-prefix myapp- --prefer-proxy --http-prefix example
```

---

## `siding sync`

Map the committed recipe onto **this** checkout.

- Allocate or merge ports in `state.json` (existing numbers stay; new keys skip common defaults and live Docker binds)
- Refresh host and compose project from current patterns
- Write `.siding/env` for mode `test`, and `.siding/env.<mode>` for other compose stacks
- Reload the shared proxy if it is already running

With no `--mode`, **every** recipe mode is allocated so `docker-compose.yml` and `docker-compose.prod-db.yml` can be up together. Pass `--mode NAME` to sync one.

Does **not** start containers and does **not** copy compose. `init` on a repo that already has a recipe is this command.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout to map |
| `--stack` | from folder | Override stack name |
| `--mode` | all modes | Omit to allocate every mode; pass a name to sync one |
| `--print-hosts` | `false` | Remind that `*.localhost` must not be added to the hosts file |
| `--dry-run` | `false` | Show mapping; do not write state or env |
| `--json` | `false` | JSON instead of text |

```powershell
siding sync --path my-app
siding sync --path my-app --print-hosts --dry-run
```

---

## `siding doctor`

Check the install, recipe, compose wiring, Docker Compose, and whether this checkout is in `state.json`.

Warns on hardcoded host ports, public URLs that bake `:port` into the browser, recipe `port_env` missing from compose, **unowned** compose publishes, `network_mode: host`, and in-container `localhost`. Failures (missing compose file, broken contract) make the process exit non-zero.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Where to start the walk |
| `--json` | `false` | JSON report |

```powershell
siding doctor --path my-app
```

---

## `siding plan`

Resolve stack, host, ports, and slot env **without** starting containers or writing state (reads existing allocation if present).

Useful to see which checkout key, compose project, and ports you would get.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout |
| `--stack` | from folder | Override stack name |
| `--mode` | `test` | Recipe mode |
| `--json` | `false` | JSON including `slot_env` and scan tips |

```powershell
siding plan --path my-app
siding plan --path my-app --json
```

---

## `siding add`

Pre-allocate globally unique host ports for this checkout into `state.json`. Idempotent: existing numbers are kept; new keys from the recipe/scan are merged.

New claims skip conventional ports (`3000` → `18000`, `5432` → `20432`, …) and treat **running Docker published ports** as taken. First `siding up` also allocates if needed, so `add` is optional.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout |
| `--stack` | from folder | Override stack name |
| `--mode` | `test` | Recipe mode |
| `--dry-run` | `false` | Print allocation; do not write `state.json` |
| `--json` | `false` | JSON |

If another checkout already claimed the recipe defaults, the **whole new block** shifts by 10 until it is free. Newly added keys shift individually so existing ports on this checkout do not move.

```powershell
siding add --path my-app
```

---

## `siding up`

Write `.siding/env` (or `.siding/env.<mode>`), allocate if needed, claim `{project}:{stack}:{mode}` lock, run `docker compose -p <project> -f …` **on the host**, reload the proxy if it is running.

Recipe `modes.<name>.compose_env_files` are extra `--env-file` flags **for that mode only**, placed **before** that mode's slot file (`.siding/env` / `.siding/env.<mode>`) so `${VAR}` in that stack's volumes/ports can come from a repo file (`docker/.env.prod-db`) without copying secrets into the slot file.

Caddy defaults to host `:80`. `siding proxy up` and `init --prefer-proxy` refuse that port only when something **other than** `siding-proxy-caddy` already holds it. A custom `--listen` (not 80) must already be answering or `up` tells you to start the proxy.

`-p` overrides a compose-file `name:` so two files that both say `name: local` do not share a project. Each mode has its own host ports.

Prints stack, URL, proxy status, board URL when the proxy is up, and allocated ports. Records git branch/SHA on the allocation for `http://siding.localhost`.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout |
| `--stack` | from folder | Override stack name |
| `--mode` | `test` | Recipe mode (`siding up --mode prod-db`) |
| `--profile` | empty | Compose profile (passed before the action) |
| `--build` | `false` | Pass `--build` to `docker compose up` |
| `--force-recreate` | `false` | Recreate containers even if config is unchanged |
| `--pull` | omit | `always`, `missing`, or `never` |
| `--remove-orphans` | `false` | Drop containers for services no longer in compose |
| `--` extra | | Extra args to `docker compose up` (service names, `--no-deps`, …) |

Always detached (`-d`). Rare Compose flags stay after `--`.

```powershell
siding up --path my-app
siding up --path my-app --mode prod-db
siding up --path my-app --build --force-recreate
siding up --path my-app -- --no-deps web
```

If compose fails and this process just claimed the lock, the lock is released.

---

## `siding down`

`docker compose down` on the host, release the lock, reload the proxy if running.

Keeps port allocations in `state.json` so the next `up` reuses the same binds. Named volumes stay unless you pass `--volumes`.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout |
| `--stack` | from folder | Override stack name |
| `--mode` | `test` | Recipe mode |
| `--profile` | empty | Compose profile |
| `-v` / `--volumes` | `false` | `docker compose down --volumes` |
| `--remove-orphans` | `false` | Drop containers for services no longer in compose |
| `--timeout` | omit | Shutdown timeout in seconds |
| `--` extra | | Extra args (`-- --rmi local`) |

```powershell
siding down --path my-app
siding down --path my-app -v
```

---

## `siding recycle`

Stop then start the stack for a mode:

1. `docker compose down`
2. `docker compose up -d`

Same `--path` / `--stack` / `--mode` / `--profile` as `up`. Keeps port allocations; named volumes stay unless you pass `-v` / `--volumes`. Accepts `up` flags (`--build`, `--force-recreate`, `--pull`) and `down` flags (`--timeout`). Extra args after `--` apply to the **up** half only. For wiped volumes and a forced rebuild, use [`reset`](#siding-reset).

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout |
| `--stack` | from folder | Override stack name |
| `--mode` | `test` | Recipe mode |
| `--profile` | empty | Compose profile |
| `-v` / `--volumes` | `false` | `docker compose down --volumes` |
| `--remove-orphans` | `false` | Drop containers for services no longer in compose |
| `--timeout` | omit | Shutdown timeout in seconds |
| `--build` | `false` | Pass `--build` to compose up |
| `--force-recreate` | `false` | Recreate containers even if config is unchanged |
| `--pull` | omit | Pull images before up: `always`, `missing`, or `never` |
| `--` extra | | Extra args for compose up |

```powershell
siding recycle --path my-app
siding recycle --path my-app --mode prod --build --force-recreate
siding recycle --path my-app -- --no-deps web
```

---

## `siding reset`

Wipe volumes and bring the stack back up rebuilt:

1. `docker compose down --volumes --remove-orphans`
2. `docker compose up -d --build --force-recreate --remove-orphans`

Same `--path` / `--stack` / `--mode` / `--profile` as `up`. Extra args after `--` apply to the **up** half only. If you do not want a rebuild, run `down -v` then `up` yourself.

```powershell
siding reset --path my-app
siding reset --path my-app -- --pull missing
```

---

## `siding tail`

`docker compose logs --follow` for this checkout's project. Optional service names. Does not claim or release the lock. Writes `.siding/env` first.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Checkout |
| `--stack` | from folder | Override stack name |
| `--mode` | `test` | Recipe mode |
| `--profile` | empty | Compose profile |
| `--tail` | `100` | Lines per container from the end (`all` for the whole log) |
| `--since` | omit | Timestamp or relative time (`10m`) |
| `--timestamps` | `false` | Show timestamps |
| `--` extra | | Extra `docker compose logs` args |

```powershell
siding tail --path my-app
siding tail web
siding tail web api --since 10m --timestamps
```

---

## `siding remove`

Detect the checkout from the current folder (walks up to `.siding/`, or matches `state.json` by path), **`siding down` each mode**, then delete that checkout's allocations and locks from `state.json`.

Does **not** delete repo files. `.siding/config.yml`, compose copies, and generated `.siding/env` stay on disk. The next `sync` / `up` rewrites env from the new allocation.

With no `--mode`, every recipe mode on this checkout is stopped and forgotten (plus leftover modes still in state). You do not need `--path` if you run it from inside the app repo.

| Flag | Default | What it does |
|------|---------|----------------|
| `--path` | cwd | Start folder; walks up to `.siding/` or a known checkout path |
| `--stack` | from folder | Override stack name |
| `--mode` | all modes | Forget one mode only |
| `--dry-run` | `false` | Print what would be stopped and forgotten |
| `--force` | `false` | Drop state even if `docker compose down` fails |
| `-v` / `--volumes` | `false` | Also wipe named compose volumes, then forget state |
| `--json` | `false` | JSON |

```powershell
cd my-app
siding remove
siding remove --path my-app --dry-run
siding remove --mode test
siding remove --force   # folder already half-deleted; drop state anyway
siding remove --volumes # wipe Postgres/Redis volumes, then forget the checkout
```

Use this before deleting or recloning a checkout so ports and Caddy routes are released.

---

## `siding worktree`

Manage **git worktrees** as sibling siding slots. Run from a checkout that already has a committed recipe.

### `siding worktree add NAME BRANCH [from]`

Create `{parent}/{folder_prefix}NAME` next to the current checkout, run `git worktree add`, then `siding sync` (does not `up`).

| Piece | Behavior |
|-------|----------|
| `NAME` | Siding stack / folder suffix only (`acme-shop` + `genie` → `../acme-genie`) |
| `BRANCH` | New git branch to create (`-b`). Independent of `NAME` |
| `from` | Optional start point (`main`, tag, commit). Default: recipe `primary_branch` (`main` if unset) |
| `--path` | Main checkout to anchor from (default: cwd) |

If `sync` fails after git succeeds, the folder remains  -  run `siding sync --path …` to finish.

```powershell
siding worktree add genie feat/my-work
siding worktree add genie feat/my-work main
siding worktree add genie feat/my-work v0.1.4
```

### `siding worktree remove [NAME]`

Remove by **NAME** from any checkout with the recipe (same suffix as `worktree add`), run from the linked worktree, or pass `--path` to it. Steps:

1. `siding down -v` for every mode
2. `siding remove`
3. `git worktree remove`

The primary worktree cannot be removed. On failure, siding prints what completed and manual recovery commands. `--force` continues after compose errors. `--dry-run` lists the steps only.

```powershell
siding worktree remove genie
cd my-app-genie
siding worktree remove
siding worktree remove --path my-app-genie
```

### `siding worktree list`

`git worktree list` plus siding stack name and `up` / `allocated` / `not in state.json`.

```powershell
siding worktree list
```

### `siding worktree sync`

List git worktrees and run `siding sync` on any checkout **not yet in state.json**  -  for example after a plain `git worktree add`. Does not create worktrees or run `up`. Skips bare worktrees and paths without a committed recipe.

```powershell
siding worktree sync
siding worktree sync --dry-run
```

---

## `siding ps`

`docker compose ps` for this checkout's project (uses `.siding/env`).

Same `--path` / `--stack` / `--mode` / `--profile` / `--` extra as `down`.

```powershell
siding ps --path my-app
```

---

## `siding proxy`

Machine-wide Caddy. Routes come from **all** checkouts in `state.json`, not from `--path`.

HTTP Host routes only:

| Port key | Sites |
|----------|--------|
| `web` | `{host}` (e.g. `app.acme.genie.test.localhost`), plus extra `http_hosts.web` labels |
| `api` | Sibling of a role-prefixed host (`api.acme.genie.test.localhost`), else `api.{host}`, plus `http_hosts.api` |

Redis, Postgres, Firebase, … stay Docker host ports in `.siding/env`. They are not Caddy sites (TCP has no `Host` header).

Reserved (never allocated to a stack): **80**, **2019**.

Caddy listens on host `:80` by design. One listener, Host matchers: `siding.localhost` is the board, each stack Host reverse-proxies its allocated port, anything else is 404 (not a redirect to the board). Upstream is `host.docker.internal:{allocated}`. Files live under `{user-data}/siding-proxy/` (Caddyfile + compose + `board/status.json`). The board UI is `ghcr.io/theblackhowling/siding-proxy:latest`. Our own proxy already on `:80` is not a conflict.

### `siding proxy up`

Start (or recreate) the proxy container. Always `--force-recreate` and `--pull always` so a new Caddyfile replaces an already-running container and a newer board image is picked up (plain `up -d` would keep the old redirect and a stale `:latest`). An image-only upgrade is this command; `proxy reload` does not pull. The image is `ghcr.io/theblackhowling/siding-proxy:latest` and must stay a **public** GHCR package so zip-only machines can pull it.

| Flag | Default | What it does |
|------|---------|----------------|
| `--listen` | `80` (migrates leftover `:1355`) | Host HTTP port. Public URLs are portless on 80 |
| `--json` | `false` | JSON |

```powershell
siding proxy up
siding proxy up --listen 1365
```

Prints `board: http://siding.localhost`. After changing listen, `proxy down` then `proxy up`. A leftover listen of `1355` is migrated to `80` on the next `proxy up` (unless you pass `--listen`). Our own Caddy already on `:80` is not treated as busy.

### `siding proxy reload`

Rewrite the Caddyfile from `state.json` and POST it to Caddy's admin API (`127.0.0.1:2019`). Fallback: `docker exec` reload. The proxy container is `--force-recreate`d only when the generated compose file changed (listen / published ports).

`up` / `down` / `sync` already reload if the proxy is running.

| Flag | What it does |
|------|----------------|
| `--json` | JSON |

### `siding proxy down`

Stop the proxy compose project. Stack allocations are unchanged.

### `siding proxy ps`

Show the proxy container (`siding-proxy-caddy`).
