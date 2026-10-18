# Apartment Lens · Core service

The backend of [Apartment Lens](https://github.com/mcittkmims/apartment-lens). A Java modular monolith whose REST API serves the [frontend](https://github.com/mcittkmims/listing-frontend), and the only component that decides which ads describe the same property.

> **Status:** design stage. There is no code yet.

## Modules

| Module | Responsibility |
|---|---|
| `catalog` | Buildings, properties and price history |
| `matching` | Deciding which ads describe the same property |
| `search` | Map and filter queries |
| `lens` | Photo and video searches |
| `review` | Uncertain matches waiting for a manual decision |
| `users`, `alerts` | Accounts, saved searches and alerts (later) |

Spring Modulith checks that modules use only each other's public API, so any module can later move into its own service.

## Matching

Each new ad is compared with nearby properties: same room count, similar area, same floor, similar address, same phone hash, and matching photos. The result is one of four decisions: same property, same building, review, or different. Hard facts beat photos, so two ads with different floors are never merged. Matching runs once when the ad arrives, and again after the daily batch adds stronger photo evidence.

## Search

An uploaded photo or video is saved and queued for the online vision worker, and the API returns the ranked properties once the worker has finished. Searches are rate-limited because each one costs CPU time.

## Data and technology

Reads `ingest`, `media` and `social`; writes `catalog`, `app` and `ops` ([schemas](https://github.com/mcittkmims/apartment-lens/blob/main/database/README.md)). Built with Java 21+, Spring Boot, Spring Modulith, jOOQ and Bucket4j.
