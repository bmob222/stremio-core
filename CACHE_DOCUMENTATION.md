# Stremio Core - Link Caching Mechanism

This document explains how Stremio Core caches links for services like RealDebrid, Premiumize, and other streaming/debrid services.

## Overview

Stremio Core implements a multi-layered caching system for addon responses, including stream links from services like RealDebrid. The caching happens at two main levels:

1. **HTTP Response Cache Headers** - Addon-defined cache control
2. **In-Memory Resource Caching** - Application-level request/response caching

## 1. HTTP Cache Control (ResourceResponseCache)

### Location
`src/types/addon/response.rs`

### Structure

```rust
pub struct ResourceResponseCache {
    /// (in seconds) Cache-Control header: max-age=$cacheMaxAge
    pub cache_max_age: Option<u64>,
    
    /// (in seconds) Cache-Control header: stale-while-revalidate=$staleRevalidate
    pub stale_revalidate: Option<u64>,
    
    /// (in seconds) Cache-Control header: stale-if-error=$staleError
    pub stale_error: Option<u64>,
    
    #[serde(flatten)]
    pub resource: ResourceResponse,
}
```

### How It Works

When an addon (like RealDebrid) returns stream links, it can include cache control information:

```json
{
  "streams": [
    {
      "url": "https://example.premiumize.me/stream/video.mkv",
      "name": "Premium Link",
      "behaviorHints": {
        "notWebReady": true,
        "proxyHeaders": {
          "request": {
            "Authorization": "Bearer TOKEN"
          }
        }
      }
    }
  ],
  "cacheMaxAge": 3600,
  "staleRevalidate": 14400,
  "staleError": 604800
}
```

**Cache Parameters:**
- `cacheMaxAge` (3600s = 1 hour): How long the response is considered fresh
- `staleRevalidate` (14400s = 4 hours): How long a stale response can be used while revalidating in the background
- `staleError` (604800s = 7 days): How long a stale response can be used if revalidation fails

This follows standard HTTP Cache-Control semantics, allowing addons to control how long their links remain valid.

## 2. In-Memory Resource Caching (ResourceLoadable)

### Location
`src/models/common/resource_loadable.rs`

### Structure

```rust
pub struct ResourceLoadable<T> {
    pub request: ResourceRequest,
    pub content: Option<Loadable<T, ResourceError>>,
}
```

### How It Works

The `resources_update` function implements request-level caching:

```rust
fn resources_update<E, T>(
    resources: &mut Vec<ResourceLoadable<T>>,
    action: ResourcesAction,
) -> Effects {
    match action {
        ResourcesAction::ResourcesRequested { request, addons, force } => {
            let (next_resources, effects) = request
                .plan(addons)
                .into_iter()
                .map(|(_, request)| {
                    resources
                        .iter()
                        // Check if we've seen this request before (caching)
                        .find(|resource| {
                            resource.request == request 
                            && resource.content.is_some() 
                            && !force
                        })
                        .map(|resource| (resource.to_owned(), None))
                        .unwrap_or_else(|| {
                            // Make new request if not cached
                            // ...
                        })
                })
                // ...
        }
    }
}
```

**Key Points:**
- Requests are matched by their `ResourceRequest` (includes addon URL and resource path)
- If a matching request is found with existing content, it's reused (no new HTTP request)
- The `force` parameter allows bypassing the cache
- Cached responses stay in memory until the application state changes

## 3. Stream Caching Flow for RealDebrid/Debrid Services

### Step-by-Step Process

1. **User Requests Streams**
   - User selects a video to watch
   - Application creates a `ResourceRequest` for stream addons

2. **Check In-Memory Cache**
   ```rust
   resources.iter().find(|resource| {
       resource.request == request && resource.content.is_some()
   })
   ```
   - If found: Return cached streams immediately
   - If not found: Proceed to fetch

3. **Fetch from Addon**
   - HTTP request sent to RealDebrid/debrid addon
   - Addon transport: `src/addon_transport/http_transport/http_transport.rs`
   - Request format: `/{resource}/{type}/{id}.json`

4. **Addon Response with Cache Info**
   - Addon returns `ResourceResponseCache` with:
     - Stream URLs (potentially with auth tokens in headers)
     - Cache control directives (`cacheMaxAge`, etc.)
   
5. **Store in Memory Cache**
   - Response stored in `ResourceLoadable`
   - Keyed by `ResourceRequest`
   - Subsequent requests use cached data

6. **Cache Invalidation**
   - Logout: All cached resources cleared
   - Manual force refresh: `force: true` parameter
   - Application restart: Cache rebuilt from scratch

## 4. Stream Link Persistence (StreamsBucket)

### Location
`src/models/ctx/update_streams.rs`

### Structure

```rust
pub struct StreamsItem {
    pub stream: Stream,
    pub r#type: String,
    pub meta_id: String,
    pub video_id: String,
    pub meta_transport_url: Url,
    pub stream_transport_url: Url,
    pub state: Option<StreamItemState>,
    pub mtime: DateTime<Utc>,
}
```

### Persistent Storage

Selected streams are persisted to storage:

```rust
fn push_streams_to_storage<E: Env + 'static>(streams: &StreamsBucket) -> Effect {
    EffectFuture::Sequential(
        E::set_storage(STREAMS_STORAGE_KEY, Some(&streams))
            .map(|result| match result {
                Ok(_) => Msg::Event(Event::StreamsPushedToStorage { uid }),
                Err(error) => Msg::Event(Event::Error { error, source })
            })
            .boxed_env(),
    )
    .into()
}
```

**What's Stored:**
- Selected stream details (including URL and auth headers)
- Playback state (time watched, position)
- Metadata association (which video, which addon)
- Last modification time

**When Updated:**
- `Msg::Internal(Internal::StreamLoaded)`: User selects a stream
- `Msg::Internal(Internal::StreamStateChanged)`: Playback position updates
- Storage persists across app restarts

## 5. Proxy Headers for Debrid Services

### Location
`src/types/resource/stream.rs`

### Structure

```rust
pub struct StreamProxyHeaders {
    #[serde(default, skip_serializing_if = "HashMap::is_empty")]
    pub request: HashMap<String, String>,
    #[serde(default, skip_serializing_if = "HashMap::is_empty")]
    pub response: HashMap<String, String>,
}

pub struct StreamBehaviorHints {
    pub proxy_headers: Option<StreamProxyHeaders>,
    // ... other fields
}
```

### How RealDebrid/Debrid Services Use This

Debrid services often require authentication headers. These are cached with the stream:

```json
{
  "url": "https://webdav.premiumize.me/video.mkv",
  "name": "Premiumize Link",
  "behaviorHints": {
    "proxyHeaders": {
      "request": {
        "Authorization": "Basic BASE64_TOKEN"
      }
    }
  }
}
```

When the streaming server proxies the request:

```rust
pub fn streaming_url(&self, streaming_server_url: Option<&Url>) -> Option<Url> {
    match (&self.source, streaming_server_url) {
        (StreamSource::Url { url }, Some(streaming_server_url))
            if self.behavior_hints.proxy_headers.is_some() => {
            // Build proxied URL with headers encoded in query params
            let mut streaming_url = streaming_server_url.to_owned();
            let mut proxy_query = form_urlencoded::Serializer::new(String::new());
            
            proxy_query.append_pair("d", origin.as_str());
            proxy_query.extend_pairs(
                request.iter().map(|header| ("h", format!("{}:{}", header.0, header.1)))
            );
            
            streaming_url.set_path(&format!("proxy/{query}/{url_path}"));
            Some(streaming_url)
        }
        // ...
    }
}
```

**Result:** 
- Auth headers are preserved in the cached stream
- Streaming server includes them when fetching from debrid service
- Headers remain valid for the cache duration

## 6. Cache Lifecycle Example: RealDebrid Stream

### Scenario: User watches a movie through RealDebrid addon

```
1. Initial Request (t=0)
   User: Select movie -> Request streams
   Cache: MISS
   Action: Fetch from RealDebrid addon
   
2. RealDebrid Response
   {
     "streams": [{
       "url": "https://realdebrid.com/dl/ABC123",
       "behaviorHints": {
         "proxyHeaders": {
           "request": {"Authorization": "Bearer TOKEN"}
         }
       }
     }],
     "cacheMaxAge": 3600  // 1 hour
   }
   
3. Store in Cache (t=0)
   ResourceLoadable: { request, content: Ready(streams) }
   HTTP Cache: max-age=3600
   
4. User Selects Stream (t=0)
   StreamsBucket: Store stream with auth headers
   Persistent Storage: Save to disk
   
5. Second Request (t=30min)
   User: Navigate away and back
   Cache: HIT
   Action: Return cached streams (no HTTP request)
   Duration: Still fresh (30min < 60min)
   
6. Third Request (t=2hr)
   User: Return after cache expiry
   Cache: STALE
   Action: Fetch fresh streams from RealDebrid
   Result: New token, updated cache
   
7. Logout
   Cache: CLEARED
   Storage: Streams removed (tied to user UID)
```

## 7. Configuration: Streaming Server Cache

### Location
`src/types/streaming_server/settings.rs`

```rust
pub struct Settings {
    pub cache_root: String,
    pub cache_size: Option<f64>,
    // ... other fields
}
```

This configures the streaming server's disk cache for torrents and downloaded content, separate from the link caching described above.

## Summary

Stremio Core's caching for RealDebrid and similar services operates at multiple levels:

1. **Addon-Level**: HTTP cache headers control how long links are valid
2. **Application-Level**: In-memory cache prevents redundant addon requests  
3. **User-Level**: Selected streams with auth headers persist across sessions
4. **Server-Level**: Streaming server caches actual media content

This multi-layered approach ensures:
- ✅ Fast response times (in-memory cache)
- ✅ Reduced API calls to debrid services (HTTP cache)
- ✅ Secure token handling (proxy headers)
- ✅ Persistence across sessions (storage)
- ✅ Proper invalidation (logout, expiry)

## Implementation Files Reference

| Component | File Path |
|-----------|-----------|
| HTTP Cache Control | `src/types/addon/response.rs` |
| Resource Caching | `src/models/common/resource_loadable.rs` |
| Stream Storage | `src/models/ctx/update_streams.rs` |
| Stream Types | `src/types/resource/stream.rs` |
| Addon Transport | `src/addon_transport/http_transport/http_transport.rs` |
| Streaming Server | `src/models/streaming_server.rs` |
