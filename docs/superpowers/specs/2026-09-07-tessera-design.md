# Tessera — Design Spec

**Date:** 2026-09-07
**Status:** Approved for planning
**Repos:** `stevenjc2009-byte/tessera` (client), `stevenjc2009-byte/tessera-server` (server) — both public

---

## 1. What it is

A Nintendo 3DS homebrew app for making pixel art in 2D and building coloured 3D
models out of hand-drawn polygon faces, with a public gallery where anyone
holding the CIA can upload, browse, download and rate creations.

The name: a *tessera* is one of the small coloured tiles in a mosaic. Pixel
tiles and flat faces joined into a whole is the app.

### The unifying idea

Every face of a model is a small pixel canvas. The 2D pixel editor **is** the
3D editor's face editor — same tools, same palette, same undo, same code. The
two halves of the app are one engine with two framings, not two products
sharing a menu.

### Goals

- A pixel art editor good enough to use on its own, with real pixel-art
  discipline (locked palettes, visible grid, one pixel per tap, no
  anti-aliasing ever).
- A polygon face editor: place points on a grid, close them into a face of any
  shape, colour it, join faces into a 3D model.
- Save a picture to the device, and export a real 3D model.
- A public gallery reachable from the console, with upload, browse, download,
  rating and reporting.
- A server that is a dead end if compromised — no path from it to the home
  network.

### Non-goals for v1

Animation frames and onion skin. Layers beyond one. Blend modes. Tile/wrap
mode. Accounts or passwords. Anything requiring the user to configure a
network.

---

## 2. Platform constraints

These are measured facts about the target hardware, and they shape everything
below.

| Constraint | Consequence |
|---|---|
| Top screen 400x240, bottom 320x240 touch | Top = preview, bottom = all interaction |
| Single-touch resistive digitizer, no multi-touch | No pinch-zoom, no gestures. Buttons for zoom/pan. Needs jitter smoothing |
| ~20MB heap on Old 3DS; 6MB VRAM shared with framebuffers | Hard ceilings on canvas size, undo depth, texture budget |
| Textures must be power-of-two, 8..1024px, stored tiled (8x8 Morton order) | Painting cannot poke pixels into a texture directly — needs swizzle-on-write or render-to-texture |
| `ssl:C` tops out at TLS 1.1 | The console cannot speak modern HTTPS. See §6 |
| 3DSX inherits Homebrew Launcher's fixed memory mode; CIA declares its own | Ship as CIA |

---

## 3. The app

### 3.1 Screen layout

- **Top screen** — 3D model preview, or a zoomed view of the 2D canvas.
  Rotates with the circle pad, zooms with L/R. Never touched, because it
  cannot be. Stereoscopic 3D is enabled for the model view; it is the one
  place the 3DS gimmick genuinely helps, because depth ambiguity is the
  hardest problem in small-screen 3D editing.
- **Bottom screen** — always a flat, axis-locked 2D surface. Either the pixel
  canvas, the face being painted, or the layout grid. One touch point, no
  ambiguity about what was meant.

This split is not a preference. Every touch voxel tool that made the user
rotate a 3D view by dragging on the touch surface was criticised for it, and
*VoxelMaker* (official 3DS eShop) already proved this exact layout works on
this exact hardware.

### 3.2 Mode A — flat pixel art (first-class)

The primary mode. A bounded pixel canvas, painted with the stylus.

**Tools.** Pencil, eraser, bucket fill, colour picker, undo — universal across
every pixel editor surveyed, never dropped even by the most minimal mobile
ones. Then line, rectangle, ellipse — near-universal second tier, and
essential because drawing straight lines pixel-by-pixel with a stylus is
miserable.

**Palettes.** Bundled constrained palettes: PICO-8, DawnBringer 16, DawnBringer
32, Game Boy 4-shade, NES. Working inside 16 colours is most of what makes
pixel art read as pixel art, and indexed pixels cost 1 byte instead of 4.
Custom palettes can be edited and saved to SD as `.gpl` or `.hex` — both plain
text, both trivial to parse.

Palettes ship **offline, in the app**. They are not fetched from Lospec: the
console cannot make a modern HTTPS connection (§2).

**Canvas sizes.** 16x16, 32x32, 64x64, 128x128. Indexed colour throughout.

**Export.** PNG to `sdmc:/tessera/`. Via lodepng vendored into the tree — one
`.c`/`.h` pair, zero dependencies. `3ds-libpng` exists as an official portlib
but brings zlib and is not needed here.

### 3.3 Mode B — polygon face editor

**Creating a face.** Tap points on the bottom screen; each tap drops a vertex,
tapping the first point closes the shape. Handles triangles, hexagons and
completely irregular shapes with one interaction.

Points snap to a square lattice (fine / coarse / off) so edges meet cleanly
and faces align when joined.

**Preset stamps.** One tap drops a ready-made triangle, square, pentagon,
hexagon or n-gon. Most faces people want are regular; making them by hand
every time gets old.

**Editing.** Drag a point to move it. Tap an edge to insert a point.
Long-press a point to delete it. Faces are never finished — a shape can be
tweaked after it is painted and joined.

**Under the hood.** The GPU draws only triangles, so polygons are triangulated
internally by ear clipping. The user never sees this; it means concave and
irregular shapes render correctly.

**Readout.** Side lengths and vertex angles shown while dragging. Cheap to
compute, and it is the difference between "roughly a hexagon" and a hexagon.

### 3.4 Mode C — painting a face

Once a face exists, the bottom screen becomes a pixel grid clipped to that
polygon's outline. Everything outside the shape is dead space that cannot be
painted, so the pixel art is bounded by the geometry.

Same toolset, same palette, same undo as Mode A. Two fill styles per face:

- **Flat** — the whole face is one palette colour. Costs almost nothing and is
  what most of a low-poly model wants.
- **Textured** — the face carries a painted pixel grid at its own resolution
  (8x8, 16x16, 32x32). Per-face resolution keeps memory sane instead of paying
  full price for every surface.

**Symmetry painting.** A stroke on one face mirrors onto the corresponding
face across the model's axis. Cut by most mobile apps — Dotpict charges for it
— but for 3D models it is a genuine time-saver and it is cheap to implement.

### 3.5 Mode D — joining faces into a model

- **Edge-to-edge snapping.** Drag a face near another; matching-length edges
  snap and hinge. This is the core join interaction and it must feel magnetic.
- **Hinge angle.** After joining, set the fold angle between the two faces —
  90 degrees for boxes, arbitrary for anything organic. Common angles snap.
- **Shared vertices, not duplicated.** Joined faces share their edge points, so
  moving a vertex moves it on every attached face. Prevents cracks.
- **Extrude an edge.** Select an edge, pull outward, get a new connected face
  already sharing that edge. Much faster than drawing and positioning each
  face from scratch.
- **Net view.** Show the model unfolded flat, papercraft-style. Doubles as a
  way to see the whole texture layout at once, and as the way to pick a face
  for painting — because the top screen is not touch-capable.
- **Open-edge highlighting.** Flag edges joined to nothing, so holes are
  obvious before export.
- **Face groups.** Named parts (head, arm, wheel) that hide, move and duplicate
  together.

### 3.6 Mode E — colouring book

Pre-made line art shipped as bitmaps with closed regions. Tap a region, flood
fill underneath the black outline layer so lines can never be overpainted.
Standard 4-connected BFS fill — the same fill code the bucket tool already
needs, so this mode is nearly free.

Gaps in line art leak, so bundled artwork must be authored closed.

### 3.7 Undo

Budget is ~20MB total. Undo must never take whole-canvas snapshots.

- **Command objects** for strokes — store only the changed-pixel list.
- **Dirty-rectangle diffs** for fills and pastes — the bounding box of what
  changed, not the frame.
- **Explicit depth cap**, with older entries RLE-compressed. Aseprite is the
  cautionary tale here: it stored full command objects and had to retrofit
  compression when histories grew.

### 3.8 Export

| What | Format | Notes |
|---|---|---|
| 2D pixel art | PNG | To `sdmc:/tessera/` |
| Model render | PNG | Hero shot or turntable frames |
| 3D model | OBJ + MTL + texture PNG | Plain text, written with `fprintf`, opens in Blender/Blockbench/engines |
| Project | Tessera native (JSON) | Points, faces, joins, palettes, textures. OBJ loses editing structure |

`.vox` is explicitly **not** a target: it stores per-voxel colour, which throws
away the per-face painting that is the entire point.

---

## 4. The server

### 4.1 Shape

One small HTTP daemon in an unprivileged Proxmox LXC. Files on disk, SQLite
index. Reached from the console through a playit.gg outbound tunnel — no port
forward, no home IP exposed.

Deliberately **not** Blocksmith's architecture. Blocksmith is a two-daemon
authoritative game server with a Noise-XX trust split because it is a game
facing the internet. A gallery is uploads, thumbnails and browsing. What is
copied from Blocksmith is the *deployment and hardening pattern*, not the
protocol.

### 4.2 API

All state-changing requests are signed (§6). Endpoints:

| Endpoint | Purpose |
|---|---|
| `POST /upload` | Model or pixel art + client-rendered thumbnail + metadata |
| `GET /browse` | Paged list: id, title, author key, thumbnail, score, date |
| `GET /model/{id}` | Download the project file |
| `GET /thumb/{id}` | Thumbnail bytes |
| `POST /vote` | Thumbs up / thumbs down, one per key per model |
| `POST /favourite` | Personal favourite (heart), unlimited, private to the key |
| `POST /report` | Reason + optional free text |
| `GET /admin/reports` | Admin key only — moderation queue |
| `POST /admin/action` | Admin key only — approve / delete / ban |

**Hearts and thumbs are separate systems and are named as such.** A heart is
"save to my favourites" — personal, private, unlimited. Thumbs are "rate this"
— public, one per person, aggregating into a score. Three buttons that all
feel the same would be worse than two that mean different things.

### 4.3 Storage

- Uploads stored **by content hash**, never by user-supplied filename. Kills
  path traversal outright.
- Files live in a directory the daemon can write and nothing can execute.
- SQLite holds the index: id, hash, title, author key, timestamps, votes,
  report count, visibility state.

### 4.4 Moderation

- **Fixed reason list:** gore, sexual, hate, stolen, spam, other. Selecting
  "other" opens a text box with a submit button. Free text on every report
  would itself be an abuse vector.
- **Reporter's key is stored**, so one person cannot report the same piece
  fifty times.
- **Auto-hide on threshold.** N distinct reports auto-unlists a model pending
  review. This is the highest-value piece of the whole moderation design: it
  drops the damage window from "until steve next looks" to minutes, and costs
  almost nothing.
- **Pull-based delivery, zero egress.** Reports sit in SQLite. Nothing is sent
  anywhere. They are fetched either via `pct enter` and a `tessera-reports`
  CLI, or through the admin endpoint from the app. The server never initiates
  an outbound connection, so the egress lockdown stays sealed.
- **Admin view in the app.** The admin key sees a queue with the reported art,
  the reason, and approve / delete / ban buttons.

Rejected: Discord webhook, ntfy, email. All require punching a hole in the
outbound firewall, and a hole outward is exactly what an attacker uses to
exfiltrate. Reconsiderable later as a separate low-privilege sender process
with egress limited to one host.

### 4.5 Abuse control without authentication

Anyone with the CIA can upload. There is no password. Abuse control therefore
cannot come from authentication:

- **Per-console keypair**, generated on first run. Not a gate — an identity.
  It attributes uploads, enforces one vote per person, and gives a ban handle.
  Blocksmith already does this (`client.pub` / `client.seed`), so it is proven
  on this hardware.
- **8 uploads per key per day.**
- **Per-IP rate limiting**, using `proxy-protocol-v2` on the playit tunnel so
  the real uploader IP survives the relay.
- **Global upload rate cap.**
- **Hard caps** on upload size and total disk. A public endpoint with no quota
  fills the disk quickly once anyone notices.
- **Ban by key**, accepting that a reinstall yields a new key. IP limiting does
  the real work against a determined abuser; key bans handle the casual case.

---

## 5. Security model

The requirement: if someone gets into the server, they reach a dead end and
cannot touch the home network.

Carried over from Blocksmith, proven on the node:

1. **Unprivileged LXC**, `nesting=0`, no device passthrough. Root inside maps
   to an unprivileged uid on the host.
2. **Zero inbound ports.** playit.gg dials out. Nothing is forwarded on the
   router; there is no listening socket reachable from the internet.
3. **nftables default-drop on input *and* output.** Only loopback, established,
   DNS, apt, and the playit uid. An attacker with code execution cannot call
   home.
4. **Systemd sandbox** — empty `CapabilityBoundingSet`, `NoNewPrivileges`,
   `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`,
   `MemoryDenyWriteExecute`, `RestrictNamespaces`, `ProtectProc=invisible`,
   `SystemCallFilter=@system-service` minus `@privileged @resources @obsolete
   @mount @debug @cpu-emulation @swap`, `MemoryMax`, `TasksMax`,
   `LimitNOFILE`.

Tightened beyond Blocksmith, because this service accepts public uploads:

5. **No `meta skuid root accept` in the output chain.** Blocksmith has one,
   which means root inside that container can reach the LAN. Tessera drops it
   and adds explicit RFC1918 destination drops (10/8, 172.16/12, 192.168/16).
   This is the difference between "hard to pivot" and "cannot pivot".
6. **The server never decodes uploaded images.** Running a memory-unsafe image
   parser on hostile input is by a wide margin the most likely way in. The
   console renders and uploads its own thumbnail; the server stores bytes and
   never interprets them.
7. **Content-hash filenames** (§4.3).

Optional, steve's call: put the container on its own VLAN or bridge with no
route to the LAN, so isolation is enforced by the network as well as the
firewall. The RFC1918 drops get most of the way there without touching network
config.

---

## 6. Transport: signed plain HTTP

The console cannot speak modern HTTPS — `ssl:C` tops out at TLS 1.1 and modern
servers refuse it.

**Decision: plain HTTP, with every state-changing request signed by the
console's keypair using libhydrogen.**

Reasoning:

- **The content is public.** Encryption protects secrets; a public gallery has
  none.
- **Anyone can upload legitimately anyway.** With no password, a MITM forging
  an upload achieves nothing they could not do by opening the app.
- **What actually matters is forgery of identity and votes**, and the right
  tool for that is signatures, not TLS.
- libhydrogen is already vendored and pinned (commit
  `617036a353cd4f6478ab6c3f98c36dd31e23ce8e`) and running on both sides of
  Blocksmith. Replay defence already exists there as `replay.c`.

Accepted cost: an observer on the path can see who downloaded what. For a
public art gallery that is near-zero.

Rejected: bundling mbedtls. It works on 3DS but costs a CA bundle against a
20MB heap, brings the hardware-entropy trap (no `sslcInit()` means silently
predictable keys), and buys almost nothing here.

Requests carry a nonce and timestamp, checked against a sliding replay window.
Admin actions are signed with the same mechanism; the admin key is simply the
one flagged as such.

---

## 7. Deployment

### 7.1 Two-script install, Blocksmith pattern

**`install/proxmox-install.sh`** — runs on the Proxmox host as root.
Autodetects storage via `pvesm status --content rootdir`, finds or downloads a
Debian template with `pveam`, creates an unprivileged container with `pct
create`, pushes the source in, hands off to the provision script.

Flags mirror Blocksmith's: `--ip`, `--gw`, `--nameserver`, `--ctid`,
`--hostname`, `--storage`, `--bridge`, `--disk`, `--memory`, `--cores`,
`--port`, `--no-start`. Env-var overrides for non-interactive use.

Carried over from Blocksmith: after `pct start`, poll for systemd readiness,
then verify DNS with `getent hosts deb.debian.org` **before touching apt** —
because `apt-get update` exits 0 even when the mirror is completely
unreachable.

**`install/container-provision.sh`** — runs inside, idempotent. Installs
packages, creates the service user, builds, **runs the test suite and a
hardening check and refuses to install if either fails**, writes the nftables
ruleset, installs the systemd unit, starts the service.

### 7.2 One-line invocation

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/stevenjc2009-byte/tessera-server/v1.0.0/install/proxmox-install.sh)"
```

Pinned to a **tag**, not `main`. This pipes a remote script into root on the
hypervisor; pinning is the one mitigation that actually matters.

Not built on `community-scripts/core`. That is a 7,000-line engine fetched
from a third-party repo at runtime, and a homelab app should not depend on
someone else's repo staying up to install. Blocksmith's standalone approach is
already proven on this node.

### 7.3 Manual steps that cannot be scripted

Both apply identically to Blocksmith and are documented there:

1. **`playit setup`** needs interactive browser approval. Writing a secret to
   `playit.toml` does not work — the daemon ignores it.
2. **The tunnel must be created in the playit.gg dashboard as
   `proxy-protocol-v2`.** Version 1 silently drops traffic with no error at
   all.

### 7.4 Self-update

Follow Blocksmith's `bs-update`: pick the newest `v*.*.*` tag, build and test
in a checkout, swap binaries only if everything passes, auto-roll-back if the
service does not come back up. Never touch state directories.

---

## 8. Repos and conventions

- `stevenjc2009-byte/tessera` — client. Public.
- `stevenjc2009-byte/tessera-server` — server. Public. Vendored into the client
  checkout as a dependency, mirroring `blocksmith` / `blocksmith-server`.
- Release convention, per the existing GitHub pattern: a `v<version>` branch
  plus annotated tag plus a published GitHub Release, branched off the previous
  tag tip, never merged to `main`.
- Commits authored as `final_destiny63 <stevenjc2009@gmail.com>`.
- Client ships as a **CIA**, so it can declare its own memory mode.

---

## 9. Phasing

Each phase ends with a build steve playtests before the next begins.

| Phase | Outcome |
|---|---|
| 1 | Skeleton: CIA builds and boots, dual-screen scaffold, touch input with smoothing, palette data loaded |
| 2 | Mode A complete — pixel canvas, all tools, undo, palettes, PNG export to SD |
| 3 | Mode E — colouring book, reusing the fill engine |
| 4 | Mode B — polygon face editor, snapping, triangulation, editing |
| 5 | Mode C — painting a face, per-face resolution, symmetry |
| 6 | Mode D — joining, hinging, net view, OBJ/MTL export |
| 7 | Server: HTTP daemon, SQLite, upload/browse/download, signing |
| 8 | Server: votes, favourites, reports, auto-hide, admin queue |
| 9 | Deployment: install scripts, nftables, systemd, playit, one-liner |
| 10 | Client gallery UI: browse, download, vote, favourite, report |

Phases 1-2 alone produce a shippable pixel art app. That is deliberate — the
2D editor is both a complete product and the foundation the rest needs.

---

## 10. Risks and unknowns

**Highest risk: the geometry editor (phase 4).** Polygon editing with
snapping, hinging, shared vertices and triangulation on a 320x240 single-touch
screen is the hardest part of the app and the most likely place for the
schedule to slip.

**Texture tiling.** Painting cannot write linearly into a PICA200 texture.
Whether to swizzle on write or render to texture must be decided in phase 2,
because it shapes the paint engine and is expensive to change later.

**Face picking without a touchable top screen.** The 3D preview cannot be
tapped. Face selection goes through the net view or a d-pad cursor. Which of
those feels better is not yet known and needs a playtest, not a decision on
paper.

**Blocksmith's server pattern is proven as deployment, not end-to-end.** Its
own README is candid: no traffic has crossed a real playit tunnel, and no real
3DS has ever connected. The container, build, hardening and units are proven on
real hardware; the network path is not. Tessera will be the first thing to
actually exercise it.

**Public gallery moderation load is unknown.** If it turns out to be more than
auto-hide plus an occasional review can handle, upload approval becomes
mandatory rather than optional.

**Memory.** ~20MB on Old 3DS is enough for what is specified, but it rules out
casual additions later — large layer stacks, deep undo, 1024x1024 per-face
textures. Budget must be tracked from phase 2, not discovered in phase 6.
