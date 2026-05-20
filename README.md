# Diverge resource pack

Server-side resource pack for the [Diverge](https://github.com/Cooper-AmplioDev/diverge-core) survival plugin. The plugin references models in the `diverge` namespace (e.g. `diverge:life_orb`) via Paper's `ItemMeta.setItemModel(NamespacedKey)`; this pack supplies the matching textures + model JSON so those references render correctly on the client.

The plugin works without the pack — items just render as their base material (e.g. a `WIND_CHARGE` for the Life Orb) with the custom name, lore, and glint intact. The pack is what makes them *look* custom.

Pack downloads live on the [Releases page](https://github.com/Cooper-AmplioDev/diverge-resource-pack/releases). Point `resource-pack=` in your Paper `server.properties` at a release asset URL and include the matching `resource-pack-sha1=`.

## Layout

```
.
├── pack.mcmeta
└── assets/
    └── diverge/
        ├── models/
        │   └── item/
        │       └── life_orb.json     # model definition
        └── textures/
            └── item/
                └── life_orb.png      # the texture (any power-of-two resolution)
```

Each new custom item adds one model JSON + one texture PNG. The plugin references them by their namespaced key (e.g. `NamespacedKey("diverge", "life_orb")`), which Minecraft resolves to `assets/diverge/models/item/life_orb.json`.

## Building the zip

For a custom build (or to test changes locally before tagging a release):

```bash
zip -r diverge-resource-pack.zip pack.mcmeta assets/
```

That produces a single zip Minecraft can consume. The root of the archive must contain `pack.mcmeta` directly, not nested under a folder.

Generate a SHA-1 of the zip — Paper uses this to detect pack changes and force re-download on update:

```bash
shasum -a 1 diverge-resource-pack.zip | awk '{print $1}'
```

For releases, the easiest path is `gh release create vX.Y.Z diverge-resource-pack.zip --generate-notes` — this tags, publishes, and uploads the zip in one shot.

## Hosting the pack

Pick one of:

1. **GitHub Releases** — fastest. Tag a release, upload the zip as a release asset. The asset URL is stable and globally CDN-cached.
2. **Static CDN** — Cloudflare R2 / S3 / any object store. Public-read, set `Content-Type: application/zip`.
3. **Self-host on the server box** — run nginx alongside Paper and serve the zip from `/static/diverge-resource-pack.zip`. Fine for small player counts.

## Wiring `server.properties`

Once the zip is hosted, point Paper at it:

```properties
resource-pack=https://example.com/diverge-resource-pack.zip
resource-pack-sha1=<the sha1 from above>
resource-pack-prompt=Custom Diverge items
require-resource-pack=false
```

- `require-resource-pack=true` boots players who refuse the pack. Recommended for a survival server where custom items meaningfully affect gameplay (otherwise they'd see Life Orbs as plain hearts of the sea and not know what's going on).
- `resource-pack-prompt` is the message shown in the accept-pack dialog. Supports MiniMessage in Paper.
- `resource-pack-sha1` is technically optional but strongly recommended — without it, Paper can't tell when the zip has changed and clients will keep using the cached old version.

## Updating the pack

1. Edit the file under `assets/`.
2. Rebuild the zip + recompute the SHA-1 (see above).
3. Upload the new zip (same URL or new URL).
4. Update `resource-pack-sha1` in `server.properties`.
5. Restart Paper (or `/reload confirm`, but a restart is safer).

Clients will see the updated pack on their next reconnect.

## Pack format compatibility

`pack.mcmeta` declares `"pack_format": 46` with `supported_formats: { min: 32, max: 100 }`. The min bound covers 1.20.x-1.21.x; the max bound is intentionally far in the future so the pack keeps working across MC versions without you having to bump it every release. Adjust `min_inclusive` upward if you start using format-specific features.

If a future MC version genuinely breaks the schema (new model format, new texture path conventions, etc.), the pack will log a warning on load and you'll know it's time to bump.

## Adding a new custom item

1. Drop a PNG into `assets/diverge/textures/item/<name>.png` (16×16, 32×32, 64×64, 128×128 — any power of two; 64×64 is a reasonable default).
2. Add `assets/diverge/models/item/<name>.json`:
   ```json
   { "parent": "minecraft:item/generated", "textures": { "layer0": "diverge:item/<name>" } }
   ```
3. From the plugin, when building the item:
   ```kotlin
   itemMeta.setItemModel(NamespacedKey("diverge", "<name>"))
   ```
4. Rebuild + re-host the pack.
