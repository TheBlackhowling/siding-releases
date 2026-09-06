# .siding/config.yml contract (draft)

Each target repo ships a **recipe** at `.siding/config.yml`. It describes kinds of stack (modes, compose files, port keys) — not a roster of named slots.

Slot identity comes from the checkout folder (or `--stack`). Allocated ports live in machine `state.json`.

`siding init` writes this file. Commit it. Generated slot env is `.siding/env` (gitignored).

See [README](../README.md), [Quick start](QUICKSTART.md) (what to check in), and [Commands](COMMANDS.md).

## Minimal example

```yaml
version: 1
project: hello
host: "{stack}.{project}.{mode}"
compose_project: "{project}-{stack}-{mode}"
slot_env_file: .siding/env

modes:
  test:
    compose_files:
      - docker-compose.yml
    ports:
      web: 18080
    port_env:
      web: WEB_PORT
```

Folder `hello` → stack `hello` → host `hello.test.localhost` (duplicate `{stack}`/`{project}` segments are dropped) → compose project `hello-test`.

## Multi-clone product

```yaml
version: 1
project: acme
folder_prefix: acme-
host: "{stack}.{project}.{mode}"
compose_project: "{project}-{stack}-{mode}"

modes:
  test:
    compose_files:
      - docker-compose.yml
    ports:
      web: 3000
      api: 8000
  prod:
    compose_files:
      - docker-compose.yml
      - docker-compose.prod-db.yml
    ports:
      web: 3100
      api: 8100
```

Folder `acme-shop` → stack `shop` → `shop.acme.test.localhost` / `acme-shop-test` when there is no `host_prefix`. With `host_prefix: example` the same folder is `app.example.shop.test.localhost` (prod: `app.example.shop.prod.localhost`). `www.` and `api.` on that root work too.

## Fields

| Field | Required | Meaning |
|-------|----------|---------|
| `project` | yes | Product id shared by every clone (locks, `{project}` in compose). Init defaults this to the folder-prefix stem (`acme-` → `acme`), not the clone folder (`acme-web2`) |
| `modes.<name>` | yes | A kind of stack (`test`, `prod`, …) |
| `modes.<name>.compose_files` | yes | Passed to `docker compose -f` (one instance per mode). After `--rewrite-compose`, these are `.siding/compose/<basename>` copies; originals at the repo root stay for non-siding users |
| `modes.<name>.ports` | no | Default first-claim ports; `state.json` overrides after `add`/`up` |
| `modes.<name>.port_env` | no | Maps port keys to env var names |
| `modes.<name>.env` | no | Static env merged into that mode’s env file |
| `host_prefix` | no | Product name only (`example`), not `app.example`. Host pattern `{prefix}.{stack}.{mode}` → root `example.shop.test.localhost`; `app`/`www`/`api` are subdomains. Init asks (or `--http-prefix` / `--host-prefix`). Compose can guess `example` from `app.example.test` |
| `http_hosts` / `modes.<name>.http_hosts` | no | Extra Caddy labels per port key. Not used for the public prefix — that is `host_prefix` |
| `entry_points` / `modes.<name>.entry_points` | no | Clickable board hosts per HTTP port key (`web: [app]`, `api: [api]`). Init asks (or `--entry web=app`). Defaults: `app` for web when `host_prefix` is set, else the apex; `api` for the API |
| `host` | no | Pattern, default `{stack}.{project}.{mode}`, or `{prefix}.{stack}.{mode}` when `host_prefix` is set |
| `compose_project` | no | Pattern, default `{project}-{stack}-{mode}` |
| `folder_prefix` | no | Stripped from the folder basename (`acme-shop` → `shop`) |
| `slot_env_file` | no | Default `.siding/env` for mode `test`. Other modes use `.siding/env.<mode>` |

Tokens in patterns: `{stack}`, `{project}`, `{mode}`, `{prefix}`. Adjacent duplicate segments are collapsed (`hello.hello.test` → `hello.test`). `.localhost` is appended to hosts if missing. `*.localhost` names are not added to the OS hosts file.

A mode may override `host` / `compose_project` with its own pattern.

## Stack resolution

1. `--stack` (any name; sanitized)
2. folder basename minus `folder_prefix`
3. folder basename

There is no roster of slot names and no implicit `main`.

## Init wizard

`siding init` writes the org recipe. If `.siding/config.yml` is already committed, **init runs `siding sync`** instead of overwriting it (`--force` re-runs the wizard).

`siding sync` maps that recipe onto **this machine**:

- allocate or merge this checkout in `state.json` (existing port numbers stay; new recipe keys are claimed)
- refresh `host` / `compose_project` from current patterns
- write `.siding/env` with `{host}{proxy_suffix}` expanded for this slot
- reload the shared proxy if it is already running

It does not start containers and does not copy or adapt compose.

`siding init` asks (or flags: `--prefer-proxy`, `--rewrite-compose`, `--http-prefix`, `--entry`, `--compose`, `--mode-name`):

1. **Which root compose files** — lists `compose.yaml` / `docker-compose.yml` / `docker-compose.*.yml` at the repo root. Type `all` or comma-separated numbers (`1,3`). **Each file is its own stack** (own ports, compose project, env file) so they can run at the same time. `docker-compose.override.yml` attaches to the primary base. Non-interactive uses every file found unless you pass `--compose`.
2. **Suffix per stack** (`test`, `prod`, …) — used in hosts (`{stack}.{project}.{suffix}`), `siding up --mode`, and `.siding/env` vs `.siding/env.prod`. Guess from the filename; type `prod` if you want that instead of `prod-db`. Flag: `--mode-name docker-compose.prod-db.yml=prod`.
3. **Caddy proxy vs raw ports** — recommended: `siding proxy up`. Caddy listens on host **`:80`**. `NEXT_PUBLIC_*` is portless. On a TTY, answering no to “stop init” continues without the proxy; non-interactive init errors if something else holds `:80`.
4. **Siding compose copies?** — write `.siding/compose/<same-basename>` from each chosen file. **The original is not modified** (`docker compose -f docker-compose.yml` still works for people who do not use siding). The copy gets:
   - `"3000:3000"` → `"${WEB_PORT:-3000}:3000"`
   - public URL defaults with `:port` → `${NEXT_PUBLIC_…}` / `${DEV_…}` filled from `.siding/env`
   - in-container `localhost` → Compose **service DNS** (`http://backend:8000`)
   - `network_mode: host` or disjoint service networks → one shared project network
   - healthchecks, GCS/emulator `localhost`, and host-facing `DEV_API_URL` are left as public/host values (not service DNS)

   `siding up` runs `--project-directory` at the repo root so build contexts and volumes in the copy still resolve like the original.

   App code that reads `NEXT_PUBLIC_*` from the environment may still need a **one-line wrap on the original** (`${NEXT_PUBLIC_ROOT_DOMAIN:-app.acme.test:3000}`) so non-siding defaults keep working. That is not a full rewrite.
5. **Public host prefix** — product name only (`example`). Do not type `app.`. With prefix `example`: `app.example.{stack}.test.localhost`, `www.example.{stack}.test.localhost`, `api.example.{stack}.test.localhost`. `none` keeps `{stack}.{project}.{mode}`. Do **not** put `*.localhost` in the hosts file.
6. **HTTP entry points** — first label for each web container (`web` → `app`, `api` → `api`). Written as `entry_points` and shown as clickable links on `http://siding.localhost`. Flag: `--entry web=app --entry api=api`.

Recipe env templates are expanded on every `plan`/`up`:

| Token | Meaning |
|-------|---------|
| `{host}` | Canonical `*.localhost` host |
| `{proxy_suffix}` / `{web_suffix}` | Empty when the proxy listen is 80; else `:{listen}`. Proxy off → `:{web}` |
| `{api_suffix}` | Same as `{web_suffix}` when the proxy is up; proxy off → `:{api}` (not the web port) |
| `{url}` | `http://{host}{proxy_suffix}` |
| `{web_host}` | Public web host (`{host}`, or first extra `http_hosts.web` label) |
| `{api_host}` | Sibling when the first label is a frontend role (`app.acme.shop.test` → `api.acme.shop.test`); otherwise `api.{host}` |
| `{stack}` `{project}` `{mode}` `{web}` `{api}` | slot + port keys |

```yaml
env:
  NEXT_PUBLIC_ROOT_DOMAIN: "{host}{proxy_suffix}"
```

## Compose ports and public URLs

`siding init`, `siding doctor`, and `siding plan` read compose files and print tips.

**Host ports** must be interpolations so parallel checkouts can move:

```yaml
ports:
  - "${WEB_PORT:-3000}:3000"
```

Not `"3000:3000"`. Map the env name in `port_env` (`web: WEB_PORT`).

**Own every published port.** Redis, Postgres, Firebase, Tika, … must be in `modes.*.ports` so each checkout gets a unique host bind. Leaving them at compose defaults (`5432`, `8080`) is how parallel stacks collide. A service with several publishes (Firebase emulators) gets one key per bind (`firebase_8080`, …). Caddy only HTTP-proxies `web` and `api`; the other keys are still allocated and written to `.siding/env`. `init` / `sync` take the full scan — `--port` overrides a key, it does not drop the rest. Doctor warns `unowned_port` when compose publishes something the committed recipe omitted.

**`NEXT_PUBLIC_*` / `VITE_*`** are baked into the browser. Caddy listens on host **`:80`** by default:

| Proxy | Public domain value |
|-------|---------------------|
| `siding proxy up` (listen 80) | `shop.acme.test.localhost` |
| Proxy off | `shop.acme.test.localhost:18000` (allocated web) |

If compose defaults are `${NEXT_PUBLIC_ROOT_DOMAIN:-host:${WEB_PORT:-3000}}`, the **slot file must override** them. Siding writes `SIDING_HOST` / `SIDING_URL` into `.siding/env`. `init` also writes discovered public vars under `modes.*.env` as `{host}{proxy_suffix}` templates so redirects and cookies hit Caddy instead of the raw container port.

**In-stack vs browser:** containers talk on the Compose network (`http://backend:8000`). Browsers talk to `*.localhost` via Caddy on `:80`. Those are different names on purpose.

See `docs/STATE.md` for the machine `state.json` schema.
