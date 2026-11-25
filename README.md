# VIctory on an RX 7900 XTX

A fork of [VIctory](https://github.com/Marvisak/VIctory), a compatibility layer
that runs an Xbox Series title natively on Windows from files the user supplies.
This one is about **speed on AMD RDNA 3**: the world at 30 frames a second on
one PC, without giving up the picture. It is not there yet.

**This repository contains no game files, code or assets.** You need your own
copy of the game. It is not affiliated with or endorsed by the game's developer
or publisher.

## Where it stands (2026-11-25)

Tested on one machine only: Windows 11, Intel i9-12900K, 64 GB RAM, Radeon
RX 7900 XTX, 1440p, the Series X profile.

| | |
|---|---|
| Menus, videos, loading screens | 30 fps |
| Prologue, indoors | 50-75 ms a frame (13-20 fps) |
| First mission with the boat | 46-53 ms (19-22 fps) |
| Street chase with many cars and people | 100-165 ms (6-10 fps), single frames far longer |
| Outdoors on foot, the camp | 60-90 ms |
| Flying | 42-57 ms |
| Longest session so far | 41 minutes, no crash |
| Memory | 40-46 GB of private memory: **64 GB of RAM recommended** |

It started at 280 ms a frame in the same scenes two days earlier.

What is known to be wrong:

- Trees, plants and far buildings flicker outdoors in some places.
- Rooms with little direct light are too dark without `-RayTracing`.
- Textures of what just came into view appear late while the title streams.
- The first run with a new driver or a new build stutters while the driver makes
  pipelines (`victory warm` makes them ahead).
- Recording a frame in the world makes the title's own watchdog end it.

## Cómo ejecutarlo

VIctory ya está instalado en el PC. No hace falta instalar de nuevo el
compilador ni el juego. `scripts\play.ps1` usa el `victory.exe` de esta
carpeta (`target\diag` o, con `-Build release`, `target\release`) si existe
y, si no, el que esté en el PATH. `-Victory` señala otro binario. La caché
y las partidas siguen en `%LOCALAPPDATA%\victory`, la misma de esa
instalación, salvo que `victory.toml` o `VICTORY_CACHE_DIR` /
`VICTORY_DATA_DIR` indiquen otra ruta. El ejecutable del juego es el de tu
copia. Si hace falta un `victory.toml`, parte de `victory.example.toml` y
deja las rutas fuera de este repositorio.

```powershell
scripts\play.ps1 -Exe <ruta del ejecutable de tu copia>
```

Ese perfil es el medido en la RX 7900 XTX. Además de las vías rápidas de
este fork, las copias de búferes del juego van a la memoria de la tarjeta
desde 64 KiB (`VICTORY_CARD_BUFFERS_MIN_KB`). Un binario compilado de este
árbol hace lo mismo solo, cuando el presupuesto local de la tarjeta es de
al menos 20 GiB. No toca relojes ni voltajes. Para volver al suelo anterior
de 1 MiB: `-Safe`, o `-More @{VICTORY_CARD_BUFFERS_MIN_MB='1'}`.

`-Safe` también deja fuera `VICTORY_THIN`, `VICTORY_JOIN_PARTS` y
`VICTORY_UPLOAD_AHEAD`. Pruébalo primero si algo se ve mal. `-RayTracing`
enciende la luz indirecta con rayos del propio juego (interiores más
claros, cerca de un tercio más lento).

La primera vez con un driver o una build nuevos, antes de jugar (se puede
cortar y seguir luego):

```powershell
victory shaders <archivos de shaders>
victory warm --used
```

Si quieres el binario de este árbol en lugar del que ya está instalado:

```powershell
cargo build --locked --profile diag -p victory --features diagnostics
```

Comandos, interruptores de depuración y la descripción de upstream:
[docs/README-fxceless.md](docs/README-fxceless.md). Cada variable `VICTORY_...`
está explicada en el comentario donde se lee.

## What is different here

In short; the reasons and the measurements are in [docs/findings.md](docs/findings.md).

- **List calls are written down, not carried out, on the title's threads**
  (`VICTORY_THIN`): eight threads of ours translate them.
- **Submissions are prepared by a thread of ours and joined**
  (`VICTORY_ASYNC_SUBMIT`, `VICTORY_JOIN_PARTS`), with what can be spread over
  threads spread: refreshes, compares (AVX2), descriptor records, landings.
- **What shaders wrote comes back beside the write guards**, by the dwords that
  changed, on a thread of its own (`VICTORY_RELAY`, `VICTORY_LAND`).
- **Buffer fills that need bytes still on the card are done on the card**
  (`VICTORY_CARD_STEPS`) instead of waiting for it.
- **The title's writes are caught more cheaply**: one view opened at a fault,
  blocks that rest, places in the title's code that look before they write.
- **Textures go up on a copy queue of their own** and a slot shows one only after
  its upload ran, which also ended a device loss that came every 15 minutes or so.
- **Boats float**: textures that shaders write and the title's CPU reads go back
  to title memory (from fxceless and Geckomacabre, on by default here).
- **Frame record and replay** with per-pass times and shader variants compared
  turn by turn ([docs/replay.md](docs/replay.md)).

## Credits

This is a tree of other people's work with more on top. The history is kept, and
commits taken from elsewhere name the commit and its author.

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and its
  contributors: the loader, the runtime services, the Direct3D 12 back end and
  the shader recompiler.
- **[fxceless/victory-fork](https://github.com/fxceless/victory-fork)**: the fork
  this one started from, and the people whose pull requests it carries.
- **[DexNom/VIctory](https://github.com/DexNom/VIctory)**: ray tracing fixes,
  cube gradients, sparse textures.
- **[Belobug/VIctory-Belobug-AMD](https://github.com/Belobug/VIctory-Belobug-AMD)**:
  the ambient light idea, pipeline workers, and work by romainhedouin.
- **[FQY55kai/ZJOU-VICTORY](https://github.com/FQY55kai/ZJOU-VICTORY)**.
- **[myokex/victory-amd](https://github.com/myokex/victory-amd)**: the guard over
  every view of a page (with Spexivus, joroyt and tightfall), and Joshua Odom's
  reads into title memory.
- **Geckomacabre** and **cxy0072019-sketch**: the texture read-back.
- **[XCL](https://github.com/ivall/xcl)** (GPL-2.0-only) was read for how the same
  title behaves under another layer; no code of it is here.
- This fork's changes were made with [Claude Code](https://claude.com/claude-code).

## License

GPL-3.0-or-later, as VIctory. See [LICENSE](LICENSE); the one file that is not
covered is named in [docs/asset-policy.md](docs/asset-policy.md).
