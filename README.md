# Apartment Lens · Core app

The monolith of [Apartment Lens](https://github.com/mcittkmims/apartment-lens): one Java app with strict internal modules. It is the only part of the system the outside world talks to.

> **Status:** design stage. This README describes the plan; there is no code yet.

## What it does

- Serves the website and API: map, filters, and a page per flat showing every site's ad and the price history.
- Decides which ads are the same flat.
- Takes photo and video searches and shows the results.
- Runs the review screen for unclear matches.
- Later: user accounts, saved searches and price-drop alerts.

## Modules

Each module is its own package. Spring Modulith tests fail if one module reaches into another's internals.

| Module | Job |
|---|---|
| `catalog` | Buildings, flats, price history |
| `matching` | Same-flat decisions |
| `search` | Map and filters |
| `lens` | Accepts photo/video searches, queues them, shows results |
| `review` | Screen for confirming unclear pairs by hand |
| `users`, `alerts` | Accounts, saved searches, notifications (later) |

## How matching works

- **Runs twice per new ad:** right away, using identical or near-identical photos plus facts, and again after the daily MacBook batch adds vectors and LightGlue scores.
- **Candidates come from Postgres queries:** ads within ~300 m (PostGIS), same room count, area within 5%, or the same phone hash. Address and description similarity come from pg_trgm.
- **Rules decide:** same flat, same building, review, or different. Facts overrule photos: two known, different floors are two different flats.
- **Every decision is reversible:** links have `valid_from`/`valid_to`, and the evidence is stored with a model version.

## How a photo or video search works

1. The upload is saved and a `lens_photo` or `lens_video` job goes into `ops.job`.
2. The vision service on the laptop picks it up and writes the ranked results.
3. The app shows the top flats, each with its matching photos or frames as proof.

Searches are rate-limited, because each one costs CPU time on the laptop.

## Database

- **Reads:** `ingest`, `media`, `social`.
- **Writes:** `catalog`, `app`, `ops`.

See [database](https://github.com/mcittkmims/apartment-lens/tree/main/database).

## Runs on

The old laptop, behind Caddy and Cloudflare Tunnel.

## Tech

- **Backend:** Java 21 or 25, Spring Boot, Spring Modulith, jOOQ, Bucket4j (rate limits).
- **Frontend:** TypeScript, React, MapLibre GL JS with OpenStreetMap tiles. It will live in `web/` in this repo.
