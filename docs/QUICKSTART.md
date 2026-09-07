# Quick start

[Siding](../README.md) · [Commands](COMMANDS.md) · [Recipe](CONTRACT.md) · [State](STATE.md)

Get one checkout running and check in the org recipe so everyone else can
`sync`. Docker Desktop must be running.

Download a binary from
[Releases](https://github.com/TheBlackhowling/siding-releases/releases/latest)
(Windows `.zip`, macOS/Linux `.tar.gz`). macOS builds are Developer ID signed.

**Windows:** PowerShell will not run `siding` from the current folder (no
implicit `.\`). Install once:

```powershell
.\siding.exe install
$env:Path = "$env:LOCALAPPDATA\siding\bin;" + $env:Path
siding version
```

That copies the exe to `%LOCALAPPDATA%\siding\bin` and updates your user PATH.
New Windows Terminal windows pick it up; Cursor/IDE terminals keep the old
PATH until the IDE restarts.

**macOS / Linux:** unpack `siding` onto PATH (`~/.local/bin` is typical), or
run `./siding install` from the unpacked binary.

See [Commands → install](COMMANDS.md#siding-install).

Work from the app checkout for the rest of this page. Pass `--path` only when
you are somewhere else.

```powershell
cd my-app
```

## 1. Look at the repo

```powershell
siding doctor
```

`doctor` is allowed to fail until a recipe exists.

## 2. First checkout: write the org recipe

Dry-run first. Scan will try to **own every published compose port** (web,
api, redis, postgres, firebase, …). That is required. Skipping them is how
parallel stacks collide.

```powershell
siding init --dry-run
```

Then run it for real. On a TTY the wizard asks, in this order:

1. **Folder prefix** (only if the folder looks like `org-clone`, e.g.
   `acme-shop` → `acme-`). Stack name is the folder basename with that prefix
   stripped. `none` keeps the full folder name.
2. **Project name** (product id shared by every clone). Default is the prefix
   stem (`acme`), not the clone folder.
3. **Which compose files** at the **repo root** (`all` or `1,3`). Nested
   compose files are ignored. Each file is its own stack except `*override*`
   files, which stay with the primary compose file.
4. **Mode suffix** per stack (`test`, `prod`, …) for hosts and
   `siding up --mode`. Guessed from the filename. Bare `siding up` uses `test`
   when that suffix exists.
5. Shared **Caddy proxy** on host `:80` (default yes). Without it you must
   remember which allocated port belongs to which name. If something other
   than `siding-proxy-caddy` already holds `:80`, init asks you to free it.
6. Whether to **copy** those compose files into `.siding/compose/` and adapt
   the copies (`${WEB_PORT:-…}`, public URL env, Compose DNS). Originals stay
   at the repo root so `docker compose` without siding still works. Default
   is yes only when the scan found problems (hardcoded ports, in-container
   `localhost`, …). On a large YAML-anchor compose, answer **no** if env
   overrides on the original already suffice.
7. **Public host prefix:** product name only (`example`), not `app.example`.
   Subdomains `app`, `www`, and `api` all work:
   `app.example.{stack}.test.localhost`. `none` keeps
   `{stack}.{project}.{mode}`. Compose can guess `example` from
   `app.example.test`. Siding does not write the OS hosts file and tells you
   not to add `*.localhost` there; browsers already resolve those names.
8. **HTTP entry points:** subdomain for each web container (`web` → `app`
   when a prefix is set, else `web`; `api` → `api`). Those names are the
   clickable links on `http://siding.localhost`. Comma extras (`app,www`)
   add more links.

Passing a flag skips that prompt: `--folder-prefix`, `--project`, `--compose`,
`--mode-name FILE=SUFFIX`, `--prefer-proxy` / `--prefer-proxy=false`,
`--rewrite-compose` / `--rewrite-compose=false`, `--http-prefix`,
`--entry KEY=LABEL`.

`--prefer-proxy` defaults to true. A scripted first init:

```powershell
siding init --project acme --folder-prefix acme- --http-prefix example
```

If `.siding/config.yml` already exists, `init` **syncs** it onto this machine
instead of overwriting (`--force` re-runs the wizard). A successful first-time
write also `sync`s, so this checkout already has `state.json` and `.siding/env`.

## 3. Check in the org files

Commit what other clones need. Leave machine-local files out.

**Commit**

| File | Why |
|------|-----|
| `.siding/config.yml` | Org recipe: modes, compose files, **all** port keys, `port_env`, public URL templates |
| `.siding/.gitignore` | Ignores generated `env`, `env.*`, and `backups/` (init writes this) |
| `.siding/compose/` (if you accepted copies) | Siding-adapted compose. Originals at the repo root stay for non-siding users |

```powershell
git add .siding/config.yml .siding/.gitignore
git add .siding/compose/   # only if init wrote copies
git status                 # .siding/env must stay untracked
```

**Do not commit**

| Path | Why |
|------|-----|
| `.siding/env` / `.siding/env.*` | This checkout’s allocated ports and host (one file per mode) |
| `.siding/backups/` | Leftover from older in-place rewrite (not used anymore) |
| `%LOCALAPPDATA%\siding\state.json` | Machine-wide allocations (macOS: `~/Library/Application Support/siding/state.json`) |

Push the branch. The next person clones, runs `siding init` (or `siding sync`),
and gets their own ports.

## 4. Shared proxy (once per machine)

Host URLs via Caddy (`http://app.<host>` on port 80). Use this. Without Caddy
you must remember which allocated port belongs to which name.

```powershell
siding proxy up
```

Caddy listens on host `:80`. Admin stays on `:2019`. The board is
`http://siding.localhost`. Recreate after listen changes: `siding proxy down`
then `siding proxy up --listen …`.

## 5. Start the stack

```powershell
siding up
siding up --build --force-recreate
siding tail web
siding ps
siding doctor
```

`up` writes `.siding/env`, claims a lock, and runs `docker compose` on the
host. If something other than siding’s Caddy holds `:80`, `up` tells you to
free it or run without the proxy. Open the `url:` line it prints
(`http://app.<host>` when you chose a prefix).

Stop with `siding down`. Allocated ports stay in `state.json` so the next `up`
reuses them. `siding down -v` also deletes named volumes. `siding reset` is
`down -v` then `up --build --force-recreate`.

Before deleting or recloning a checkout, forget it from this machine:

```powershell
siding remove
```

## Next clone of the same product

No wizard. The recipe is already in git:

```powershell
cd my-app-other
siding init
siding proxy up    # if this machine does not have it yet
siding up
```

`init` here is `sync`: map the committed recipe onto this folder’s ports and
`.siding/env`.

## If it fights you

- Stop whatever already bound `80` / `2019` / `3000` / `5432` / `8080` before
  the first `up` (our own `siding-proxy-caddy` on `:80` is fine).
- Do not mix another allocator (shell `$env:WEB_PORT` wins over `.siding/env`).
- Run `siding doctor` and fix `unowned_port` / `port_hardcoded` before expecting
  two checkouts to coexist.
- Full flag list: [Commands](COMMANDS.md).
