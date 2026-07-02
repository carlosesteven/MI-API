# Architecture & Endpoints

## Architecture

The entire API lives in `api.py`. It proxies two upstream sources:

### 1. AniList GraphQL (`https://graphql.anilist.co`)

All anime metadata: search, filter, collections, info, characters, relations, recommendations.

### 2. MI Pipe

Episode lists and M3U8 streaming URLs.

**Protocol:** every request is base64-encoded JSON sent as `?e=<payload>`. Every response is base64 + gzip compressed JSON. `_encode_pipe_request()` and `_decode_pipe_response()` handle this.

**Cloudflare bypass:** the pipe is protected by Cloudflare bot detection that checks the TLS fingerprint of the client (not just headers). Standard Python HTTP clients (`httpx`, `requests`) are blocked because their TLS handshake doesn't match a real browser. The API uses [`curl_cffi`](https://github.com/yifeikong/curl_cffi) with `impersonate="chrome110"` to replicate Chrome's exact TLS fingerprint — no cookies or tokens required.

**Session management:** a single global `pipe_session` is kept alive across requests to reuse the TCP/TLS connection (~100ms per request after the first ~300ms handshake). If the session fails, it is automatically replaced and the request is retried. If the retry also fails, the endpoint returns `503 Pipe unavailable`.

### ID encoding

Episode IDs returned by the pipe are base64-encoded. `_translate_id()` decodes a single ID; `_deep_translate()` recursively walks the full response and decodes all IDs.

---

## Endpoints

### Search & Discovery

| Method | Path           | Description                                                       |
| ------ | -------------- | ----------------------------------------------------------------- |
| GET    | `/search`      | Search anime by name. Params: `query`, `page`, `per_page`         |
| GET    | `/suggestions` | Lightweight autocomplete. Params: `query`                         |
| GET    | `/spotlight`   | Top 10 trending + popular                                         |
| GET    | `/filter`      | Advanced filter by genre, tag, year, season, format, status, sort |

### Collections

| Method | Path               | Description                                                          |
| ------ | ------------------ | -------------------------------------------------------------------- |
| GET    | `/trending`        | Currently trending. Params: `page`, `per_page`                       |
| GET    | `/popular`         | Most popular of all time. Params: `page`, `per_page`                 |
| GET    | `/upcoming`        | Not yet released. Params: `page`, `per_page`                         |
| GET    | `/recent`          | Currently airing (AniList). Params: `page`, `per_page`               |
| GET    | `/schedule`        | Upcoming airing schedule with timestamps. Params: `page`, `per_page` |
| GET    | `/recent-episodes` | Recently aired episodes from MI pipe (cached in Redis)               |

### Anime Details

| Method | Path                          | Description                                |
| ------ | ----------------------------- | ------------------------------------------ |
| GET    | `/info/{anilist_id}`          | Full anime page — all AniList fields       |
| GET    | `/anime/{id}/characters`      | Paginated character list with voice actors |
| GET    | `/anime/{id}/relations`       | Related media (sequels, prequels, etc.)    |
| GET    | `/anime/{id}/recommendations` | Community recommendations                  |

### Streaming

| Method | Path                                              | Description                                                                             |
| ------ | ------------------------------------------------- | --------------------------------------------------------------------------------------- |
| GET    | `/episodes/{anilist_id}`                          | Episode list per provider (kiwi, hop, ally, etc.) organized by sub/dub                  |
| GET    | `/watch/{provider}/{anilistId}/{category}/{slug}` | Streaming sources via slug (recommended)                                                |
| GET    | `/sources`                                        | Streaming sources via explicit params: `episodeId`, `provider`, `anilistId`, `category` |
