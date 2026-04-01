# Cross-Project Dependencies

This document maps which repositories depend on which, and what happens when you change one. Use this to understand the blast radius of your changes and to plan coordinated deploys.

---

## Dependency Graph

```
ThriveChurchOfficialAPI (Global API)
  ↑
  │ depends on
  │
  ├─ ThriveAPIMediaTool (Admin Tool)
  ├─ ThriveChurchOfficialApp_CrossPlatform (Mobile App)
  ├─ Thrive-FL.org (Website)
  ├─ AWSLambdas (AI Pipeline)
  └─ Thrive_Stream_Controller (Stream Controller)

AWSLambdas (AI Pipeline)
  ├─ depends on
  │  ├─ ThriveChurchOfficialAPI (reads/writes via MongoDB)
  │  ├─ Azure Speech API
  │  └─ OpenAI API
  │
  └─ triggers from
     └─ ThriveChurchOfficialAPI (when sermon created)

ThriveAPIMediaTool (Admin Tool)
  ├─ depends on
  │  └─ ThriveChurchOfficialAPI (for all operations)
  │
  └─ indirect
     └─ AWSLambdas (triggers pipeline when admin uploads)

ThriveChurchOfficialApp_CrossPlatform (Mobile App)
  └─ depends on
     └─ ThriveChurchOfficialAPI (for all content)

Thrive-FL.org (Website)
  └─ depends on
     └─ ThriveChurchOfficialAPI (for all content)

Thrive_Stream_Controller (Stream Controller)
  ├─ depends on
  │  └─ ThriveChurchOfficialAPI (optional: to mark sermons as "live")
  │
  └─ external
     ├─ OBS Studio (via WebSocket)
     └─ ProPresenter 7 (via HTTP)

ProPresenter_Automations
  └─ depends on
     └─ ProPresenter 7 & OBS Studio APIs
```

---

## By Dependant Repo

### ThriveChurchOfficialAPI (Global API)

**Role:** Central hub — nothing else works if this is down.

**Depends On:**
- MongoDB (production: `mongo-thrive-production`, dev: `mongo-localhost`)
- Redis (production only; in-memory fallback in dev)
- AWS S3 (for audio and podcast feed storage)
- Azure Speech API (indirectly via Lambda)
- OpenAI API (indirectly via Lambda)

**Who Depends On It:**
- ThriveAPIMediaTool (Admin Tool)
- ThriveChurchOfficialApp_CrossPlatform (Mobile App)
- Thrive-FL.org (Website)
- AWSLambdas (AI Pipeline)
- Thrive_Stream_Controller (Stream Controller)

**Critical Methods/Endpoints (breaking change risks):**

| Endpoint | Dependents | Breaking Change? |
|----------|-----------|------------------|
| `GET /api/messages` | Mobile, Website | **YES** — removes sermon list |
| `GET /api/series/{id}` | Mobile, Website | **YES** — breaks series pages |
| `POST /api/messages` | Admin Tool, Lambda | **YES** — can't upload sermons |
| `PUT /api/messages/{id}` | Lambda (enrichment) | **YES** — breaks data enrichment |
| `GET /api/config` | Mobile, Website | **MAYBE** — if config structure changes |
| `POST /api/upload/request-url` | Admin Tool | **YES** — can't upload audio |

**Backward Compatibility Notes:**
- Add new fields to responses (safe) ✓
- Remove fields from responses (breaks clients) ✗
- Change HTTP status codes (may break retry logic) ✗
- Change authentication method (breaks all clients) ✗
- Add required request body fields (breaks old clients) ✗

---

### ThriveAPIMediaTool (Admin Tool)

**Role:** Uploading gate — staff can't upload sermons if this is down.

**Depends On:**
- ThriveChurchOfficialAPI (for all API calls)
- AWS S3 (for signed URLs and direct upload)

**Who Depends On It:**
- Nothing directly, but staff depend on it to populate content

**Critical Operations:**

| Operation | What Breaks If It Fails |
|-----------|--------|
| Sermon upload | New sermons can't be published |
| Series creation | Can't organize sermons into series |
| Config updates | Church settings can't be changed |

**Version Constraints:**
- Must be compatible with Global API version (no version pinning; assumes latest API is always live)
- If API changes response structure, Admin Tool needs update within same release cycle

**Deploy Risk:**
- **LOW** — downtime only affects staff, doesn't break playback
- Deploy independently; no coordination needed with App/Website

---

### ThriveChurchOfficialApp_CrossPlatform (Mobile App)

**Role:** Attender playback — most users' primary experience.

**Depends On:**
- ThriveChurchOfficialAPI (for all sermon data)
- AWS S3 (for audio streaming)

**Who Depends On It:**
- Nothing (clients depend on it)

**Critical Operations:**

| Operation | What Breaks If It Fails |
|-----------|--------|
| List sermons | No home screen feed |
| Play audio | Can't listen to messages |
| Search | Can't find sermons by topic |
| Offline download | Can't cache sermons locally |

**Version Constraints:**
- Expects API to return specific fields (series, messages, summaries, tags, artwork)
- If API changes message schema, old app versions may break
- **Solution:** API must maintain backward compatibility OR old app versions must be forced to update via app store

**Deploy Risk:**
- **CRITICAL** — affects all users
- Should coordinate with API changes
- Plan multi-version support if schema changes are required

**Rollback Procedure:**
- If app version X breaks due to API changes, revert API to previous version
- Alert users to update app from store (takes 24-48 hours to propagate)

---

### Thrive-FL.org (Website)

**Role:** Public discovery — visitors' entry point.

**Depends On:**
- ThriveChurchOfficialAPI (for sermon content)
- AWS S3 (for audio and images)

**Who Depends On It:**
- Nothing (public-facing only)

**Critical Operations:**

| Operation | What Breaks If It Fails |
|-----------|--------|
| Load homepage | No sermon list visible |
| Render sermon pages | No sermon playback |
| Search sermons | No discovery |

**Version Constraints:**
- Same as Mobile App — expects specific API response schema
- Typically more flexible (Next.js can handle missing fields gracefully)
- ISR (Incremental Static Regeneration) caches pages; won't notice API changes immediately

**Deploy Risk:**
- **HIGH** — affects all visitors
- Update API, then update Website (within same release)
- Website revalidates on-demand; no need to clear cache manually

---

### AWSLambdas (AI Pipeline)

**Role:** Content enrichment — without this, sermons have no summaries, tags, or podcast feeds.

**Depends On:**
- ThriveChurchOfficialAPI (reads raw sermon data, writes enriched data)
  - Actually: MongoDB directly (bypasses API for writes, for performance)
- AWS S3 (reads audio, writes transcripts and podcast RSS)
- Azure Speech API (transcription)
- OpenAI API (summarization, tagging)

**Who Depends On It:**
- ThriveChurchOfficialAPI (implicitly — API returns enriched data written by Pipeline)
- Mobile App (reads summaries, tags, waveform)
- Website (reads summaries, tags)
- Podcast platforms (consume RSS generated by Pipeline)

**Critical Operations:**

| Lambda | What Breaks If It Fails |
|--------|---------|
| transcription_processor | Sermons have no transcripts; can't be searched |
| sermon_processor | No summaries or tags (summaries + tags are critical UX) |
| podcast_rss_generator | Podcast platforms don't sync; RSS feed is stale |
| series_summary_processor | No series-level summaries (nice-to-have, not critical) |

**Version Constraints:**
- Must match MongoDB schema (Series, Messages, PodcastEpisodes collections)
- If you change API response schema, Pipeline may break (it reads from MongoDB, not API)
- Be careful when renaming MongoDB fields

**Deploy Risk:**
- **MEDIUM** — doesn't break existing playback, but new sermons won't be enriched
- Can be deployed independently
- If it fails, queue up new sermons in Admin Tool and retry after fix is deployed

**Failure Modes:**

| Failure | Impact | Recovery |
|---------|--------|----------|
| Azure Speech API down | Transcription fails | Retry after Azure recovers |
| OpenAI API rate limit | Summarization queued slowly | Wait or upgrade API plan |
| MongoDB connection error | Can't read/write enriched data | Fix connection string, redeploy |
| S3 permissions error | Can't read audio or write RSS | Check IAM role, redeploy |
| Lambda timeout | Long sermons don't transcribe completely | Increase timeout in SAM, redeploy |

---

## Release Sequencing

### Scenario 1: API Schema Change (New Required Field)

**Sequence:**
1. Deploy new **API version** (with backward compatibility)
2. Deploy **Mobile App** update (reads new field)
3. Deploy **Website** update (reads new field)
4. Stop supporting old API version (after app stores rollout complete)

**Rollback:**
- Revert API, then Mobile/Website (clients will work again with old API)

---

### Scenario 2: AI Pipeline Enhancement (e.g., Better Tagging)

**Sequence:**
1. Deploy new **Lambda** version
2. Optionally re-process existing sermons (manual operation)
3. No client changes needed (API response already includes tags)

**Rollback:**
- Revert Lambda, re-process sermons with old version

---

### Scenario 3: Admin Tool UI Changes

**Sequence:**
1. Deploy new **Admin Tool** version
2. No other changes needed (Admin Tool is isolated)

**Rollback:**
- Revert Admin Tool; staff may lose in-progress uploads

---

### Scenario 4: Critical Security Fix in API

**Sequence:**
1. Deploy **API** fix immediately
2. Deploy **Mobile App + Website** updates within 24 hours
3. Contact users to update app (if needed)

**Rollback:**
- Test thoroughly before deploying; critical fixes shouldn't need rollback

---

## Change Impact Analysis

### If You Change Global API...

**Changing:** Sermon message response schema

**Impacts:**
- Mobile App ✓ (must update to read new/removed fields)
- Website ✓ (must update to display new fields)
- Admin Tool ✓ (if it displays sermon summaries, may need update)
- AI Pipeline ✓ (if it writes to changed fields, must update)
- Podcast platforms ✗ (no direct impact; only read RSS)

**Action Items:**
1. Check all clients (Mobile, Website, Admin Tool) for schema usage
2. Deploy API with **backward compatibility** (keep old fields, add new ones)
3. Update clients to use new fields
4. After all clients updated, remove old fields in next API version

---

### If You Change Mobile App API Usage...

**Changing:** Mobile App calls `GET /api/messages?sort=date` → switches to `?sort=relevance`

**Impacts:**
- Global API ✗ (no change needed; already supports relevance sorting)
- Website ✗ (uses different sorting logic; no impact)
- Admin Tool ✗ (no impact)
- AI Pipeline ✗ (no impact)

**Action Items:**
1. Deploy Mobile App update
2. No coordination needed with other repos

---

### If You Change MongoDB Schema...

**Changing:** Rename `Messages.summary` → `Messages.tldrSummary`

**Impacts:**
- Global API ✓ (must update model + mapper)
- Mobile App ✓ (must read `tldrSummary` instead of `summary`)
- Website ✓ (must read `tldrSummary` instead of `summary`)
- AI Pipeline ✓ (must write to `tldrSummary`)

**Action Items:**
1. Database migration script (MongoDB schema is flexible; old docs still work, but queries won't find new field)
2. Update **API** to read/write both old and new field names (migration period)
3. Update **Mobile + Website** to use new field name
4. Update **Lambda** to write new field name
5. After everything is deployed, run migration script to rename fields in all documents
6. Remove backward-compatibility code from API

---

### If You Change AI Pipeline Processing...

**Changing:** Sermon Processor now also generates `studyGuide` in addition to summary

**Impacts:**
- Global API ✗ (no change; API already returns all MongoDB fields)
- Mobile App ✓ (optional; can show study guide if present)
- Website ✓ (optional; can show study guide if present)
- Admin Tool ✗ (no impact)

**Action Items:**
1. Update **Lambda** to generate study guide
2. Update **Mobile App** to display study guide (optional feature flag)
3. Update **Website** to display study guide (optional feature flag)
4. Can be done independently; no strict ordering required

---

### If You Change Admin Tool...

**Changing:** Admin Tool UI redesign (no API changes)

**Impacts:**
- Global API ✗ (no impact)
- Mobile App ✗ (no impact)
- Website ✗ (no impact)
- AI Pipeline ✗ (no impact)

**Action Items:**
1. Deploy **Admin Tool** independently
2. No coordination needed

---

## Version Compatibility Matrix

| Component A | Component B | Compatibility | Notes |
|-------------|-----------|---|---------|
| API v1.5 | Mobile App v1.0 | ✓ Compatible | App was built against API v1.4 |
| API v2.0 | Mobile App v1.0 | ✗ Likely broken | API removed backward compatibility |
| API v1.5 | Admin Tool v1.3 | ✓ Compatible | Admin Tool uses public API endpoints |
| Lambda v3.0 | API v1.5 | ✓ Compatible | Lambda only reads/writes MongoDB; doesn't talk to API |
| Website (ISR) | API v1.5 | ✓ But stale | Website caches old data; revalidates on demand |

---

## Deployment Checklist

Before deploying a change:

- [ ] Identify affected repos (use this document)
- [ ] Check version compatibility (can old/new versions coexist?)
- [ ] Plan rollback procedure (what breaks if this fails?)
- [ ] Update documentation (keep Architecture docs in sync)
- [ ] Notify team (if change affects user-facing behavior)
- [ ] Test in staging first (if possible)
- [ ] Monitor logs after deploy (watch for errors in dependent repos)

---

## Common Mistakes to Avoid

1. **Deploying API changes without updating Admin Tool** → Staff UI breaks
2. **Changing MongoDB schema without migration** → Existing documents don't match new code
3. **Removing API endpoints without deprecation period** → Old app versions break immediately
4. **Deploying Lambda without testing locally** → Sermons don't enrich; no way to debug in prod
5. **Forgetting to update integration matrix** → Team doesn't know blast radius of changes

---

## Asking "What Do I Need to Update?"

If you're changing component X:

1. **Check "Who Depends On It" section** for X above
2. **For each dependent**, ask:
   - Does it call my API/interface?
   - Does it read/write the same data?
   - Does it expect specific behavior from my code?
3. **Update those repos** in the same release
4. **Test together** in staging (if possible)
5. **Update this document** if you discover new dependencies
