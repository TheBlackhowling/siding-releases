# Quick start

[Siding](../README.md) Â· [Commands](COMMANDS.md) Â· [Recipe](CONTRACT.md) Â· [State](STATE.md)

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

See [Commands â†' install](COMMANDS.md#siding-install).

Work from the app checkout for the rest of this page. Pass `--path` only when
you are somewhere else.

```powershell
```

## 1. Point at the app repo

From the app checkout, or pass `--path`:

```powershell
siding doctor
```

`doctor` is allowed to fail until a recipe exists.

## 2. First checkout: write the org recipe

Dry-run first. Scan will try to **own every published compose port** (web, api, redis, postgres, firebase, …). That is required  -  skipping them is how parallel stacks collide.

```powershell
siding init --dry-run
```

Then run it for real. Typical product clone (`acme-shop` → stack `shop`):

```powershell
siding init --project myapp --folder-prefix myapp- --prefer-proxy
```

On a TTY the wizard asks:

1. Which compose files at the **repo root** to process (numbered list  -  `all` or `1,3`). Nested compose files are ignored. Each file becomes its own stack except `*override*` files, which stay with the primary compose file.
2. A **suffix** per stack (`test`, `prod`, …) used in hosts and `siding up --mode`. Guess from the filename.
3. **Repo env files per mode**  -  each mode is asked separately. Only files for **that** mode's compose stack are listed. These are `docker compose --env-file` on `siding up --mode NAME`, not `.siding/env`. Flag: `--compose-env-file prod=docker/.env.prod-db`.
4. Shared Caddy proxy (host `:80`) vs raw published ports (prefer the proxy).
5. **Public host prefix**  -  product name only (`example`), not `app.example`. Subdomains `app`, `www`, and `api` all work: `app.example.{stack}.test.localhost`. `none` keeps `{stack}.{project}.{mode}`. Compose can guess `example` from `app.example.test`. Do not add `*.localhost` to the hosts file.
6. **HTTP entry points**  -  subdomain for each web container (`web` → `app`, `api` → `api`). Those names are the clickable links on `http://siding.localhost`. Comma extras (`app,www`) add more links. Flag: `--entry web=app --entry api=api`.
7. Whether to copy those compose files into `.siding/compose/` and adapt the copies (`${WEB_PORT:-…}`, public URL env, Compose DNS). **Originals stay at the repo root** so `docker compose` without siding still works. On a large YAML-anchor compose, answer **no** the first time  -  env overrides on the original often suffice.

Flags skip the prompts: `--compose`, `--mode-name FILE=SUFFIX`, `--compose-env-file MODE=PATH`, `--prefer-proxy`, `--http-prefix`, `--entry KEY=LABEL`, `--rewrite-compose`.

If `.siding/config.yml` already exists, `init` **syncs** it onto this machine instead of overwriting (`--force` re-runs the wizard).

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
| `.siding/env` / `.siding/env.*` | This checkout's allocated ports and host (one file per mode) |
| `.siding/backups/` | Leftover from older in-place rewrite (not used anymore) |
| `%LOCALAPPDATA%\siding\state.json` | Machine-wide allocations |

Push the branch. The next person clones, runs `siding init` (or `siding sync`), and gets their own ports.

## 4. Shared proxy (once per machine)

Host URLs via Caddy (`http://app.<host>` on port 80):

```powershell
siding proxy up
```

Caddy listens on host `:80`. Admin stays on `:2019`. Recreate after listen changes: `siding proxy down` then `siding proxy up --listen …`.

## 5. Start the stack

```powershell
siding up
siding up --build --force-recreate
siding tail web
siding ps
siding doctor
```

`up` writes `.siding/env`, claims a lock, and runs `docker compose` on the host. If something other than siding's Caddy holds `:80`, `up` tells you to free it or run without the proxy. Open the `url:` line it prints (`http://app.<host>`).

Stop with `siding down`. Allocated ports stay in `state.json` so the next `up` reuses them. `siding down -v` also deletes named volumes. `siding recycle` is `down` then `up` on the same mode (volumes kept). `siding reset` is `down -v` then `up --build --force-recreate`.

Before deleting or recloning a checkout, forget it from this machine (detects the repo from cwd):

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

`init` here is `sync`: map the committed recipe onto this folderâ€™s ports and
`.siding/env`.

## Git worktrees

Sibling folders for parallel branches (same repo, different stack names):

```powershell
siding worktree add genie feat/my-work
siding worktree list
siding worktree sync          # adopt a plain git worktree add
siding worktree remove genie  # from the main checkout
```

Close the linked folder as a Cursor workspace before `worktree remove` on
Windows â€” an open workspace can block deleting the directory.
## If it fights you

- Stop whatever already bound `80` / `2019` / `3000` / `5432` / `8080` before the first `up` (our own `siding-proxy-caddy` on `:80` is fine).
- Do not mix another allocator (shell `$env:WEB_PORT` wins over `.siding/env`).
- Run `siding doctor --path …` and fix `unowned_port` / `port_hardcoded` before expecting two checkouts to coexist.
- Full flag list: [Commands](COMMANDS.md).
