# Apartment Lens · Core service

The backend of [Apartment Lens](https://github.com/mcittkmims/apartment-lens): one Java app with strict internal modules. Its REST API is the only way into the system from outside; the [frontend](https://github.com/mcittkmims/listing-frontend) is its client.

> **Status:** design stage. This README describes the plan; there is no code yet.

## What it does

- Serves the REST API the frontend uses: map search, filters, property pages with every source's ad and the price history.
- Decides which ads are the same flat.
- Accepts photo and video searches and returns the results.
- Holds the review queue for unclear matches.
- Later: user accounts, saved searches and price-drop alerts.

## Modules

Each module is its own package. Spring Modulith tests fail if one module reaches into another's internals.

| Module | Job |
|---|---|
| `catalog` | Buildings, flats, price history |
| `matching` | Same-flat decisions |
| `search` | Map and filter queries |
| `lens` | Accepts photo/video searches, queues them, returns results |
| `review` | Unclear pairs waiting for a manual decision |
| `users`, `alerts` | Accounts, saved searches, notifications (later) |

## How matching works

- **Runs twice per new ad:** right away, using identical or near-identical photos plus facts, and again after the daily batch vision worker adds vectors and same-room check scores.
- **Candidates come from Postgres queries:** ads within ~300 m (PostGIS), same room count, area within 5%, or the same phone hash. Address and description similarity come from pg_trgm.
- **Rules decide:** same flat, same building, review, or different. Facts overrule photos: two known, different floors are two different flats.
- **Every decision is reversible:** links have `valid_from`/`valid_to`, and the evidence is stored with a model version.

## How a photo or video search works

1. The upload is saved and a `lens_photo` or `lens_video` job goes into `ops.job`.
2. The online vision worker picks it up and writes the ranked results.
3. The API returns the top flats, each with its matching photos or frames as proof.

Searches are rate-limited, because each one costs CPU time on the server.

## Database

- **Reads:** `ingest`, `media`, `social`.
- **Writes:** `catalog`, `app`, `ops`.

See [database](https://github.com/mcittkmims/apartment-lens/tree/main/database).

## Runs on

The server, behind the reverse proxy.

## Tech

Java 21 or 25, Spring Boot, Spring Modulith, jOOQ, Bucket4j (rate limits).
