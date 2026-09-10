# state.json schema (draft)

Single machine-wide file under the user data directory (see [README](../README.md)). Checkout identity is the **normalized absolute repo root path**, not a workspace slug.

Command behavior: [Commands](COMMANDS.md). First run: [Quick start](QUICKSTART.md).

## Location

| OS | Default |
|----|---------|
| Windows | `%LOCALAPPDATA%\siding\state.json` |
| macOS | `~/Library/Application Support/siding/state.json` |
| Linux | `~/.local/share/siding/state.json` |

Override with `SIDING_ROOT` (file or directory) or `siding --root`.

## Example

```json
{
  "version": 1,
  "proxy": { "listen": 80 },
  "checkouts": {
    "C:/Users/you/Projects/acme-shop": {
      "project": "acme",
      "stack": "shop",
      "modes": {
        "test": {
          "host": "shop.acme.test.localhost",
          "compose_project": "acme-shop-test",
          "ports": { "web": 3010, "api": 8010 }
        }
      }
    }
  },
  "locks": {
    "acme:shop:test": {
      "checkout": "C:/Users/you/Projects/acme-shop",
      "compose_project": "acme-shop-test"
    }
  }
}
```

## Keys

| Key | Meaning |
|-----|---------|
| `checkouts` | Map of normalized repo root → allocations |
| `checkouts.*.project` | From `.siding/config.yml` `project` |
| `checkouts.*.stack` | Resolved stack name at last write |
| `checkouts.*.modes.*.ports` | Allocated host ports (global uniqueness enforced at allocation time) |
| `checkouts.*.modes.*.git_branch` / `git_sha` / `last_up_at` | HEAD at last successful `siding up` (board guess) |
| `checkouts.*.modes.*.entry_points` | Clickable board hosts per HTTP port key (`web: [app]`, `api: [api]`) |
| `locks` | `{project}:{stack}:{mode}` → owning checkout |

## Resolution

When you run `siding plan`, `siding sync`, `siding up`, or `siding remove`:

1. Start path = `--path` or cwd
2. Walk up to `.siding/config.yml` → repo root
3. Checkout key = `NormalizeLockPath(repo_root)`
4. Stack = `--stack` → folder basename minus `folder_prefix` → basename
5. Ports = `state.checkouts[key].modes[mode].ports` if present, else contract mode defaults
6. Host / compose project = expanded patterns, unless state overrides them

## Port allocation

`siding add` / `siding sync` (and the first `siding up` for a checkout) treat host binds as machine-global:

1. If this checkout already has ports for the mode, keep them (idempotent) and **merge** any new keys from the org recipe.
2. New claims **do not start** on conventional app/db ports. Compose defaults below 10000 (and a few well-known ones like `27017`) are lifted by **15000** first (`3000` → `18000`, `5432` → `20432`, `8000` → `23000`). Ports already in the siding band (hello's `18080`) stay put.
3. Busy set = every checkout in `state.json` + reserved proxy ports (`80`, `2019`) + **currently published Docker host ports** (`docker inspect` on running containers). `network_mode: host` containers cannot list binds; doctor warns.
4. Each **new key** is claimed on its own. If that number is taken, it shifts by 10 (max 100 tries). Duplicate recipe keys (`gcs-emulator` / `gcs_emulator`) collapse to one. Two services that share a compose default get distinct host ports.
5. Independent compose files are **separate modes**. `siding sync` (no `--mode`) allocates all of them. Mode `test` writes `.siding/env`; `prod-db` writes `.siding/env.prod-db`. `docker compose -p` uses `{project}-{stack}-{mode}` so a compose `name:` field cannot collide.

`down` releases the lock but keeps the allocation so the next `up` reuses the same ports. `siding remove` downs the stack, then deletes the checkout row (and its locks) so a reclone can `init` onto a blank allocation.

## Shared proxy

`siding proxy up` writes `{user-data}/siding-proxy/` (Caddyfile + compose) and starts **siding-proxy-caddy** on loopback:

| Bind | Role |
|------|------|
| `127.0.0.1:80` | HTTP Host routes (override with `siding proxy up --listen`) |
| `127.0.0.1:2019` | Caddy admin API (`siding proxy reload` POSTs a new Caddyfile here) |

Those two ports are reserved and never allocated to a stack. A leftover `proxy.listen` of `0` or `1355` is migrated to `80` on read. Explicit `--listen 8080` is kept.

`http://siding.localhost` is a status board: every allocated checkout, whether the slot lock is held (`up`), clickable entry-point links (`app`, `api`, …), and git branch/SHA recorded at the last successful `siding up`. Stack Hosts are matched separately and never redirect to the board. An unknown Host gets 404. The UI is the published `ghcr.io/theblackhowling/siding-proxy:latest` image (Caddy + React SPA). Data is `{user-data}/siding-proxy/board/status.json`, rewritten whenever proxy files are written. `siding proxy up` pulls `:latest` so the board UI can move without a new CLI.

HTTP port keys only:

| Port key | Sites |
|----------|--------|
| `web` | `{host}` (plus extra `http_hosts.web` labels) |
| `api` | Sibling when the host starts with a frontend role (`app.acme…` → `api.acme…`); otherwise `api.{host}` |

`redis` / `postgres` stay as Docker host binds in `ports` / `.siding/env`. They do not get Caddy sites (TCP has no `Host` header).

Upstream is `host.docker.internal:{allocated}`. Public URLs are portless on listen 80. `siding up` / `down` / `sync` reload via the admin API; the proxy container is force-recreated only when its compose publish ports change.

## Relocation

If a repo moves on disk, the checkout key changes. Future: `siding doctor --fix-path` to migrate entries (not implemented yet).
