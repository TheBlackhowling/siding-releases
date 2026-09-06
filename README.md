# Siding

**Park a stack on a siding.** Named local Docker stacks — isolated ports, compose project names, and `*.localhost` hosts — without copying allocator scripts into every app.

Not vinyl. A railroad siding: the main line stays clear; each checkout waits on its own spur.

Source is private. This repo holds **public binaries** and user docs.

- [Releases](https://github.com/TheBlackhowling/siding-releases/releases/latest) — zip / tar.gz + checksums
- [Quick start](docs/QUICKSTART.md) — first recipe, what to commit, first `up`
- [Commands](docs/COMMANDS.md) — every command, flag, and what it does
- [Repo recipe](docs/CONTRACT.md) — committed `.siding/config.yml`
- [Machine state](docs/STATE.md) — `state.json` and the shared proxy

## Install

Download the archive for your OS from
[Releases](https://github.com/TheBlackhowling/siding-releases/releases/latest).

**Windows** (amd64 zip):

```powershell
.\siding.exe install
$env:Path = "$env:LOCALAPPDATA\siding\bin;" + $env:Path
siding version
```

New Windows Terminal / cmd windows pick up PATH. Cursor and other IDE terminals keep the IDE’s environment until the IDE restarts.

**macOS / Linux:** unpack `siding` onto your PATH (`~/.local/bin` is typical). macOS builds are unsigned.

`checksums.txt` is on each Release. `siding version` prints the tag, commit, and build date.

## What it is designed to do

Siding is for people who keep **several checkouts of the same product** (or several products) running at once. Docker Compose defaults (`3000`, `8000`, `5432`, `8080`) collide the moment two folders try to bind the same host ports. Per-repo PowerShell allocators drift. Hosts-file hacks do not scale.

Siding splits that problem in two:

| Layer | Where | What |
|-------|--------|------|
| **Org recipe** | committed `.siding/config.yml` | Kinds of stack: modes, compose files, **every published port key**, public URL templates |
| **This machine** | `%LOCALAPPDATA%\siding\state.json` | This checkout’s allocated ports, host, compose project |

The recipe does not name slots. The **folder** is the slot: `acme-shop` becomes stack `shop` (optional `folder_prefix`). A new clone needs no recipe edit.

Compose always runs **on the host**. The shared proxy (Caddy on host **`:80`**) is optional and machine-wide: browsers use `http://app.shop.acme.test.localhost` while Redis and Postgres stay unique Docker host binds so two checkouts still cannot sit on `5432`.

## Intentions

- **Own every published port.** Web, API, Redis, Postgres, Firebase, Tika — if compose publishes it, siding allocates it. Caddy only HTTP-proxies `web` and `api`; the rest still move so they cannot collide.
- **One recipe per product, many checkouts.** Commit `.siding/config.yml`. Do not commit `.siding/env` or machine `state.json`.
- **Identity from the path.** `--path` or cwd → walk up to `.siding/` → folder basename (minus prefix) → stack. No roster of named stacks in the recipe, no overlay beside the exe.
- **In-stack DNS vs browser URLs are different on purpose.** Containers talk `http://backend:8000` on the Compose network. Browsers talk `*.localhost` via Caddy on `:80`.
- **Never edit the OS hosts file.** Browsers resolve `*.localhost` without it. `--print-hosts` only prints lines for curl/Node/Windows if you want them.
- **Host binary for compose.** `up` / `down` / `reset` / `tail` / `remove` / `ps` / `proxy` must run as a native exe so bind mounts are your Windows (or macOS/Linux) paths, not a container’s `/src`.

Siding is not a cloud orchestrator, not a replacement for Docker Compose, and not a TCP/hostname router for Postgres. It is a local slot allocator plus an optional HTTP front door.

## License

MIT — see [LICENSE](LICENSE).
