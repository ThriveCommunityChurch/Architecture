# Integration Matrix

This document shows how components in the Thrive system communicate with each other. Use this when you're changing an endpoint, modifying authentication, or adding a new data flow.

## Format

Each table shows:
- **Source** — Component making the request
- **Destination** — Where the request goes
- **Protocol** — How they communicate (REST, GraphQL, direct DB, event/queue, etc.)
- **Auth** — How the request is authenticated
- **Key Operations** — Examples of actual operations
- **Notes** — Any special conditions or gotchas

---

## Admin Tool ↔ Global API

| Field | Value |
|-------|-------|
| **Source** | Admin Tool (Angular, browser) |
| **Destination** | Global API (C#/.NET) |
| **Protocol** | HTTPS REST JSON |
| **Base URL (dev)** | `http://localhost:8080` |
| **Base URL (prod)** | `https://api.thrive-fl.org` (or equivalent App Runner URL) |
| **Auth** | JWT (Bearer token in Authorization header) |
| **Token Storage** | MongoDB `RefreshTokens` collection (server-side) |
| **Refresh Flow** | Use `/auth/refresh` endpoint with refresh token |

### Operations

| Operation | Method | Endpoint | Auth | Purpose |
|-----------|--------|----------|------|---------|
| Login | POST | `/auth/login` | None (public) | Get JWT access token + refresh token |
| Refresh Token | POST | `/auth/refresh` | Refresh token (body) | Get new access token |
| Create Series | POST | `/api/series` | JWT | Create new sermon series |
| Update Series | PUT | `/api/series/{id}` | JWT | Change series metadata (title, artwork, etc.) |
| Create Message | POST | `/api/messages` | JWT | Upload new sermon + audio link |
| Update Message | PUT | `/api/messages/{id}` | JWT | Change message metadata or trigger re-processing |
| List Series | GET | `/api/series` | None | Fetch all series (public) |
| Get Config | GET | `/api/config` | None | Fetch church configuration settings |
| Update Config | PUT | `/api/config` | JWT | Change app settings (contact, links, etc.) |
| Upload Audio | POST (multipart) | S3 (via signed URL) | AWS SigV4 | Upload sermon audio file |

### Error Handling

- **401 Unauthorized** → Refresh token and retry
- **403 Forbidden** → JWT valid but user lacks permission
- **409 Conflict** → Duplicate series/message (e.g., same date + speaker)
- **413 Payload Too Large** → Audio file exceeds size limit (check API config)

---

## Mobile App ↔ Global API

| Field | Value |
|-------|-------|
| **Source** | Mobile App (React Native / Expo) |
| **Destination** | Global API |
| **Protocol** | HTTPS REST JSON |
| **Auth** | None required for GET; Bearer JWT for future POST operations |
| **Caching** | React Query (client-side, configurable TTL) |
| **Rate Limiting** | API applies rate limits per IP; mobile app should implement exponential backoff |

### Operations

| Operation | Method | Endpoint | Auth | Purpose |
|-----------|--------|----------|------|---------|
| List Recent Messages | GET | `/api/messages?limit=20&sort=recent` | None | Home screen sermon feed |
| Search Messages | GET | `/api/messages?search=hope&tags=1,2,3` | None | Find sermons by keyword or tag |
| Get Series Details | GET | `/api/series/{id}` | None | Fetch series metadata + all messages |
| Get Message Details | GET | `/api/messages/{id}` | None | Fetch full sermon data (summary, tags, artwork) |
| Get Config | GET | `/api/config` | None | Fetch app-wide settings (contact, give link, etc.) |
| Track Playback | POST | `/api/analytics/play` | Optional JWT | Log that a user started playing |
| Track Completion | POST | `/api/analytics/complete` | Optional JWT | Log that a user finished a sermon |

### Media Access

- **Audio URLs** — API returns S3 pre-signed URLs
- **Download Permission** — Mobile app caches audio locally (optional feature)
- **No Auth Required** — S3 URLs are public (via CloudFront distribution)

### Error Handling

- **Network Errors** → Retry with exponential backoff (max 3 attempts)
- **404 Not Found** → Sermon may have been deleted; show error gracefully
- **500 Server Error** → Show "try again later" to user

---

## Website ↔ Global API

| Field | Value |
|-------|-------|
| **Source** | Website (Next.js) |
| **Destination** | Global API |
| **Protocol** | HTTPS REST JSON |
| **Auth** | None required (all endpoints public) |
| **Caching** | Next.js ISR (Incremental Static Regeneration) + API caching |
| **Revalidation** | Pages revalidated on-demand when API data changes |

### Operations

| Operation | Method | Endpoint | Auth | Purpose |
|-----------|--------|----------|------|---------|
| Get Homepage Data | GET | `/api/messages?limit=5` + `/api/config` | None | Load latest sermons + settings |
| Get Series Page | GET | `/api/series/{id}` | None | Show all messages in a series |
| Get Sermon Page | GET | `/api/messages/{id}` | None | Display full sermon with playback |
| Search Sermons | GET | `/api/messages?search=...&tags=...` | None | Search results page |
| Get Events | GET | `/api/events` | None | Upcoming events section |

### Static Generation Strategy

- **Series list pages** → ISR with 24-hour revalidation
- **Individual sermon pages** → ISR with 7-day revalidation
- **Homepage** → ISR with 1-hour revalidation (to show latest sermons)

### Error Handling

- **API Down** → Show cached version (fallback)
- **404 Not Found** → Return 404 page (sermon was deleted)
- **Timeouts** → Retry once, then show error

---

## AI Pipeline ↔ Global API & MongoDB

| Field | Value |
|-------|-------|
| **Source** | AWS Lambda (Python) |
| **Destination** | Global API (for triggers) + MongoDB (for reads/writes) |
| **Protocol** | HTTPS REST (trigger) + MongoDB driver (direct) |
| **Auth** | API: none (internal service) / MongoDB: connection string (env var) |
| **Invocation** | EventBridge rule or direct Lambda invoke from API |

### Data Flow: Sermon Upload

```
1. Admin Tool → Global API: POST /api/messages (with audio S3 URL)
2. Global API: Creates message record in MongoDB
3. Global API: Invokes transcription_processor Lambda
4. Lambda chain: transcription → sermon_processor → podcast_rss_generator
5. Each Lambda: Reads from MongoDB, writes enriched data back
```

### Operations

| Operation | Lambda | Purpose |
|-----------|--------|---------|
| Transcribe Audio | `transcription_processor` | Download audio from S3, call Azure Speech API, store transcript |
| Enrich Sermon | `sermon_processor` | Generate summary + tags (OpenAI), create waveform, update MongoDB |
| Generate Series Summary | `series_summary_processor` | Aggregate completed series, generate overview summary |
| Generate Podcast Feed | `podcast_rss_generator` | Create podcast descriptions, update RSS XML on S3 |

### MongoDB Collections (read/write)

| Collection | Operation | Purpose |
|-----------|-----------|---------|
| `Messages` | READ | Get raw sermon data (audio URL, metadata) |
| `Messages` | WRITE | Update with summary, tags, waveform, transcript link |
| `Series` | READ | Check if series is complete (has EndDate) |
| `Series` | WRITE | Update with series-level summary |
| `PodcastEpisodes` | WRITE | Add/update episode with podcast description |

### Error Handling

- **Audio Download Fails** → Retry 3 times, then log error (don't crash pipeline)
- **Azure Speech Timeout** → Fallback to transcript placeholder
- **MongoDB Connection Error** → Exponential backoff (Lambdas have timeout limit)

---

## Livestream Controller ↔ OBS & ProPresenter

| Field | Value |
|-------|-------|
| **Source** | Stream Controller (C#/.NET web app) |
| **Destination** | OBS Studio (WebSocket) + ProPresenter 7 (HTTP API) |
| **Protocol** | WebSocket (OBS 4.9+), HTTP REST (ProPresenter) |
| **Auth** | OBS: WebSocket auth token / ProPresenter: HTTP basic auth |
| **Timeout** | 30 seconds (OBS), 10 seconds (ProPresenter) |

### Operations

| Service | Operation | Purpose |
|---------|-----------|---------|
| **OBS** | Get Scene List | Populate UI dropdown of available scenes |
| **OBS** | Switch to Scene | Change active video output (camera 1, camera 2, slides, etc.) |
| **OBS** | Start Streaming | Begin live broadcast to Facebook/YouTube |
| **OBS** | Stop Streaming | End live broadcast |
| **OBS** | Get Stream Status | Show "streaming" indicator in UI |
| **OBS** | Monitor CPU/Memory | Alert if system is overloaded |
| **ProPresenter** | Next Slide | Advance slide deck |
| **ProPresenter** | Previous Slide | Go back slide |
| **ProPresenter** | Show/Hide Graphics Layer | Toggle graphics overlay on NDI output |

### Error Handling

- **OBS Not Responding** → Show "connection lost" in UI, prompt to restart OBS
- **ProPresenter Timeout** → Retry once, then show error
- **Network Disconnect** → Auto-reconnect every 5 seconds

---

## Admin Tool ↔ AWS S3

| Field | Value |
|-------|-------|
| **Source** | Admin Tool (browser, JavaScript) |
| **Destination** | AWS S3 |
| **Protocol** | HTTPS REST (multipart upload) |
| **Auth** | AWS SigV4 (signed requests from backend) |
| **Flow** | Admin Tool requests signed URL from API, then uploads directly to S3 |

### Operations

| Operation | Purpose | Size Limit |
|-----------|---------|------------|
| Upload sermon audio | Store MP3/WAV file in S3 bucket | 500 MB |
| Upload series artwork | Store JPG/PNG image | 10 MB |
| Upload message artwork | Store JPG/PNG image | 10 MB |

### Upload Flow

```
1. Admin Tool: Click "Upload Audio"
2. Admin Tool → Global API: POST /api/upload/request-url (filename + size)
3. Global API: Generates AWS SigV4 signed URL (valid 15 min)
4. Global API: Returns signed URL to Admin Tool
5. Admin Tool: Direct upload to S3 using signed URL
6. S3: Confirms upload, Admin Tool shows success
```

---

## Mobile App ↔ AWS S3

| Field | Value |
|-------|-------|
| **Source** | Mobile App (React Native) |
| **Destination** | AWS S3 (or CloudFront CDN) |
| **Protocol** | HTTPS GET |
| **Auth** | None (public URLs with CloudFront distribution) |
| **Caching** | App caches audio locally for offline play |

### Operations

| Operation | URL Type | Purpose |
|-----------|----------|---------|
| Download sermon audio | Pre-signed or public S3 URL | Stream or cache audio |
| Download artwork | CloudFront CDN URL | Show series/message images |
| Get podcast RSS | CloudFront CDN URL | Podcast apps sync feed |

---

## Website ↔ AWS S3

| Field | Value |
|-------|-------|
| **Source** | Website (Next.js) |
| **Destination** | AWS S3 (CloudFront CDN) |
| **Protocol** | HTTPS GET |
| **Auth** | None (public CDN URLs) |

### Operations

| Operation | Purpose |
|-----------|---------|
| Fetch audio URLs | Embed playback in sermon page |
| Fetch artwork images | Show series/message thumbnails |
| Fetch podcast RSS | Consume for podcast player widget |

---

## Podcast Platforms ↔ S3

| Field | Value |
|-------|-------|
| **Source** | Apple Podcasts, Spotify, Google Podcasts, etc. |
| **Destination** | AWS S3 (podcast RSS feed) |
| **Protocol** | HTTPS GET (RSS XML) |
| **Auth** | None (public feed) |
| **Refresh Interval** | Platform-dependent (typically every 24 hours) |

### Data

- **Feed URL** → S3 location of `podcast.xml`
- **Episode URLs** → S3 direct URLs or CloudFront CDN
- **Update Trigger** → AI Pipeline regenerates RSS after each new sermon

---

## Summary Table: Who Talks to Whom

| Source | Destination | Protocol | Auth | Frequency |
|--------|-------------|----------|------|-----------|
| Admin Tool | Global API | HTTPS REST | JWT | On demand (staff) |
| Admin Tool | S3 | HTTPS (signed) | AWS SigV4 | On upload |
| Mobile App | Global API | HTTPS REST | None (public) | On user action |
| Mobile App | S3 | HTTPS | None (public) | On demand |
| Website | Global API | HTTPS REST | None (public) | Page load + ISR |
| Website | S3 | HTTPS | None (public) | Page load |
| AI Pipeline | Global API | HTTPS REST | None (internal) | On trigger |
| AI Pipeline | MongoDB | Direct driver | Connection string | Continuous |
| AI Pipeline | S3 | HTTPS + SDK | AWS credentials | Continuous |
| AI Pipeline | Azure API | HTTPS | API key | Per sermon |
| AI Pipeline | OpenAI API | HTTPS | API key | Per sermon |
| Podcast Platforms | S3 | HTTPS GET | None (public) | Periodic poll |
| Stream Controller | OBS | WebSocket | Token | Continuous |
| Stream Controller | ProPresenter | HTTP REST | Basic auth | On demand |

---

## Notes for Developers

### Adding a New Integration

1. **Define the protocol** — REST, gRPC, direct DB, event, WebSocket?
2. **Specify auth** — JWT, API key, AWS SigV4, none?
3. **Document the endpoint** — URL, method, required headers
4. **Error handling** — Retry logic, fallbacks, circuit breaker?
5. **Update this matrix** — Keep it in sync as you add flows

### Common Gotchas

- **Admin Tool can't reach API?** → Check `localhost:8080` in dev; verify API is running
- **Mobile App shows old data?** → Clear React Query cache or wait for revalidation
- **Podcast feed not updating?** → Check S3 permissions and RSS XML syntax
- **Livestream stuttering?** → Check OBS/ProPresenter network latency and WebSocket timeout settings
- **Lambda timeout errors?** → Increase timeout in SAM template; check MongoDB connection pooling

### Rate Limiting & Quotas

- **Global API** → 1000 req/min per IP (configurable)
- **Azure Speech API** → 20 concurrent transcriptions (check account limits)
- **OpenAI API** → Per-model pricing; monitor usage in dashboard
- **S3** → 3500 PUT/COPY/POST/DELETE per second per partition (not usually a bottleneck)
