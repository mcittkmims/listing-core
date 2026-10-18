# Apartment Lens · Core service

The core service is the backend of [Apartment Lens](https://github.com/mcittkmims/apartment-lens). It is a Java application built as a modular monolith, and its REST API is the only way into the system from outside. The [frontend](https://github.com/mcittkmims/listing-frontend) is its only client.

> **Status:** design stage. This README describes the plan; there is no code yet.

## Responsibilities

The core service answers every request from the frontend: map searches with filters, property pages with the ads from every source and their price history, and photo and video searches. It is also the only component that decides whether two ads describe the same property, and it keeps a review queue for the pairs it cannot decide on automatically. User accounts, saved searches and price-drop alerts will be added later.

## Modules

The application is split into modules, each in its own package. Spring Modulith verifies in a test that no module reaches into the internals of another, which keeps the monolith easy to change and makes it possible to move a module into its own service later.

| Module | Responsibility |
|---|---|
| `catalog` | Buildings, properties and price history |
| `matching` | Deciding which ads describe the same property |
| `search` | Map and filter queries |
| `lens` | Accepting photo and video searches and returning their results |
| `review` | Pairs of ads waiting for a manual decision |
| `users`, `alerts` | Accounts, saved searches and notifications (later) |

## Matching

Matching runs twice for every new ad. The first pass runs as soon as the ad's photos are fingerprinted and relies on identical or near-identical photos together with the ad's facts. The second pass runs after the daily batch, when image vectors and same-room checks are available for the new photos.

Candidate ads come from PostgreSQL queries rather than from code that loops over all ads. PostGIS finds ads within about 300 metres with the same room count and an area within 5%, and ads that share a phone hash are added as candidates. pg_trgm scores how similar two addresses or descriptions are. The matching rules combine these facts with the photo evidence from the vision workers and produce one of four decisions: same property, same building, review, or different.

Hard facts override photo evidence. Two ads with different known floors describe different properties even if every photo matches, because developers reuse the same renders for all units in a building. Every decision is stored with its evidence and the version of the rules that made it, and links between ads and properties are closed rather than deleted, so a wrong decision can always be reversed.

## Photo and video search

When the frontend uploads a photo or video, the core service stores the file, creates a search record and adds a job to the queue. The online vision worker processes the job and writes the ranked results back. The frontend polls the search until the results are ready, and the core service returns the matching properties together with the photos or video frames that matched. Searches are rate-limited, because each one uses CPU time on the server.

## Data access

The core service reads the `ingest`, `media` and `social` schemas and writes to `catalog`, `app` and `ops`. The schemas are described in the [main repository](https://github.com/mcittkmims/apartment-lens#database).

## Technology

The service uses Java 21 or later with Spring Boot and Spring Modulith, jOOQ for SQL-heavy queries such as PostGIS and pgvector lookups, and Bucket4j for rate limiting. It runs on the server behind the reverse proxy.
