# Diverge resource pack

> **Published pack:** [Cooper-AmplioDev/diverge-resource-pack](https://github.com/Cooper-AmplioDev/diverge-resource-pack) — releases page is the canonical place to grab the zip for `server.properties`. The folder you're looking at right now is the working copy that gets edited then exported to that repo when cutting a new version.

This folder is the source for the Diverge custom-items resource pack. The plugin references models in the `diverge` namespace (e.g. `diverge:life_orb`); this pack supplies their textures + model JSON.

The plugin works without the pack — items just render as their base item (e.g. a `WIND_CHARGE` for the Life Orb) with the custom name and lore intact. The pack is what makes them *look* custom.

## Layout

```
docs/resource-pack/
├── pack.mcmeta
└── assets/
    └── diverge/
        ├── items/
        │   └── life_orb.json         # NEW (1.21.4+): client item definition;
        │                             # this is what setItemModel resolves to
        ├── models/
        │   └── item/
        │       └── life_orb.json     # plain model (parent + textures);
        │                             # referenced from items/life_orb.json
        └── textures/
            └── item/
                └── life_orb.png      # the texture (any power-of-two resolution)
```

The two-file split (`items/` + `models/item/`) is the post-1.21.4 layout. Server-side, the plugin calls `ItemMeta.setItemModel(NamespacedKey("diverge", "life_orb"))`. The client looks for that identifier at `assets/diverge/items/life_orb.json` — the *items* file. That items file then points at the plain model in `assets/diverge/models/item/life_orb.json`, which carries the texture references like before.

The new `items/` file can do more than a plain model lookup — `range_dispatch`, `select`, `condition` etc. let you swap models based on item state. For static textures like the Life Orb we keep it as a single `minecraft:model` entry.

Each new custom item adds **three** files:

1. `assets/diverge/items/<name>.json` (the client item definition)
2. `assets/diverge/models/item/<name>.json` (the plain model JSON)
3. `assets/diverge/textures/item/<name>.png` (the texture)

## Building the zip

```bash
cd docs/resource-pack
zip -r ../diverge-resource-pack.zip pack.mcmeta assets/
```

That produces a single zip Minecraft can consume. (Don't zip the *folder* — zip its *contents*. The root of the archive must contain `pack.mcmeta`, not `resource-pack/pack.mcmeta`.)

Generate a SHA-1 of the zip — Paper uses this to detect pack changes and force re-download on update:

```bash
shasum -a 1 docs/diverge-resource-pack.zip | awk '{print $1}'
```

## Hosting the pack

Current setup: the pack is hosted on GitHub Releases at [Cooper-AmplioDev/diverge-resource-pack](https://github.com/Cooper-AmplioDev/diverge-resource-pack/releases). The release URLs redirect to GitHub's release-asset CDN, which serves anonymous requests with global caching — exactly what Paper wants.

If you ever need to switch hosts:

1. **GitHub Releases** (current) — tag a release, upload the zip as a release asset. URL is stable and globally CDN-cached.
2. **Static CDN** — Cloudflare R2 / S3 / any object store. Public-read, set `Content-Type: application/zip`.
3. **Self-host on the server box** — run nginx alongside Paper and serve the zip from `/static/diverge-resource-pack.zip`. Fine for small player counts.

## Wiring `server.properties`

Current production values (v0.1.0):

```properties
resource-pack=https://github.com/Cooper-AmplioDev/diverge-resource-pack/releases/download/v0.1.0/diverge-resource-pack-v0.1.0.zip
resource-pack-sha1=5d6a871bab3d75ae6f1bbee9b416cdd3a74c49d5
resource-pack-prompt=Diverge custom items
require-resource-pack=true
```

- `require-resource-pack=true` boots players who refuse the pack. Recommended for a survival server where custom items meaningfully affect gameplay (otherwise they'd see Life Orbs as plain hearts of the sea and not know what's going on).
- `resource-pack-prompt` is the message shown in the accept-pack dialog. Supports MiniMessage in Paper.
- `resource-pack-sha1` is technically optional but strongly recommended — without it, Paper can't tell when the zip has changed and clients will keep using the cached old version.

## Updating the pack

1. Edit files under `docs/resource-pack/assets/` here in `diverge-core`.
2. Clone (or pull) the [pack repo](https://github.com/Cooper-AmplioDev/diverge-resource-pack) somewhere — call that path `$PACK`.
3. `cp -r docs/resource-pack/* "$PACK/"` (this copies the new sources over).
4. From `$PACK`, build a versioned zip and tag a release:
   ```bash
   cd "$PACK"
   git add -A && git commit -m "Pack vX.Y.Z: <what changed>"
   git push
   VERSION=vX.Y.Z
   zip -r "diverge-resource-pack-$VERSION.zip" pack.mcmeta assets/ -x "*.DS_Store"
   shasum -a 1 "diverge-resource-pack-$VERSION.zip"
   gh release create "$VERSION" "diverge-resource-pack-$VERSION.zip" \
     --title "$VERSION" --generate-notes
   ```
5. Copy the new SHA-1 + asset URL into `server.properties`.
6. Restart Paper (or `/reload confirm`, but a restart is safer).

Clients will see the updated pack on their next reconnect. Because the asset filename is *versioned*, the old release URL stays valid forever — useful if you ever need to roll back.

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
