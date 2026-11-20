# Build and install

*Part of the [SixNG tool guide](README.md). Numbers and timings as of 2026-11-20.*

**Requirements:** Rust 1.80 or newer (stable). No other dependencies. The
tool is developed and tested on Windows; the GUI renders with wgpu
(Direct3D 12 or Vulkan). Other platforms may build but are untested.

This repository holds the documentation only. Build from the SixNG source:

```powershell
git clone <SixNG source repository url>
cd <checkout>
cargo build --release
```

This builds two programs in `target/release/`:

- `sixng` (`sixng.exe` on Windows): the CLI;
- `sixng-gui`: the desktop app.

**Game files.** SixNG reads the installed game files of your own Xbox
Series copy (the folder with the `*.rpf` archives and `data/ttdb/*.ttpf`
stores). It does not obtain them for you. See [LEGAL.md](../LEGAL.md).

**Two directories:**

| Setting | Flag | Environment variable | What it is |
|---|---|---|---|
| game root | `--game <dir>` | `SIXNG_GAME` | the game install. Always set it: the built-in default will not match your machine. |
| work dir | `--work <dir>` | `SIXNG_WORK` | indexes (`<work>/index/`), GUI cache and the default map bake. Default `./work`. |

Exports go wherever you point `-o/--out`.

**Disk space.** The install is about 224 GB. Indexes are small (about 95 MB
for `levels.rpf`). Exports can be very large; see the size tables in [CLI](02-cli.md#extracting-models-textures-and-files). Always try
`--dry-run` first.

---

[Index](README.md) · [CLI](02-cli.md) →
