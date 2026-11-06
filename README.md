# MochiDB

A local SQLite dataset of AniList anime metadata, Anime Skip timestamps, and
downloaded cover/banner images. The database is designed for fast lookups by
**AniList ID** and for sharing as an open dataset.

- Database: `anime_timestamps.db` (schema version **2**)
- Images: `images/` (`{anilist_id}_cover.*`, `{anilist_id}_banner.*`)
- Backups: `backups/` (pre-restructure copies of the original DB)
- Rebuild script: `scripts/restructure_v2.py`

## Layout

```
MochiDB/
  anime_timestamps.db        # restructured SQLite database (v2)
  images/                    # 28,366 downloaded cover/banner files
  backups/                   # original + pre-restructure database copies
  scripts/
    restructure_v2.py        # rebuilds the v2 DB from a legacy copy
  README.md
```

Image files are **not** embedded in the database. `anime_images.image_path` is
relative to this folder (for example `images/16498_cover.jpg`) and
`anime_images.image_url` points at the original AniList CDN URL.

## Quick start

```sql
-- Everything for one AniList ID (metadata, images, counts)
SELECT * FROM v_anime WHERE anilist_id = 16498;

-- All skip timestamps for one AniList ID
SELECT * FROM v_anime_timestamps
WHERE anilist_id = 16498
ORDER BY episode_number_value, time_seconds;

-- All timestamps for one episode
SELECT * FROM v_episode_timestamps
WHERE episode_id = '...'
ORDER BY time_seconds;

-- Resolve a MAL ID to the local record
SELECT * FROM anime WHERE mal_id = 16498;

-- Only titles that actually have timestamps
SELECT * FROM anime WHERE skip_status = 'matched';
```

`time_seconds` is seconds from the start of the episode. `timestamp_type` is one
of the names in `timestamp_types` (Intro, Credits, Canon, Filler, Recap, etc.).

## Schema

### Core tables

| Table | Primary key | Purpose |
| --- | --- | --- |
| `anime` | `anilist_id` | AniList catalogue metadata + scraper status |
| `anime_skip_shows` | `anime_skip_id` | Anime Skip shows |
| `anime_show_matches` | (`anilist_id`, `anime_skip_id`) | Many-to-many AniList ↔ Anime Skip mapping |
| `episodes` | `episode_id` | Anime Skip episodes (`episode_number_value` is numeric) |
| `timestamps` | `timestamp_id` | Skip timestamps, normalized to `timestamp_types` |
| `anime_images` | (`anilist_id`, `image_type`) | Cover/banner download state and paths |

### Metadata tables

| Table | Purpose |
| --- | --- |
| `schema_version` | Applied schema migrations |
| `dataset_meta` | Dataset name, sources, counts, generation time, notes |
| `timestamp_types` | Normalized timestamp type names |
| `scraper_state` | Internal scraper checkpoint state |

### Views

| View | Use |
| --- | --- |
| `v_anime` | Anime metadata + cover/banner + show/episode/timestamp counts |
| `v_anime_images` | One row per AniList ID with cover/banner paths and URLs |
| `v_anime_timestamps` | Flat AniList ID → show → episode → timestamp rows |
| `v_episode_timestamps` | Flat episode → timestamp rows |

### Indexes

Lookups by `anilist_id` use the primary key. Additional indexes cover `mal_id`,
titles (case-insensitive), show names, the scraper queue, episodes per show,
timestamps per episode, timestamp types, and image status.

## Data notes

The v2 rebuild cleaned up the legacy database:

- Removed **164** timestamps with negative `time_seconds`.
- Removed **29** exact duplicate timestamps (same episode/time/type).
- Normalized **7** literal `"null"` episode numbers to `NULL`.
- Reclassified **20** poisoned `error` rows (`Row not found`) as `no_match`.
- Removed **4** episodes left with no timestamps and reclassified **1**
  `matched` title as `no_timestamps`.
- Added `episode_number_value` (numeric) and `timestamps.type_id` lookup.
- Enabled `STRICT` tables, `CHECK` constraints, and foreign keys.

Current totals:

| Entity | Count |
| --- | --- |
| Anime | 22,491 |
| Anime Skip shows | 666 |
| Show matches | 1,299 |
| Episodes | 7,449 |
| Timestamps | 42,629 |
| Image rows | 44,982 (28,366 files on disk) |

### Scraper compatibility

The legacy table and column names are preserved, so `cent-anilist-scraper` can
still read and write the database. New rows are kept normalized by triggers
(`episode_number_value`, `timestamp_types`). The scraper opens the DB in WAL
mode, which is fine; the distributed file itself is in single-file `DELETE`
journal mode.

## Provenance & attribution

- Anime metadata: [AniList](https://anilist.co)
- Timestamp data: [Anime Skip](https://anime-skip.com)
- Images: downloaded from AniList's CDN and referenced by original URL

If you publish this dataset, verify and respect each source's terms of service
and attribution requirements. The `dataset_meta` table records these sources.
Choose a license for the database structure and scripts separately from the
upstream data.

## Rebuilding

The restructure script converts a legacy database into the v2 schema without
modifying the source:

```powershell
python scripts\restructure_v2.py
# writes anime_timestamps.v2.db
```

Options:

```text
--source PATH   legacy database (default: anime_timestamps.db)
--output PATH   destination (default: anime_timestamps.v2.db)
--force         overwrite an existing output file
```
