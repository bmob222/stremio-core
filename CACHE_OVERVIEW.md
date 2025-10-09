# Stremio Core - Caching Overview for RealDebrid/Debrid Services

## Quick Reference

This is a simplified overview of how Stremio Core caches links from services like RealDebrid and Premiumize. For detailed information, see [CACHE_DOCUMENTATION.md](./CACHE_DOCUMENTATION.md).

## Caching Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Request for Stream                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   Check In-Memory Cache               │
        │   (ResourceLoadable)                  │
        │                                       │
        │   - Request matches?                  │
        │   - Content exists?                   │
        └───────────┬───────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼ Cache HIT             ▼ Cache MISS
┌───────────────┐       ┌──────────────────────┐
│ Return Cached │       │  Fetch from Addon    │
│ Streams       │       │  (HTTP Request)      │
│               │       │                      │
│ - URLs        │       │  Example:            │
│ - Auth Headers│       │  RealDebrid API      │
│ - Metadata    │       │  Premiumize API      │
└───────────────┘       └──────────┬───────────┘
                                   │
                                   ▼
                        ┌─────────────────────────┐
                        │ Addon Response          │
                        │ (ResourceResponseCache) │
                        │                         │
                        │ {                       │
                        │   streams: [...],       │
                        │   cacheMaxAge: 3600,    │
                        │   staleRevalidate: ..., │
                        │   staleError: ...       │
                        │ }                       │
                        └──────────┬──────────────┘
                                   │
                                   ▼
                        ┌─────────────────────────┐
                        │ Store in Memory Cache   │
                        │ (ResourceLoadable)      │
                        └──────────┬──────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│              User Selects Stream for Playback                │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ Persist to Storage        │
                │ (StreamsBucket)           │
                │                           │
                │ - Selected stream         │
                │ - Auth headers            │
                │ - Playback state          │
                │ - Survives app restart    │
                └───────────────────────────┘
```

## Cache Layers

### Layer 1: HTTP Cache Control (Addon-Defined)
- **File:** `src/types/addon/response.rs`
- **What:** Cache directives from the addon (RealDebrid, etc.)
- **Duration:** Defined by addon (typically 1 hour for premium links)
- **Contains:** `cacheMaxAge`, `staleRevalidate`, `staleError`

### Layer 2: In-Memory Cache (Application)
- **File:** `src/models/common/resource_loadable.rs`
- **What:** Cached addon responses in application memory
- **Duration:** Until app restart or logout
- **Contains:** Full stream objects with URLs and auth tokens

### Layer 3: Persistent Storage (User-Specific)
- **File:** `src/models/ctx/update_streams.rs`
- **What:** User's selected streams saved to disk
- **Duration:** Until logout or manual clear
- **Contains:** Selected stream, playback position, metadata

## Example: RealDebrid Stream Caching

```json
// 1. Addon Response (from RealDebrid)
{
  "streams": [
    {
      "url": "https://realdebrid.com/dl/ABC123",
      "name": "RealDebrid 1080p",
      "behaviorHints": {
        "proxyHeaders": {
          "request": {
            "Authorization": "Bearer rd_token_xyz"
          }
        }
      }
    }
  ],
  "cacheMaxAge": 3600  // Valid for 1 hour
}

// 2. In-Memory Cache (ResourceLoadable)
ResourceLoadable {
  request: ResourceRequest { base: "https://realdebrid-addon.com", ... },
  content: Ready(streams)  // Cached response
}

// 3. Persistent Storage (StreamsBucket)
StreamsItem {
  stream: Stream { url: "...", proxy_headers: { ... } },
  state: { time_watched: 1234, ... },
  mtime: 2024-01-15T10:30:00Z
}
```

## Key Benefits

✅ **Fast Response** - In-memory cache returns instantly  
✅ **Reduced API Calls** - Respects addon's cache duration  
✅ **Secure Tokens** - Auth headers cached with streams  
✅ **Persistence** - Selected streams survive restarts  
✅ **Graceful Degradation** - Stale content usable if API fails

## Cache Invalidation

- **Logout:** All caches cleared
- **Force Refresh:** `force: true` bypasses cache
- **Expiry:** After `cacheMaxAge` seconds
- **App Restart:** In-memory cache cleared, storage persists

## Related Files

| Component | File |
|-----------|------|
| HTTP Cache | `src/types/addon/response.rs` |
| Memory Cache | `src/models/common/resource_loadable.rs` |
| Persistent Cache | `src/models/ctx/update_streams.rs` |
| Stream Types | `src/types/resource/stream.rs` |
| Proxy Headers | `src/types/resource/stream.rs` (StreamProxyHeaders) |

For detailed explanations, code examples, and implementation details, see [CACHE_DOCUMENTATION.md](./CACHE_DOCUMENTATION.md).
