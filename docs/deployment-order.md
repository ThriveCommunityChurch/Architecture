# Deployment Order & Release Planning

This document describes the safe sequence for deploying changes across the Thrive ecosystem. Use this when coordinating multiple repos or rolling back failures.

---

## Core Principles

1. **Central-out pattern:** Deploy the **Global API first**, then fan out to dependent services.
2. **Backward compatibility:** API must support old clients during rollout period.
3. **Monitor before moving:** Verify each service is healthy before deploying the next one.
4. **Rollback symmetry:** Roll back in reverse order (last deployed first).

---

## Standard Deployment Sequence

Use this order for most multi-repo changes:

### Phase 1: Infrastructure & Core API
| Step | Repo | Component | Verification |
|------|------|-----------|--------------|
| 1 | `ThriveChurchOfficialAPI` | Global API | Swagger UI responds, health check passes |
| 2 | `AWSLambdas` | AI Pipeline | Lambda functions callable, MongoDB connection works |

### Phase 2: Admin / Staff Tools
| Step | Repo | Component | Verification |
|------|------|-----------|--------------|
| 3 | `ThriveAPIMediaTool` | Admin Tool | Admin can login, upload test sermon |

### Phase 3: User-Facing Clients
| Step | Repo | Component | Verification |
|------|------|-----------|--------------|
| 4 | `ThriveChurchOfficialApp_CrossPlatform` | Mobile App | App loads, can browse sermons (iOS: through TestFlight, Android: new build) |
| 5 | `Thrive-FL.org` | Website | Website loads, sermon pages render, search works |

### Phase 4: Production Streaming (if needed)
| Step | Repo | Component | Verification |
|------|------|-----------|--------------|
| 6 | `Thrive_Stream_Controller` | Stream Controller | Dashboard connects to OBS/ProPresenter |
| 7 | `ProPresenter_Automations` | Automations | Automations respond to ProPresenter events |

---

## Scenario-Based Sequences

### Scenario A: API-Only Change (e.g., New Endpoint, Bugfix)

**Changes to:** `ThriveChurchOfficialAPI` only (backward compatible)

**Deploy order:**
1. Deploy **API**
2. Mobile App and Website automatically use new endpoint (no code changes)
3. Done

**Rollback:** Revert API, restart service

**Timing:** ~5 minutes total

---

### Scenario B: API Schema Change (New Field)

**Changes to:** `ThriveChurchOfficialAPI` (adds new field), Mobile App and Website (consume new field)

**Deploy order:**
1. Deploy **API** with new field (must be optional in response, not break old clients)
2. Wait for Mobile App to roll out to app stores (24-48 hours for iOS, 2-4 hours for Android via Google Play)
3. Deploy **Website** (next.js app updates faster)
4. Verify all clients see new field before removing backward-compatibility code

**Backward compatibility period:**
- API adds field but marks old field as deprecated (both present for 2-3 releases)
- Old Mobile App sees old field, new Mobile App sees both, latest Mobile App reads new field

**Rollback:** Revert API to previous version; all clients continue working with old field

**Timing:** 24-48 hours (waiting for app store rollout)

---

### Scenario C: AI Pipeline Enhancement (Better Summarization)

**Changes to:** `AWSLambdas` only (no schema changes, just different summaries)

**Deploy order:**
1. Deploy **Lambda** (e.g., new sermon_processor version)
2. Existing sermons already have summaries; new sermons get new summaries
3. No client changes needed (Mobile App + Website already display summaries)

**Optional:**
- Re-process existing sermons with new Lambda (manual operation, time-consuming)
- Monitor quality of new summaries before deciding to bulk-update

**Rollback:** Revert Lambda; re-process sermons with old version

**Timing:** ~5 minutes to deploy, ~2-3 hours per 100 sermons to re-process (if needed)

---

### Scenario D: Admin Tool Only Change (UI Redesign)

**Changes to:** `ThriveAPIMediaTool` only (no API changes)

**Deploy order:**
1. Deploy **Admin Tool** (standalone Docker container)
2. Staff uses new UI immediately

**Rollback:** Revert Admin Tool container

**Timing:** ~2 minutes (Docker restart)

---

### Scenario E: MongoDB Schema Migration (Rename Field)

**Changes to:** API, AI Pipeline, and optionally Mobile/Website

**This is dangerous. Follow this sequence carefully:**

1. **Prepare API** (before data change):
   - Update API code to read/write BOTH old and new field names
   - Deploy API with backward-compatibility code
   - Verify API still works with old database schema

2. **Deploy updated API:**
   - API now writes to `newFieldName` but still reads `oldFieldName` for backward compatibility

3. **Run migration script** (offline, ideally):
   - MongoDB script to copy all documents: `oldFieldName` → `newFieldName`
   - Example: `db.Messages.updateMany({}, [{ $set: { newFieldName: "$oldFieldName" } }])`
   - Test on dev MongoDB first, then run on production

4. **Remove backward-compatibility code** (in next release):
   - API removes code that reads old field
   - All clients are now updated to use new field
   - Clean up old field from MongoDB (optional)

**Example timeline:**
- Day 1: Deploy API with migration code (still uses old field in responses)
- Day 2-3: Wait for Mobile App + Website to update (reads new field)
- Day 4: Run migration script (all documents now have both fields)
- Day 5+: Deploy second API version (stops using old field, removes backward-compatibility)

**Rollback at any step:**
- Before migration: Revert API, all clients still work (API writes old field)
- After migration: Can't easily un-migrate (keep backup of MongoDB first)

**Timing:** 3-5 days (due to app store rollout and careful sequencing)

---

## Deployment-By-Deployment Checklists

### Deploying Global API

**Before Deployment:**
- [ ] Run full test suite locally (`dotnet test`)
- [ ] Check for breaking changes (schema, endpoint signature)
- [ ] Confirm backward compatibility (old clients should still work)
- [ ] Review configuration (environment variables, connection strings)
- [ ] Tag version in git (`git tag -a v1.X.Y`)

**Deployment Steps:**
1. Push to `dev` branch (trigger GitHub Actions CI)
2. Wait for CI/CD pipeline to pass
3. Merge `dev` → `master` to trigger deployment to production
4. Monitor AWS App Runner logs (`aws logs tail`)
5. Test health endpoint: `curl https://<api-url>/health`

**Post-Deployment:**
- [ ] Verify Swagger UI loads
- [ ] Test critical endpoints (list sermons, create message)
- [ ] Check MongoDB connection (query `db.Messages.findOne()`)
- [ ] Monitor error logs for 10 minutes
- [ ] Notify team in Slack if any issues

**Rollback (if needed):**
```bash
git revert <commit-hash>
git push master  # Triggers automatic revert deployment
```

---

### Deploying Admin Tool

**Before Deployment:**
- [ ] Run test suite (`ng test --watch=false`)
- [ ] Build Docker image locally (`docker build -t admin-tool:latest .`)
- [ ] Test image locally (`docker run -p 4200:80 admin-tool:latest`)
- [ ] Verify it connects to API (`curl http://localhost:4200` should load)

**Deployment Steps:**
1. Build and push Docker image to ECR or Docker Hub
2. Deploy to hosting (Docker/ECS or self-hosted)
3. Test login endpoint
4. Test sermon upload form

**Post-Deployment:**
- [ ] Staff confirms login works
- [ ] Test upload with small test file
- [ ] Monitor network requests in browser DevTools
- [ ] Check browser console for errors

**Rollback:**
- Redeploy previous Docker image tag

---

### Deploying Mobile App

**Before Deployment:**
- [ ] Run test suite (`npm test`)
- [ ] Build APK/IPA locally
- [ ] Test on physical device (if possible) or emulator
- [ ] Verify API connection (check network tab in DevTools)
- [ ] Verify audio playback works with test sermon

**iOS Deployment:**
1. Build with EAS (`eas build --platform ios`)
2. Upload to TestFlight for internal testing
3. Wait 1-2 hours for app review
4. Gather feedback from testers
5. Submit to App Store (requires Apple review, ~24 hours)
6. Users auto-update (or prompt them to)

**Android Deployment:**
1. Build with EAS (`eas build --platform android`)
2. Upload to Google Play Console
3. Release to 5% of users first (staged rollout)
4. Monitor crash rates for 24 hours
5. If stable, release to 100% of users

**Post-Deployment:**
- [ ] Monitor Firebase Crashlytics for crashes
- [ ] Check app store reviews for complaints
- [ ] Verify new feature works for users who update

**Rollback:**
- Remove app store listing OR
- Submit app update to revert to previous version (slower, ~24 hour review)

---

### Deploying Website

**Before Deployment:**
- [ ] Run test suite (`npm test`)
- [ ] Build locally (`npm run build`)
- [ ] Test locally (`npm start`)
- [ ] Verify API connections
- [ ] Check page load performance

**Deployment Steps:**
1. Push to `dev` branch
2. Merge `dev` → `master` (triggers AWS Amplify auto-deploy)
3. Monitor Amplify build logs
4. Verify CloudFront cache invalidation
5. Test critical pages (homepage, sermon page)

**Post-Deployment:**
- [ ] Homepage loads (check home page data is fresh)
- [ ] Sermon pages render (try specific sermon)
- [ ] Search works
- [ ] Audio playback works
- [ ] Monitor Amplify logs for errors

**Rollback:**
- Revert commit to `master`
- Amplify automatically rebuilds and deploys previous version

---

### Deploying AI Pipeline (Lambdas)

**Before Deployment:**
- [ ] Run test suite (`pytest`)
- [ ] Validate SAM template (`sam validate`)
- [ ] Build locally (`sam build`)
- [ ] Test with sample audio file locally (if possible)
- [ ] Verify MongoDB connection string in environment variables
- [ ] Check Azure Speech API and OpenAI API credentials

**Deployment Steps:**
1. Build Lambdas: `sam build`
2. Deploy to staging first: `sam deploy --config-file samconfig-staging.toml`
3. Test on staging:
   - Upload test sermon via Admin Tool pointing to staging API
   - Verify Lambda executes (check CloudWatch logs)
   - Verify summary + tags appear in MongoDB
4. If staging passes, deploy to production: `sam deploy --config-file samconfig-production.toml`
5. Monitor CloudWatch logs for new sermon uploads

**Post-Deployment:**
- [ ] CloudWatch logs show Lambda execution
- [ ] No errors in logs (check "ERROR" level messages)
- [ ] Test with real sermon upload (Admin Tool)
- [ ] Verify summary appears in MongoDB within 2 minutes
- [ ] Verify podcast RSS regenerates

**Rollback:**
- AWS Lambda automatically keeps previous version
- Update SAM template to point to previous Lambda version
- `sam deploy` redeploys previous version

---

### Deploying Stream Controller

**Before Deployment:**
- [ ] Build Docker image
- [ ] Test locally against OBS WebSocket
- [ ] Test against ProPresenter HTTP API
- [ ] Verify all scene switching logic works

**Deployment Steps:**
1. Deploy to streaming machine (Docker Compose or manual Docker)
2. Test WebSocket connection to OBS
3. Test HTTP connection to ProPresenter
4. Verify dashboard loads in browser

**Post-Deployment:**
- [ ] Dashboard connects to OBS (green "Connected" indicator)
- [ ] Scene list populates
- [ ] Slide controls respond (next/previous)
- [ ] Scene switching works

**Rollback:**
- Stop Docker container, restart previous version

---

## Coordinating Large Changes

When multiple repos need updates simultaneously:

### Step 1: Plan (Before any coding)
- [ ] List all repos that need changes
- [ ] Identify dependencies (use [cross-project-dependencies.md](cross-project-dependencies.md))
- [ ] Determine backward-compatibility requirements
- [ ] Create deployment sequence (use this document)

### Step 2: Implement (In parallel, but don't deploy yet)
- [ ] Implement changes in all repos
- [ ] Code review in each repo
- [ ] Write tests for all changes
- [ ] Verify test suites pass

### Step 3: Deploy Staging (In sequence, to parallel environment)
- [ ] Deploy API to staging
- [ ] Deploy Admin Tool to staging
- [ ] Deploy Mobile App to staging (if possible)
- [ ] Deploy Website to staging
- [ ] Deploy Lambda to staging
- [ ] Test end-to-end flow in staging (upload → enrich → display)

### Step 4: Deploy Production (In sequence, following this document)
- [ ] Deploy API
- [ ] Wait 10 minutes, monitor logs
- [ ] Deploy Admin Tool
- [ ] Deploy Website
- [ ] Deploy Mobile App
- [ ] Deploy Lambda
- [ ] Verify end-to-end flow

### Step 5: Post-Deployment Verification
- [ ] Admin uploads test sermon
- [ ] Lambda processes it (check logs)
- [ ] Website shows it (ISR revalidates)
- [ ] Mobile App shows it (after cache expiry or force refresh)
- [ ] Podcast feed updates

### Step 6: Communicate
- [ ] Notify team of deployment
- [ ] Alert users if major feature change
- [ ] Monitor error logs for 1 hour
- [ ] Be on-call for rollback if issues arise

---

## Rollback Decision Tree

```
Is there a critical error?
│
├─ YES
│  │
│  ├─ API Error (service down, 500s)
│  │  └─ Revert API immediately
│  │     └─ All clients fallback to previous API version
│  │
│  ├─ Mobile App Crash (users can't open app)
│  │  └─ Revert App in app store (24-48 hour turnaround)
│  │  └─ Meanwhile, run API rollback to stabilize (if needed)
│  │
│  ├─ Website Down (users can't view sermons)
│  │  └─ Revert Website immediately (Amplify redeploys old version)
│  │
│  ├─ Lambda Error (sermons don't enrich)
│  │  └─ Revert Lambda
│  │  └─ Re-process failed sermons after fix
│  │
│  └─ Admin Tool Error (staff can't upload)
│     └─ Revert Admin Tool
│
└─ NO → Monitor for 1 hour, then declare deployment stable
```

---

## Common Rollback Scenarios

### Scenario: API returns null field, breaks Mobile App

**Time to detect:** 1-5 minutes (user error reports)
**Time to fix:** 5 minutes (fix code + redeploy)
**Impact:** Users can't see sermon data

**Recovery:**
1. Revert API to previous version
2. Verify Mobile App works again (should work immediately)
3. Fix the bug in API code
4. Re-deploy API with fix

---

### Scenario: Lambda timeout, sermons don't enrich

**Time to detect:** 5-15 minutes (staff notices no summaries)
**Time to fix:** 5 minutes (increase timeout + redeploy)
**Impact:** New sermons don't get summaries

**Recovery:**
1. Increase Lambda timeout in SAM template
2. Redeploy Lambda
3. Wait for next sermon upload to test
4. If still broken, rollback to previous Lambda version
5. Investigate root cause (audio too long? API too slow?)

---

### Scenario: Database migration partially completes

**Time to detect:** 10-30 minutes (queries start failing)
**Time to fix:** 30 minutes (run second migration script)
**Impact:** Some documents missing fields

**Recovery:**
1. Don't rollback (data is already partially migrated)
2. Write SQL/MongoDB script to complete migration
3. Test on backup database first
4. Run on production database
5. Verify data integrity

---

## Maintenance Windows

If a planned change risks downtime:

1. **Schedule maintenance window** (announce 1 week ahead)
2. **Set maintenance mode** in Global API (returns 503 for all endpoints)
3. **Notify users** on website/app that service is temporarily down
4. **Execute deployment** in sequence
5. **Test** each component thoroughly
6. **Disable maintenance mode** to bring service back online

**Estimated maintenance window:** 30 minutes - 2 hours (depending on scope)

---

## Health Checks & Monitoring

After each deployment, verify:

| Component | Health Check Command | Expected Response |
|-----------|---------------------|------------------|
| API | `curl https://api.../health` | `{ "status": "ok" }` |
| Admin Tool | Open dashboard in browser | Login page loads |
| Mobile App | Open app on device | Home screen shows sermons |
| Website | `curl https://thrive-fl.org` | Homepage HTML |
| Lambda | Check CloudWatch logs | No ERROR level messages |
| Database | `db.Messages.findOne()` | Recent sermon document |
| S3 | `aws s3 ls s3://bucket/` | Audio files present |

---

## Emergency Contacts & Escalation

- **API Down** → Wyatt (dev lead) → Check AWS App Runner dashboard
- **Database Down** → MongoDB support + Wyatt → Restore from backup
- **Lambda Stuck** → Check CloudWatch logs → Increase timeout or reduce batch size
- **Mobile App Crash** → Revert app in store → Fix code → Redeploy next version
- **Website Offline** → Revert to previous Amplify deployment
- **Livestream Issues** → Check OBS/ProPresenter logs → Restart services

---

## Version Tagging Convention

Use semantic versioning for releases:

- `v1.0.0` = Major version (breaking changes)
- `v1.1.0` = Minor version (new features, backward compatible)
- `v1.0.1` = Patch version (bugfix)

**Example:**
- API v1.0.0 → new schema (breaking)
- API v1.1.0 → new endpoints (backward compatible)
- API v1.0.1 → bugfix (backward compatible)

**Deploy strategy:**
- Patch versions: Deploy immediately (low risk)
- Minor versions: Deploy after testing (medium risk, backward compatible)
- Major versions: Plan carefully, coordinate all clients (high risk)
