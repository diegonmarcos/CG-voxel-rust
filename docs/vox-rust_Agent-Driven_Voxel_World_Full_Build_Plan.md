# vox-rust: Agent-Driven Voxel World — Full Build Plan

Oct 21, 2026 · @Diego

## Goals and non-goals

vox-rust is a Luanti-class voxel game engine rewritten in Rust, with one change of role: the world is built and edited by AI agents (bots) and humans join as players who explore, interact and talk to bots but never edit blocks. Quality target is at or above Luanti 5.15 on every player-facing feature; roughly 80% of the engine comes from existing Rust crates, and the rest (about 25–40k lines) is ours.

**Goals**

- Dedicated server in a VM; clients connect over the network (WebTransport, browser-first). Server is authoritative.
- Full world simulation at Luanti level: mapgen with biomes, caves and decorations; sunlight and light sources; liquids; falling nodes; timed block rules (ABM/LBM equivalents); day/night.
- Bots are first-class players: body, name tag, animation, inventory, chat; they move, path, place and break like a Luanti player would.
- Web client at Luanti quality or better: Bevy on WASM with WebGPU, first-person controller, HUD, inventory view, chat, menus, server list, texture and sound packs.
- Lua modding through an mlua-hosted `minetest.*` API at 100% of lua\_api.md, including entities, CSM and the Lua main menu, so ContentDB mods and games run unchanged.
- Existing Luanti worlds load (map.sqlite importer); own format is the default.
- Bot brains plug in over a socket (LLM, RL, scripted) and get a headless fast-forward mode for off-hours world building.

**Non-goals**

- Human block editing. No dig/place tools, no creative mode, no world edit for humans; the API refuses it server-side.
- Mobile native apps in v1 (Android/iOS later via Bevy mobile targets). Desktop builds are kept because they cost nothing; stock Luanti clients can also connect through the Luanti-protocol listener.
- Pixel-based bot observations.

## Architecture

&#91;embedded content: vox-rust architecture · 6 components, visual layer optional\]

Humans and bots are both clients of one server; every edit passes the player API and its permission check, and only bot sessions carry edit rights. Simulation, Lua, mapgen and persistence live beside the grid on the server, so the client only renders and sends input.

## Reuse inventory

Rows marked *verified* were checked against docs.rs, crates.io or the repo on 2026-10-21; the rest are well-known crates to confirm against Bevy 0.18 in Phase 0. *reference* rows are read or copied piecemeal, never added as dependencies. GPL projects are read only.

| Luanti subsystem | Crate / repo | Status | License | What it covers | Used as |
| --- | --- | --- | --- | --- | --- |
| Engine core (ECS, schedules, assets, audio, input, windowing) | [bevy](https://github.com/bevyengine/bevy) | 0.18 verified | MIT/Apache | everything Irrlicht + Luanti's own scheduler do | dependency |
| Chunk meshing and rendering | [bevy\_voxel\_world](https://github.com/splashdust/bevy_voxel_world) | 0.15.1 verified | MIT/Apache | multithreaded greedy meshing, chunk lifecycle, texture atlas, raycast, custom shader hook | dependency (client) |
| Meshing primitives | [block-mesh-rs](https://github.com/bonsairobo/block-mesh-rs) | 0.2 verified | MIT/Apache | greedy/simple meshers for anything bevy\_voxel\_world does not cover (e.g. liquids, plant-like nodes) | dependency |
| Lua modding (`minetest.*` API) | [mlua](https://github.com/mlua-rs/mlua) | current | MIT | Lua 5.1/5.4/LuaJIT VM embedding, sandboxing, async | dependency (server) |
| Networking | [lightyear](https://github.com/cBournhonesque/lightyear) or [bevy\_replicon](https://github.com/projectharmonia/bevy_replicon) + [renet](https://github.com/lucaspoffo/renet) | confirm 0.18 | MIT/Apache | server-authoritative replication, client prediction, WebTransport and WebSocket transports for browser clients | dependency |
| Player physics and character controller | [avian3d](https://github.com/Jondolf/avian) | confirm 0.18 | MIT/Apache | kinematic controller, AABB voxel colliders for loaded chunks only | dependency |
| Bot pathfinding | [bevy\_northstar](https://github.com/JtotheThree/bevy_northstar) | 0.6.2 verified | MIT | HPA\* 3D, nav masks, dynamic invalidation | dependency (server) |
| Mapgen noise | [noise](https://github.com/Razaekel/noise-rs) | 0.9 verified | MIT/Apache | Perlin/Simplex/Fbm/Ridged; Luanti's v7 and carpathian are noise stacks on top | dependency |
| World persistence | [rusqlite](https://github.com/rusqlite/rusqlite) + [zstd](https://github.com/gyscos/zstd-rs) | current | MIT | map.sqlite-style storage; same shape as Luanti's | dependency |
| Luanti world import | [minetestworld](https://github.com/UgnilJoZ/rust-minetestworld) | verified | MIT/Apache | reads Luanti map.sqlite, nodes and metadata | dependency (importer) |
| GUI (menus, inventory, chat, server list) | [bevy\_egui](https://github.com/vladbat00/bevy_egui) 0.39 | verified | MIT/Apache | formspec replacement, themeable | dependency (client) |
| HUD and in-world UI | Bevy UI (built in) | 0.18 | MIT/Apache | crosshair, hotbar, health, name tags, waypoints | dependency |
| Sound | bevy\_audio or [bevy\_kira\_audio](https://github.com/NiklasEi/bevy_kira_audio) | confirm 0.18 | MIT/Apache | positional node sounds, footsteps, ambience | dependency (client) |
| Textures and sounds | [minetest\_game](https://github.com/luanti-org/minetest_game) | CC BY-SA 3.0 | — | the default Luanti look; share-alike applies | asset |
| Textures (alt, no attribution) | [Kenney Voxel Pack](https://github.com/KenneyNL/Voxel-Pack) | CC0 | — | 16×16 blocks and models | asset |
| Player models | [minetest\_game](https://github.com/luanti-org/minetest_game) character.b3d → glTF | CC BY-SA | — | rigged humanoid for humans and bots | asset |
| Bot brains over socket | [tokio](https://github.com/tokio-rs/tokio) + [rmp-serde](https://github.com/3Hren/msgpack-rust) | current | MIT | Unix/TCP socket, MessagePack | dependency (server) |
| Compressed huge worlds (optional) | [voxelis](https://github.com/WildPixelGames/voxelis) | 25.4.0 verified | MIT/Apache | SVO-DAG store behind the `WorldGrid` trait | optional |
| Bevy voxel game with player loop | [rustcraft](https://github.com/c2i-junia/rustcraft) | verified | MIT | biomes, trees, inventory, hotbar, save, client/server | reference, copy code |
| Minecraft server rewrites in Rust | [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin), [Valence](https://github.com/valence-rs/valence) | current | MIT | chunk streaming, entity sync, plugin model; the proven shape for a server rewrite | reference |
| Full Rust voxel RPG | [veloren](https://gitlab.com/veloren/veloren) | current | GPL-3 | LOD, server/client split, worldgen | reference only |
| The spec we are matching | [luanti](https://github.com/luanti-org/luanti) | 5.15.2 verified | LGPL-2.1 | Lua API reference, mapgen parameters, lighting rules, node definitions | reference only |

**Added 2026-10-21 after a GitHub sweep for Luanti-specific Rust code.** The Luanti ecosystem pieces already in Rust cut the protocol, map-format, auth and model-loader work to integration. Where two crates cover the same thing (minetest-protocol vs luanti-protocol, mtmap vs minetestworld), the newer or better maintained one is the dependency and the other a reference.

| Luanti subsystem | Crate / repo | Status | License | What it covers | Used as |
| --- | --- | --- | --- | --- | --- |
| Luanti wire protocol (RUDP, packets, commands) | [minetest-protocol](https://github.com/paradust7/minetest-protocol) | 0.1.4, 2023, verified | MIT | packet (de)serialization, typed ToClient/ToServer commands, channels, split packets, reliable retries, peer ids | dependency: stock Luanti clients can connect |
| Luanti protocol, second implementation | [minetest-rust/mt\_net](https://github.com/minetest-rust/mt_net), [mt\_rudp](https://github.com/minetest-rust/mt_rudp), [mt\_ser](https://github.com/minetest-rust/mt_ser), [mt\_auth](https://github.com/minetest-rust/mt_auth), [mt\_client](https://github.com/minetest-rust/mt_client) | verified org, activity low | MIT | RUDP, serialization, SRP auth, a client skeleton | reference / fallback to minetest-protocol |
| Luanti map database format | [mtmap](https://github.com/LizzyFleckenstein03/mtmap) | verified | MIT | map.sqlite block (de)serialization | dependency (persistence, importer) |
| Headless Luanti bot clients | [hydra](https://github.com/minetest-rust) (Lizzy Fleckenstein) | verified mention | — | Lua API for headless bot clients against a Luanti server | reference for bot client design |
| Luanti in the browser | [luanti-wasm](https://github.com/paradust7/minetest-wasm) | verified, C++/emscripten | MIT | the whole engine as a web app; WebSocket-to-UDP proxy pattern for network play | reference: the web client architecture we are matching |
| GPU particles | [bevy\_hanabi](https://github.com/djeedai/bevy_hanabi) | confirm 0.18 | MIT/Apache | particle spawners; needs compute shaders, so WebGPU only; CPU fallback for WebGL2 is ours | dependency (client) |
| Sky, sun, moon, stars, fog | Bevy built-in atmosphere (0.16+) + [bevy\_atmosphere](https://github.com/JonahPlusPlus/bevy_atmosphere) | confirm 0.18 | MIT/Apache | physically based sky, `set_sky` mapping | dependency (client) |
| Localization | [bevy\_fluent](https://github.com/kgv/bevy_fluent) or [fluent](https://github.com/projectfluent/fluent-rs) + [gettext-rs](https://github.com/gettext-rs/gettext-rs) for mod .po files | confirm | MIT/Apache | engine i18n; Luanti mods ship gettext .tr/.po | dependency |
| Touch controls | [bevy\_touch\_stick](https://github.com/Jeremy-Turner/bevy_touch_stick) or Bevy touch input | confirm 0.18 | MIT/Apache | virtual joystick for mobile browsers | dependency (client) |
| World and auth backends | [sqlx](https://github.com/launchbadge/sqlx) (PostgreSQL), [rusty-leveldb](https://github.com/dermesser/leveldb-rs), [redis-rs](https://github.com/redis-rs/redis-rs) | current | MIT/Apache | the three extra Luanti backends | dependency (server) |
| Minimap | own, on Bevy UI | — | — | surface and radar modes from server-sent height/light data | ours |
| Lua main menu | mlua + formspec renderer (same path as in-game formspecs) | — | — | Luanti's menu is Lua over formspecs | ours |

**Second sweep, 2026-10-21: every Luanti-specific Rust project found on crates.io and GitHub**

| Luanti subsystem | Crate / repo | Status | License | What it covers | Used as |
| --- | --- | --- | --- | --- | --- |
| Luanti wire protocol, maintained fork | [luanti-protocol](https://github.com/kawogi/luanti-rs) 0.2.0 (luanti-rs) | Mar 2025, verified | MIT | successor of minetest-protocol with the Luanti rename; same packet, command, RUDP and reliability coverage; `luanti` meta-crate re-exports it | dependency: replaces the minetest-protocol row above |
| Luanti file and network formats + wgpu map viewer | [luanti\_dsrs](https://github.com/Desour/luanti_dsrs) (Desour, Luanti core dev) | Apr 2025, verified | — | map.sqlite deserialization matched to Luanti sub-versions, nodedef JSON, simple wgpu meshing | reference for format edge cases |
| Luanti worlds, read and modify | [minetestworld](https://github.com/UgnilJoZ/rust-minetestworld) 0.6.1 | verified | MIT/Apache | MapBlock get/set, VoxelManip-style node access, async | dependency (importer, tooling) |
| World mapper | [minetest-worldmapper](https://github.com/UgnilJoZ/minetest-worldmapper) 0.3.7 | verified | MIT/Apache | top-down map render with transparency | tooling as-is |
| Protocol inspector | [minetest-shark](https://github.com/paradust7/minetest-protocol) / luanti-shark | verified | MIT | proxy that decodes every packet | debugging tool as-is |
| .b3d models in Bevy | [bevy\_b3d](https://docs.rs/bevy_b3d) + `b3d` crate | verified on docs.rs, confirm 0.18 | MIT/Apache | loads Luanti's character.b3d and mod models directly, no glTF conversion | dependency (client) |
| glTF for Luanti models | [minetest-gltf](https://github.com/jordan4ibanez/minetest-gltf) 3.0.1 | verified | MIT | easy-gltf fork with bone animation keyframes, built for a Luanti rewrite | dependency or reference |
| Earlier ground-up rewrite | [jordan4ibanez/minetest-rust](https://github.com/jordan4ibanez/minetest-rust) | verified exists, wgpu + SDL2, activity unknown | — | the previous attempt at exactly this project | reference: mine it for Lua-API and loader code before writing ours |

**Searched and found nothing in Rust (confirmed ours to write):** formspec parser or renderer, any Luanti mapgen (v5/v6/v7/valleys/carpathian/fractal) port, .mts schematic reader, .x model loader, Luanti lighting or liquid simulation, any `minetest.*` Lua API reimplementation beyond jordan4ibanez's partial attempt.

**Dropped:** `bevy_rl`, `voxy`, `big-brain`, `seldom_state` (see earlier revisions); none fit a game engine.

## What we write ourselves

About 40–55k lines of Rust after the full-parity audit. The crates give us rendering, physics, networking transport, a Lua VM and meshing; Luanti's actual game rules, its Lua API surface and the bot layer are ours. Sizes are estimates of final code, not first drafts.

| Module | Lines (est.) | Covers | Notes |
| --- | --- | --- | --- |
| `WorldGrid` + node registry | 1,500 | chunked node store (16³ blocks like Luanti's MapBlock), node ids, param1/param2, metadata | source of truth on the server; `slice`, `dirty`, chunk streaming |
| Lighting | 1,200 | sunlight column propagation, point lights, flood fill on edit, day/night ratio | Luanti's algorithm, ported; runs on server, shipped as param1 |
| Liquids and falling nodes | 900 | flowing/source liquids, viscosity, falling sand/gravel | Luanti's `liquid` loop, simplified first |
| ABM / LBM scheduler | 500 | timed per-node rules (grass spread, fire, decay), run from Lua or Rust | needed for mod parity |
| Mapgen | 2,500 | v7-style: heightmap, biomes by heat/humidity, caves, ores, decorations, dungeons | noise stacks only; parameters copied from Luanti's defaults |
| Lua API (`minetest.*`) | 8,000+ | node/item/craft registration, callbacks, chat, HUD, formspec parsing, player API, metadata, timers | the largest item; tracked as an API coverage table |
| Formspec interpreter → egui | 1,500 | render Luanti formspec strings as egui widgets | lets mod UIs work unchanged |
| Server: sessions, auth, chunk streaming, entity sync | 2,000 | on top of lightyear/replicon | permission model refuses human edits |
| Client: first-person controller, interaction, HUD, inventory, chat, menus, server list, settings | 3,000 | on top of avian3d, egui, Bevy UI | copy rustcraft where it fits |
| Player and bot avatars | 600 | glTF model, animation states, name tags | shared by humans and bots |
| Bot layer: executor, northstar bridge, observation builder, socket harness | 2,000 | the earlier plan, kept; bots act through the same player API | headless fast-forward mode |
| Persistence | 800 | SQLite map store, player data, mod storage; Luanti importer |  |
| Sound and texture packs | 400 | pack discovery, atlas build, positional audio hooks |  |
| Tooling and tests | 2,000 | world diff, replay, determinism, API coverage report |  |

**Added 2026-10-21 for full parity**

| Module | Lines (est.) | Covers | Notes |
| --- | --- | --- | --- |
| Remaining mapgens | 4,000 | v5, v6, flat, fractal, valleys, carpathian, singlenode; schematics; dungeons | noise stacks ported from Luanti with its default parameters |
| Entity / object API | 3,000 | ObjectRef, LuaEntity, attachments, animation, item entities | required by every mob mod |
| Particles | 800 | spawner definitions → bevy\_hanabi, CPU fallback for WebGL2 |  |
| Sky and lighting API | 600 | set\_sky, set\_sun/moon/stars, clouds, fog, set\_lighting | maps onto Bevy atmosphere |
| Minimap | 500 | surface + radar from server-sent data |  |
| Lua main menu + ContentDB browser | 1,500 | formspec-based menu, package install into browser storage |  |
| CSM | 1,200 | sandboxed Lua in the web client, client callbacks |  |
| Luanti protocol listener | 1,000 | luanti-protocol integration, SRP auth, media transfer | stock clients connect |
| Extra backends | 800 | PostgreSQL, LevelDB, Redis behind one `MapStore` trait |  |
| Localization | 500 | fluent for engine, gettext for mods, formspec translation |  |
| Touch and input remapping | 600 | virtual joystick, bindings UI, Gamepad API |  |
| Media: packs, overrides, remote media, b3d/x loaders | 1,200 | texture override files, remote media server, model loaders | .b3d via bevy\_b3d, glTF via minetest-gltf; only the .x parser is ours |
| Screenshots, /status, announce, IPv6 | 400 |  |  |

## Workspace layout and Cargo features

One workspace, six crates. The server binary runs in the VM with no GPU and never links wgpu; the client is a WASM web app served from the same VM.

```
vox-rust/
  Cargo.toml                 # workspace; opt-level = 3 for deps in dev
  crates/
    vox-core/                # WorldGrid, node registry, lighting, liquids, mapgen, persistence. glam + noise + rusqlite. No Bevy.
    vox-proto/               # network message types, serde + rmp; shared by server and client
    vox-lua/                 # mlua host, minetest.* API, formspec parser (data model only)
    vox-server/              # binary: Bevy MinimalPlugins + lightyear server, sessions, chunk streaming, ABM scheduler, bot harness, socket for brains
    vox-client/              # one crate, two targets: wasm32 web app (primary) + native desktop builds from CI: Bevy DefaultPlugins + bevy_voxel_world + avian3d + egui; formspec renderer; HUD, menus
    vox-web/                 # static site: index.html, wasm loader, served by the VM behind TLS (WebTransport needs HTTPS)
    vox-bots/                # executor, northstar bridge, observations; depends on vox-core + vox-proto only
  python/voxrust/            # brain client for LLM/scripted bots
  games/                     # Lua games and mods (minetest_game first)
  assets/                    # textures, sounds, models, shaders
  deploy/                    # systemd unit, VM image notes, firewall rules
```

Feature gates:

- `vox-server` uses `bevy` with `default-features = false` (ecs, app, time, log, tasks). Adding `--features headless-fast` disables the network tick and runs the bot loop unthrottled for off-hours world building.
- `vox-client` enables `bevy/default`, `bevy_voxel_world`, `avian3d`, `bevy_egui`, audio.
- `vox-core` optional `voxelis` backend behind the same `WorldGrid` trait.
- `vox-lua` optional `luajit` feature (mlua `luajit`) for mod-heavy servers; default Lua 5.1 for Luanti API compatibility.

API note for Bevy 0.18: buffered events are `Message`, `MessageWriter`, `MessageReader` with `.write()`; `#[derive(Event)]` is for observers only.

## Roadmap

Ten phases, each gated by something a human can see or a test can prove. Estimates assume a small team driving coding agents full-time; a solo part-timer should double them. A human walks a networked bot-built world at the end of Phase 3; full Luanti parity (everything except human editing) is the end of Phase 8.

1. **Phase 0, bootstrap (1 week)** — workspace, CI, Bevy 0.18 pins, confirm every "confirm 0.18" crate compiles together, wasm32 toolchain; salvage pass over jordan4ibanez/minetest-rust and luanti\_dsrs to list what code can be lifted (Lua API stubs, loaders, format readers) before anything is written. Gate: server builds on the Linux VM and the client builds for wasm32-unknown-unknown and loads in Chrome and Firefox.
2. **Phase 1, core world (3 weeks)** — `WorldGrid`, node registry, persistence through mtmap-compatible map.sqlite, Luanti world import without conversion. Gate: a minetest\_game world loads and re-saves byte-identical; 1,000-block mapgen under 50 ms per block.
3. **Phase 2, networked server + web client (5 weeks)** — lightyear over WebTransport for the web client, luanti-protocol 0.2 for stock Luanti clients, sessions, SRP auth (mt\_auth as reference), map.sqlite through minetestworld and mtmap, chunk streaming by distance, first-person controller with avian3d, bevy\_voxel\_world rendering, HUD. Gate: a browser client and a stock Luanti 5.15 client walk the same world together with no desync for 10 minutes.
4. **Phase 3, bots (3 weeks)** — bot avatars, executor, northstar bridge, socket harness, `BuildBox`/`PathTo`, human edit refusal. Gate: 10 bots build a village while a human walks through it in the browser; humans cannot place or break anything.
5. **Phase 4, world simulation (5 weeks)** — lighting, day/night, liquids, falling nodes, ABM/LBM scheduler, node timers, particles and particle spawners, sky API (skybox, sun, moon, stars, clouds, fog, `set_lighting`). Gate: side-by-side screenshots with Luanti on the same imported world; a bot digs a channel and water flows; weather mods run.
6. **Phase 5, all mapgens (4 weeks)** — v5, v6, v7, flat, fractal, valleys, carpathian, singlenode with Luanti's default parameters, biomes, ores, decorations, dungeons, schematics. Gate: each mapgen produces the same terrain classes as Luanti for the same seed, verified by world diff on 100 blocks.
7. **Phase 6, Lua API complete (12–16 weeks, ongoing)** — mlua host (LuaJIT, 5.1 fallback), starting from whatever the salvage pass lifted from minetest-rust, 100% of `lua_api.md`: nodes, items, crafts, callbacks, chat commands, privileges, HUD, formspecs, metadata, inventories, mod storage, mod security sandbox, `request_insecure_environment`, entity/object API (mobs, vehicles, item drops), player API, client-side modding (CSM), Lua main menu. Gate: minetest\_game loads with zero Lua errors; API coverage report at 100% of documented functions; VoxeLibre and Mineclonia start.
8. **Phase 7, client complete (5 weeks)** — Lua main menu rendered through the formspec path, in-client ContentDB browser, server list and master-server announce, settings, input remapping, touchscreen controls for mobile browsers, texture packs with overrides, `.b3d`/`.x` model loading (bevy\_b3d, minetest-gltf, own .x parser), remote media servers, positional audio, minimap (surface + radar), screenshots, chat with bot addressing, inventory and crafting view. Gate: a new player opens the URL in a browser, installs a texture pack from ContentDB, and plays 30 minutes without the console.
9. **Phase 8, platform parity (3 weeks)** — PostgreSQL, LevelDB and Redis world/auth backends, IPv6, `/status` and server announce, gettext localization for engine and mods, translations in formspecs. Gate: switch a world between SQLite and PostgreSQL with no data change; UI in Spanish and English.
10. **Phase 9, beyond Luanti (ongoing)** — LOD rendering past Luanti's view range, Voxelis backend for huge worlds, bot fleet tooling (blueprints, task boards), client prediction tuning, spectator/replay mode. Gate per item.

Total to Phase 8: roughly 11–13 months with a team, 22–26 months solo. First public release at the end of Phase 4 (playable, bots building, not yet mod-complete); parity release at the end of Phase 8.

## Network and bot protocols

Two protocols, one authority. Game clients (humans) and bots both talk to the server; bots additionally expose a socket where their brains plug in. Every world edit passes the same permission check, and human sessions carry `can_edit = false`.

**Game protocol (server ↔ client, lightyear over WebTransport)**

| Direction | Message | Purpose |
| --- | --- | --- |
| client → server | `Hello { name, auth, client_version }` | session + role (human / bot) |
| server → client | `WorldInfo { node_defs, item_defs, media_hashes, time }` | registry sync, same as Luanti's TOCLIENT\_NODEDEF |
| server → client | `Chunk { pos, nodes, light, metadata }` (zstd) | streamed by distance, ring order |
| server → client | `NodeChanged { pos, node, light_delta }` | incremental edits from bots and simulation |
| client → server | `PlayerInput { move, look, jump, sneak, interact_target }` | humans; `interact` is use/punch only, never dig/place |
| server → client | `EntitySync { players, bots, objects }` | positions, animation state, name tags, 20 Hz |
| both | `Chat`, `Formspec`, `HudChange`, `Inventory` | mirrors Luanti's packets 1:1 so mods behave |

**Bot protocol (server ↔ brain, MessagePack over Unix/TCP socket)**

- `Observe` → local node grid (7³ default), position, yaw, inventory, nearby entities, last action result, chat since last tick.
- Actions: `Noop`, `Move(dir)`, `Turn`, `Place`, `Break`, `Use`, `Say(text)`, `PathTo(pos)`, `BuildBox { min, max, node }`, `BuildBlueprint(id)`.
- Lockstep is optional: in normal play bots run on the 20 Hz server tick and slow brains get `Noop`; in `headless-fast` mode the server waits for every brain each tick.

**Rules shared by both:** bots are processed in id order; two writes to one node in one tick resolve to the lowest id; a human `PlayerInput` that would edit is dropped and logged, never applied.

## Luanti parity audit

Verdict: at the end of Phase 8 vox-rust matches or exceeds Luanti 5.15 on every capability except one chosen difference, no human dig/place, and one pending item, native mobile apps. The first table below was the Phase 6 view; the second adds the capabilities a full Luanti audit turned up, all now scheduled.

| Capability | Luanti 5.15 | vox-rust at Phase 6 | Verdict |
| --- | --- | --- | --- |
| Voxel world, chunk streaming, node registry | yes | yes, same 16³ block model | parity |
| Mapgen | v5/v6/v7/flat/carpathian/valleys + biomes, caves, dungeons | v7-style + biomes + caves + ores; others in Phase 7 | lag (variety) |
| Lighting, day/night | yes | yes, same propagation rules, PBR shading on top | better (visuals) |
| Liquids, falling nodes | yes | yes | parity |
| ABM / LBM, node timers | yes | yes | parity |
| Lua modding `minetest.*` | full | ≥ 80% of what minetest\_game uses, growing | per-mod |
| Formspecs | yes | rendered through egui | parity, nicer |
| Multiplayer server, auth, privileges | yes | yes, WebTransport, client prediction, playable in a browser | better (netcode) |
| Client: first-person, HUD, inventory, chat, menus, server list | yes | yes | parity |
| Texture and sound packs | yes | yes | parity |
| Player models and animation | yes | yes, glTF | parity |
| Entities / mobs via mods | yes | Phase 7 | lag until then |
| World persistence | map.sqlite | SQLite, plus importer for Luanti worlds | parity |
| Rendering range / LOD | fixed view range | LOD beyond view range | better |
| Bots as players | mods only (basic\_robot, maidroid) | native, pathing, socket brains | better |
| Human editing | yes | removed by design | difference |
| Luanti network protocol | yes | own protocol; shim optional | difference |

**Capabilities added 2026-10-21 so the table covers all of Luanti**

| Capability | Luanti 5.15 | vox-rust at Phase 8 | Verdict |
| --- | --- | --- | --- |
| All 8 mapgens with default parameters, schematics, dungeons | yes | yes (Phase 5) | parity |
| Entity / object API, mobs via mods | yes | yes (Phase 6) | parity |
| Particles and particle spawners | yes | yes, GPU on WebGPU, CPU on WebGL2 | parity |
| Sky API: skybox, sun, moon, stars, clouds, fog, lighting | yes | yes | parity |
| Minimap, surface and radar | yes | yes | parity |
| Lua main menu, in-client ContentDB browser | yes | yes, through the formspec renderer | parity |
| Client-side modding (CSM) | yes | yes, sandboxed in the web client | parity |
| Mod storage, mod security, insecure environment | yes | yes | parity |
| World backends: SQLite, PostgreSQL, LevelDB, Redis | yes | yes | parity |
| Auth backends, SRP | yes | yes | parity |
| Localization: engine and mod translations | yes | yes | parity |
| Touchscreen controls, input remapping, joystick | yes | touch + remap in browser; joystick via Gamepad API | parity |
| Texture packs, overrides, remote media, .b3d/.x models | yes | yes | parity |
| Screenshots, /status, master-server announce, IPv6 | yes | yes | parity |
| Luanti network protocol | yes | yes, via luanti-protocol; stock clients connect | parity |
| Desktop / mobile native clients | yes | web app primary; Linux, Windows, macOS native from the same crate; mobile later | difference |
| Human dig and place | yes | removed by decision | difference |

With these rows the verdict changes: every Luanti capability is at parity or better after Phase 8 except human editing (removed on purpose) and native mobile apps (later).

## Risks, open questions, rejected options

**Risks**

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Lua API surface is the long tail: thousands of functions, mods rely on undocumented behaviour | high | prioritize by minetest\_game call frequency; API coverage report in CI; accept per-mod parity |
| Bevy 0.19 ships mid-project and lightyear / avian / bevy\_voxel\_world lag by weeks | high | pin 0.18 until Phase 6; isolate Bevy-facing code in vox-server and vox-client |
| Netcode: client prediction for first-person on a voxel world is fiddly | medium | start with server-authoritative + interpolation; add prediction only for the local player's movement |
| Lighting + liquids are the two subsystems people judge by eye | medium | port Luanti's algorithms directly, test with imported worlds side by side |
| bevy\_voxel\_world cannot render non-cube node types (plantlike, nodebox, mesh, liquids) | high | its custom-meshing hook plus block-mesh for extra drawtypes; budget 1,500 lines |
| Scope: this is a 1–2 year project even with coding agents | certain | phase gates are the control; every phase ends playable |
| Browser limits: no threads without cross-origin isolation, WebGPU not universal, 2–4 GB memory ceiling | high | serve with COOP/COEP headers for wasm threads; WebGL2 fallback; smaller view range in the browser than native, never fewer features |

**Decisions (2026-10-21)**

- Lua: LuaJIT by default with Lua 5.1 fallback, matching Luanti.
- Humans: everything a Luanti player can do except dig and place. Doors, chests, signs, levers, crafting view, chat, inventory and privileges all stay; only the editing verbs and their UI go.
- First game: minetest\_game, as the compatibility bar.
- Platforms: Linux only for now. The server runs in a Linux VM; the primary client is the web app (Bevy on wasm32, WebGPU with WebGL2 fallback), served by the VM and opened in a browser. The same crate also compiles natively, so Linux, Windows and macOS desktop builds ship as free by-products of CI; nothing is cut. Android and iOS come later through Bevy's mobile targets.
- Transport: WebTransport (lightyear supports it) because browsers cannot open raw UDP/QUIC; WebSocket fallback for restrictive networks.

**Rejected options**

- Running stock Luanti with a Rust bot server beside it: meets "bots build, humans play" in weeks, but it is not a Rust engine and cannot exceed Luanti's quality.
- Forking Veloren: GPL and a different game; its netcode and LOD are read-only inspiration.
- Reimplementing Luanti's wire protocol first: a lot of work for compatibility with clients we are replacing anyway; kept as a Phase 7 option.
- bevy\_rl, voxy, big-brain, seldom\_state, pixel observations: not game-engine material.

## Token budget and cost controls

Target: cut the agent budget from 1.5–4.5B input / 30–60M output tokens to roughly 0.6–1.2B input / 15–25M output, about a 60% reduction, without cutting scope. The rules below are binding for every coding-agent session on this repo.

**1. Repository layout for small context**

- One crate per subsystem (already in the workspace plan) plus a `CLAUDE.md` per crate: purpose, public API, invariants, how to test, in under 300 words. Agents read the crate's file, never the whole repo.
- `docs/luanti-ref/`: `lua_api.md` split into one file per API section (nodes, items, entities, formspecs, mapgen…). A task links one section, not the 12k-line original.
- `docs/decisions/`: one short ADR per architectural decision from this plan, so agents do not re-derive them.
- No agent reads crate source from `~/.cargo`; API notes for each dependency live in `docs/crates/<name>.md`, written once.

**2. PR size and session shape**

- One PR = one API function group, one mapgen, one packet family, or one node drawtype. Target under 300 lines landed, under 40 agent turns.
- Every PR starts from a written spec in the issue: inputs, outputs, acceptance test. No exploratory sessions on the main repo; exploration happens in a scratch crate and only the conclusion is written down.
- Sessions end when the test passes; refactors are their own PR.

**3. Model tiering**

| Work | Model class | Why |
| --- | --- | --- |
| Architecture, netcode, lighting/liquid algorithms, debugging cross-crate bugs | strongest model | judgment-heavy, errors are expensive |
| Lua API functions with a clear spec, mapgen parameter ports, packet structs, formspec elements, tests | mid-tier model | mechanical, spec-driven, cheap to verify |
| Doc comments, CLAUDE.md upkeep, changelog, renaming | smallest model | no judgment needed |

**4. Generate, don't write**

- Packet structs and serializers come from luanti-protocol; never retyped.
- `minetest.*` function stubs with signatures and doc strings are generated by a script from the split `lua_api.md`; agents fill bodies only.
- Mapgen noise parameter tables are extracted from Luanti's `defaultsettings.cpp` and `mapgen_*.cpp` by script into RON files; the Rust mapgen reads them.
- Node and item definitions for tests come from a dumped `nodedefs.json` (the luanti\_dsrs method), not hand-written.

**5. Short debug loops**

- `cargo check` before `cargo test`; `cargo nextest` with the failing test only; compiler output truncated to the first error.
- Golden-file tests for mapgen, lighting and liquids: a world block in, a hash out. A failing agent sees one hash mismatch, not 10,000 nodes.
- Build cache (sccache) and pinned toolchain so no session pays for a cold Bevy build.

**6. Caching and context hygiene**

- Prompt caching on every session; stable system prompt + CLAUDE.md first, task last, so the cached prefix is large.
- Agents summarize their own state every 20 turns into the PR description and restart from it rather than carrying 100-turn histories.
- Tool output trimmed: test runners print pass/fail counts and the first failure only.

**7. Reuse before writing (Phase 0 salvage pass)**

- Inventory jordan4ibanez/minetest-rust, luanti\_dsrs, rustcraft and the minetest-rust org for liftable code; every lifted file is credited in `THIRD_PARTY.md`. Lifting 5k lines saves roughly 100M input tokens of agent work.

**8. Version freeze**

- Bevy 0.18 and all plugin versions pinned until Phase 8; one planned upgrade PR per phase boundary, never mid-feature. Every unplanned upgrade costs 5–10 PRs of churn.

**Expected effect by lever:** small context and generated stubs \~30%, model tiering \~25% of cost (not tokens), caching \~20% of billed input, short debug loops \~15%, salvage \~10%.

## Autonomous verification: taking the human out of the loop

Goal: a PR merges with no human reading it when the automated gates pass, and the human sees only a daily digest and the handful of PRs the gates could not decide. Every gate below is machine-checkable; most reuse an existing test engine rather than tests we write.

**1. Reference oracle: Luanti itself**

- Run stock Luanti 5.15 headless in CI as the oracle. For mapgen, lighting, liquids and ABMs the test is differential: same seed, same world, same actions in both engines, compare node grids. Zero hand-written expected values.
- Luanti's own `devtest` game (ships with the engine, exercises most of `lua_api.md`) runs against vox-server; every Lua error is a failing test. Luanti's `builtin` unit tests and `games/devtest/mods/unittests` run unchanged.
- Stock Luanti client connects to vox-server through luanti-protocol in CI and joins, moves, chats; a disconnect or protocol error fails the build.

**2. Existing test engines reused**

| Layer | Engine | What it checks |
| --- | --- | --- |
| Lua API | Luanti `devtest` + `unittests` mod, `busted`, `mineunit` (Minetest mod unit test framework) | API coverage, callbacks, formspec strings |
| Rust units | `cargo nextest`, `proptest` (property tests), `insta` (snapshot tests) | grid, lighting, liquids, serializers |
| Determinism | own harness: run twice, hash world + action log | any nondeterminism fails |
| Protocol | luanti-shark captures as fixtures; `cargo fuzz` on packet decoders | wire compatibility, crash safety |
| Web client | Playwright: load page, connect, walk, screenshot | client boots, renders, reaches the world |
| Visual parity | screenshot diff against Luanti on the same world and camera (`pixelmatch` / `odiff`, SSIM threshold) | lighting, sky, node drawtypes |
| Bots | scenario files with a target structure; pass = structure built within N ticks | pathing, executor, API |
| Performance | `criterion` benches with regression thresholds; tick-rate budget per phase gate | no silent slowdowns |
| Mods | top 20 ContentDB mods and VoxeLibre start in CI; log scanned for Lua errors | real-world compatibility |

**3. Agents that test and review agents**

- Author agent writes code and tests. A second, separate reviewer agent (different prompt, no shared context) reviews the diff against the issue spec and the crate's CLAUDE.md, and may request changes; it cannot approve its own author.
- A tester agent runs the scenario and visual suites, reads failures, and either fixes or writes a reproduction into the issue.
- Merge queue merges automatically when: all gates green, reviewer agent approves, coverage not reduced, no new `unsafe`, benchmarks within threshold.

**4. What still reaches the human**

- A daily digest: PRs merged, gates failed, mods broken, performance deltas.
- Any PR the reviewer agent rejects twice.
- Any change to a public API, a wire format, or `docs/decisions/`.
- Visual diffs above threshold that the tester agent cannot attribute to an intended change.

Expected effect: human time drops from reading 300 PRs to roughly 20–40 escalations plus daily digests, which moves the solo calendar from 90–180 days toward 45–90, bounded by agent quota rather than review time.
