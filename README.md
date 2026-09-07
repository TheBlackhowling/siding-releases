# Siding

Run several local Docker Compose stacks at once: different products, or
several clones or git worktrees of the same product, without colliding
ports and without remembering which URL belongs to which folder.

Each folder is a slot. A second clone and a worktree of the same repo both
work: each gets its own ports, compose project name, and `*.localhost` host.
The recipe lives in git; the numbers stay on this machine.

Inside a slot, the stack’s services talk to each other on the Compose
network (`http://backend:8000`, `postgres:5432`). Those internal ports do
not need rewriting. Siding only moves what is published to the host: the
endpoints a person actually hits.

Siding also serves a local UI at `http://siding.localhost`: the available
HTTP endpoints for each slot, the git branch and SHA recorded at last
`siding up`, and the compose project name.

Park a stack on a siding: the main line stays clear; each checkout waits on
its own spur.

Source is private. This repo holds **public binaries** and user docs.

- [Releases](https://github.com/TheBlackhowling/siding-releases/releases/latest): zip / tar.gz + checksums
- [Quick start](docs/QUICKSTART.md): first recipe, what to commit, first `up`
- [Commands](docs/COMMANDS.md): every command, flag, and what it does
- [Repo recipe](docs/CONTRACT.md): committed `.siding/config.yml`
- [Machine state](docs/STATE.md): `state.json` and the shared proxy

## Why

Compose defaults (`3000`, `8000`, `5432`, `8080`) assume one stack per
machine. Two folders up at the same time fight over those binds. Per-repo
allocator scripts drift. Hosts-file hacks do not scale.

The worse cost is keeping the map in your head: which endpoint is this
branch, which is the other product, which database is safe to wipe.

Siding is a local slot allocator plus an optional HTTP front door. One
committed recipe describes the product. This machine assigns each folder a
slot. Browsers use `*.localhost` names instead of raw ports. The board at
`http://siding.localhost` is the map: endpoints, last-up git, compose
project, and whether the slot is up.

It is not a cloud orchestrator, not a replacement for Docker Compose, and
not a TCP router for Postgres.

## Install

Download the archive for your OS from
[Releases](https://github.com/TheBlackhowling/siding-releases/releases/latest).

**Windows** (amd64 zip):

```powershell
.\siding.exe install
$env:Path = "$env:LOCALAPPDATA\siding\bin;" + $env:Path
siding version
```

New Windows Terminal / cmd windows pick up PATH. Cursor and other IDE
terminals keep the IDE’s environment until the IDE restarts.

**macOS / Linux:** unpack `siding` onto your PATH (`~/.local/bin` is typical).

`checksums.txt` is on each Release. `siding version` prints the tag, commit,
and build date.

## How it is split

| Layer | Where | What |
|-------|--------|------|
| **Org recipe** | committed `.siding/config.yml` | Kinds of stack: modes, compose files, **every published port key**, public URL templates |
| **This machine** | `%LOCALAPPDATA%\siding\state.json` | This checkout’s allocated ports, host, compose project |

The recipe does not name slots. The **folder** is the slot. The stack name
is the folder basename unless the recipe sets `folder_prefix`; then that
prefix is trimmed (`acme-shop` with `folder_prefix: acme-` becomes stack
`shop`; without a prefix, folder `acme-shop` is stack `acme-shop`). A new
clone needs no recipe edit. A second product is just another recipe in
another folder.

Compose always runs **on the host**. The shared Caddy proxy (host **`:80`**)
is optional and machine-wide. With it, browsers use portless names like
`http://app.shop.acme.test.localhost`. Without it, you must remember which
allocated port belongs to which name. That is a lot of cognitive load. Use
Caddy (`siding proxy up`). Redis and Postgres still get unique Docker host
binds because two checkouts cannot sit on `5432`. We are looking into
easier access for Redis and other database instances.

## Rules

- **Own every published port.** Web, API, Redis, Postgres, Firebase, Tika:
  if compose publishes it, siding allocates it. Caddy only HTTP-proxies
  `web` and `api`; the rest still move because they cannot collide.
- **One recipe per product, many checkouts.** Commit `.siding/config.yml`.
  Do not commit `.siding/env` or machine `state.json`.
- **Identity from the path.** `--path` or cwd → walk up to `.siding/` →
  folder basename, minus `folder_prefix` when that is set → stack. No
  roster of named stacks in the recipe.
- **Use the Caddy proxy.** Without it, every slot is a name plus a port you
  have to remember. With it, URLs are portless `*.localhost` names and the
  board lists them.
- **In-stack traffic stays on the Compose network.** Services in a slot
  reach each other by service name and container port. Nothing that is not
  exposed to the user needs a rewritten host port. Browsers talk
  `*.localhost` via Caddy on `:80`.
- **`*.localhost` on purpose.** We use `*.localhost` so siding does not
  rewrite the OS hosts file. Browsers resolve those names without it. You
  can still write the hosts file yourself if a tool needs it.
  `--print-hosts` prints the lines.

## License

MIT: see [LICENSE](LICENSE).
