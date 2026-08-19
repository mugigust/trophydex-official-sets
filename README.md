# trophydex-official-sets

This repository hosts community/official trophy and achievement sets for
[TrophyDex](https://github.com/mugigust/trophydex), a local-first desktop
app for cataloging game trophies and achievements. It's used to import
trophy lists for games that have no native trophy/achievement platform of
their own — for example Nintendo Switch, Nintendo Switch 2, or Wii U
titles.

TrophyDex reads this repository directly over
`raw.githubusercontent.com` — there's no build step, no API key, and no
rate limit for the app to worry about. Any commit pushed to `main` is
live in the app within a few minutes (raw.githubusercontent.com's CDN
cache).

This repo is pre-configured as the default "Custom Cloud" source in
TrophyDex, but anyone can point the app at their own fork or their own
repo using the same layout — see the app's Connections screen → Custom
Cloud.

## How the app browses this repo

Browsing is a two-step drill-down:

1. The app fetches [`info.json`](#infojson) to list the **platforms**
   this repo publishes sets for (e.g. "Nintendo Switch 2", "Wii U").
2. Once the user picks a platform, the app fetches that platform's own
   `manifest.json` to list the **games** available for it.
3. Once the user picks a game, the app fetches that game's own
   `manifest.json` — the actual trophy list — and imports it.

Nothing above the leaf manifest needs to declare a file path: every file
is located by a deterministic path built from `platform_id` and
`game_id` alone.

## Repository layout

```
/info.json
/sets/<platform_id>/manifest.json
/sets/<platform_id>/<game_id>/manifest.json
```

Example:

```
info.json
sets/
  switch2/
    manifest.json
    mario-kart-world/
      manifest.json
    zelda-echoes/
      manifest.json
  wiiu/
    manifest.json
    breath-of-the-wild/
      manifest.json
```

Every manifest carries a `schemaVersion`. The app **refuses** (rather
than best-effort-parses) any version it doesn't recognize, so a future
format change fails cleanly instead of silently importing malformed
data. The current version is `1`.

Images (`cover_art_url`, `icon`) are always plain external URLs, never
bundled into the app. The simplest approach is committing image files
into this repo (e.g. under `covers/` and `icons/<game_id>/`) and
referencing them via their own `raw.githubusercontent.com` URL — see the
examples below.

---

## 1. `info.json` (repo root)

Lists every platform this repo publishes sets for.

```json
{
  "schemaVersion": 1,
  "platforms": [
    {
      "platform_id": "switch2",
      "platform_title": "Nintendo Switch 2",
      "cover_art_url": "https://raw.githubusercontent.com/mugigust/trophydex-official-sets/main/covers/switch2.jpg"
    }
  ]
}
```

| Field | Required | Description |
|---|---|---|
| `platform_id` | yes | Slug used to locate this platform's folder — must exactly match the directory name under `sets/`. Must be unique within this repo. Treat it like a URL slug: once published, don't rename it — the app uses it (combined with the game id) to track which games have already been imported and to re-sync them later. |
| `platform_title` | yes | Display name shown in the app's platform picker (e.g. "Nintendo Switch 2"). Safe to edit any time — it's re-read on every import/sync. |
| `cover_art_url` | no | Thumbnail shown next to the platform in the picker. Omit if you don't have one; the app falls back to a placeholder icon. |

---

## 2. `sets/<platform_id>/manifest.json` (one per platform)

Lists every game available for that platform.

```json
{
  "schemaVersion": 1,
  "games": [
    {
      "game_id": "mario-kart-world",
      "game_title": "Mario Kart World",
      "cover_art_url": "https://raw.githubusercontent.com/mugigust/trophydex-official-sets/main/covers/mario-kart-world.jpg"
    }
  ]
}
```

| Field | Required | Description |
|---|---|---|
| `game_id` | yes | Slug used to locate this game's manifest — must exactly match the directory name under `sets/<platform_id>/`. Must be unique within the platform. Same "don't rename once published" rule as `platform_id` — it's part of what identifies an already-imported game. |
| `game_title` | yes | Display name shown in the app's game picker. Safe to edit any time. |
| `cover_art_url` | no | Thumbnail shown next to the game in the picker. |

---

## 3. `sets/<platform_id>/<game_id>/manifest.json` (one per game)

The actual trophy/achievement list — this is what gets imported. Note
this file predates the two above and keeps its own, slightly different
field naming (camelCase, not snake_case) — don't copy the `_url`
convention from `info.json`/the platform manifest into this file.

```json
{
  "schemaVersion": 1,
  "title": "Mario Kart World",
  "coverUrl": "https://raw.githubusercontent.com/mugigust/trophydex-official-sets/main/covers/mario-kart-world.jpg",
  "trophies": [
    {
      "id": "first-place",
      "name": "Podium Finish",
      "description": "Finish in 1st place in any race",
      "tier": "bronze",
      "group": null,
      "icon": "https://raw.githubusercontent.com/mugigust/trophydex-official-sets/main/icons/mario-kart-world/first-place.png",
      "order": 1
    }
  ]
}
```

**Top level:**

| Field | Required | Description |
|---|---|---|
| `title` | yes | The game's title, shown once imported into the user's Dex. |
| `coverUrl` | no | The game's cover art, shown once imported. |
| `trophies` | yes | Array of trophy/achievement objects, described below. Can be empty while you're still working on a set. |

**Each entry in `trophies`:**

| Field | Required | Description |
|---|---|---|
| `id` | yes | Stable identifier for this trophy, unique within the game. Never rename once published — it's how the app tells "this trophy already exists" from "this is a new trophy" on re-sync. |
| `name` | yes | The trophy's display name. |
| `description` | no | What the player needs to do to earn it. |
| `tier` | no | One of `"bronze"`, `"silver"`, `"gold"`, `"platinum"`. Anything else (including omitted) is treated as no tier. |
| `group` | no | Groups trophies together (e.g. a DLC pack's name). Use `null` or omit for the main/base-game list. |
| `icon` | no | External URL to the trophy's icon. |
| `order` | no | Integer controlling display order (lower first). Trophies without an `order` keep their position in the array. |

Importing is idempotent and safe to re-run (the app's "Sync trophies"
does exactly this): re-importing a game never clears a trophy the user
already marked as earned, and never duplicates trophies — it matches on
each trophy's `id`.

---

## Adding a new platform

1. Add an entry to `info.json`.
2. Create `sets/<platform_id>/manifest.json` (can start with an empty
   `"games": []` array).
3. Commit and push.

## Adding a new game to an existing platform

1. Create `sets/<platform_id>/<game_id>/manifest.json` with the game's
   trophy list.
2. Add a matching entry to `sets/<platform_id>/manifest.json`.
3. Commit both in the same push — if the game entry is added before its
   manifest exists, it'll show up in the app's picker but fail to
   import until the manifest is actually there.

## Updating an already-published set

Just edit the game's `manifest.json` (add/change trophies, fix a typo,
update the cover) and push. Anyone who already imported that game can
re-sync from inside the app (DexEntryDetail → Sync trophies) to pull in
the changes — nothing needs to be reinstalled or re-added.