# Repository to Architecture Mapping

This document maps each GitHub repository to the architectural components shown in the [System Overview](system-overview.md).

## Quick Reference Table

| GitHub Repo | Architecture Component | Tech Stack | Primary Purpose |
|---|---|---|---|
| `ThriveChurchOfficialAPI` | Global API | C#/.NET 8, MongoDB, Redis, Docker | Central hub for all sermon and configuration data |
| `ThriveAPIMediaTool` | Admin Tool | Angular 20, TypeScript, Docker/nginx | Staff UI for uploading and managing content |
| `ThriveChurchOfficialApp_CrossPlatform` | Mobile App | React Native 0.81 / Expo, TypeScript, Zustand, React Query | iOS and Android mobile app for attenders |
| `Thrive-FL.org` | Public Website | Next.js 16, TypeScript, Howler.js, AWS Amplify | Public marketing site and sermon browsing |
| `AWSLambdas` | AI Processing Pipeline | Python 3.11, AWS Lambda, Azure Speech API, OpenAI | Transcription, summarization, tagging, podcast generation |
| `Sermon_Summarization_Agent` | AI Pipeline (LangGraph Agent) | Python, LangGraph, OpenAI | Future: advanced sermon analysis and enrichment |
| `ThriveAPIMediaTool` | Media Storage & Serving | AWS S3 | Hosts sermon audio files and podcast RSS feeds |
| `Thrive_Stream_Controller` | Livestream Control Dashboard | C#/.NET, React, Docker | One-click stream management UI |
| `ProPresenter_Automations` | Livestream Graphics Automation | Python, ProPresenter API | Auto-triggers OBS scene changes and graphics during playback |
| `Architecture` | Documentation | Markdown, GitHub Pages | Central source of truth for system architecture and relationships |

## Component Details

### Global API
**Repository:** `ThriveChurchOfficialAPI`
**Stack:** C#/.NET 8, MongoDB, Redis, Docker
**Endpoints:** Swagger UI at `/swagger` (dev: `localhost:8080`)

**Responsibilities:**
- Store and serve sermon series and messages
- Manage church configuration settings
- Handle JWT authentication for admin operations
- Provide search and discovery endpoints
- Track message playback analytics
- Manage refresh tokens and user sessions

**Data Flow:**
- **Reads from:** MongoDB (SermonSeries database), Redis cache
- **Writes to:** MongoDB, Redis
- **Consumed by:** Mobile App, Website, Admin Tool, AI Pipeline, Livestream Controller

### Admin Tool
**Repository:** `ThriveAPIMediaTool`
**Stack:** Angular 20, TypeScript, Docker/nginx
**Dev URL:** `localhost:4200` (expects API at `localhost:8080`)

**Responsibilities:**
- Provide staff interface for uploading sermon audio
- Manage sermon metadata (title, date, series, speaker, artwork)
- Update series and configuration settings
- Trigger AI processing pipeline for newly uploaded sermons

**Data Flow:**
- **Reads from:** Global API (GET endpoints, public)
- **Writes to:** Global API (POST/PUT endpoints, requires JWT)
- **File uploads to:** AWS S3 (sermon audio)

### Mobile App
**Repository:** `ThriveChurchOfficialApp_CrossPlatform`
**Stack:** React Native 0.81 / Expo, TypeScript, Zustand (state), React Query (data fetching)
**Root folder:** `ThriveChurchExpo/`

**Responsibilities:**
- Display sermon series and messages to attenders
- Stream and download sermon audio
- Show summaries, tags, and artwork
- Provide "Connect", "Give", and other church resources
- Support search and filtering by topic

**Data Flow:**
- **Reads from:** Global API (HTTPS REST)
- **Fetches media from:** AWS S3 (sermon audio)
- **Limitations:** iOS builds not possible on Windows; Android and Expo Go only

### Public Website
**Repository:** `Thrive-FL.org`
**Stack:** Next.js 16, TypeScript, Howler.js (audio playback), AWS Amplify
**Dev URL:** `localhost:3000` (via `npm run dev`)

**Responsibilities:**
- Display sermon content to visitors
- Show church events and visitor information
- Marketing and outreach pages
- Same sermon browsing experience as mobile app (from same API)
- SEO-friendly sermon pages

**Data Flow:**
- **Reads from:** Global API (HTTPS REST)
- **Fetches media from:** AWS S3 (sermon audio)
- **Deployment:** AWS Amplify (auto-deploy on git push)

### AI Processing Pipeline
**Repository:** `AWSLambdas`
**Stack:** Python 3.11, AWS Lambda, Azure Speech-to-Text, OpenAI API, AWS S3
**Config:** AWS SAM template at repo root

**Responsibilities:**
- Transcribe sermon audio using Azure Speech-to-Text
- Generate message summaries (130-180 words, TLDR style)
- Assign topical tags from 90+ categories (17 categories total)
- Generate waveform data for audio player visualization
- Create podcast-friendly descriptions (for non-church audiences)
- Generate series-level summaries when a series is complete

**Lambda Functions:**
1. `transcription_processor` – Receives upload event, downloads audio, calls Azure Speech API, stores transcript
2. `sermon_processor` – Generates summary, tags, waveform; updates MongoDB; conditionally invokes series_summary_processor
3. `podcast_rss_generator` – Creates podcast descriptions, updates PodcastEpisodes collection, regenerates RSS XML
4. `series_summary_processor` – Aggregates completed series, generates cohesive series summary, updates MongoDB

**Data Flow:**
- **Triggered by:** Global API (after sermon upload)
- **Reads from:** MongoDB (raw sermon data), AWS S3 (sermon audio)
- **Writes to:** MongoDB (enriched data: summaries, tags, waveform), AWS S3 (podcast RSS feed, transcript)
- **External services:** Azure Speech API (transcription), OpenAI API (summarization/tagging)

### Sermon Summarization Agent
**Repository:** `Sermon_Summarization_Agent`
**Stack:** Python, LangGraph (AI orchestration), OpenAI, LangChain
**Purpose:** Advanced multi-step sermon analysis (future expansion of AI pipeline)

**Responsibilities:**
- Advanced sermon understanding and enrichment
- Potential future integration with main AI Pipeline
- Currently separate from production flow

### Media Storage & Serving
**Service:** AWS S3
**Hosted in:** Global API configuration (S3 bucket paths and credentials)

**Hosted on S3:**
- Sermon audio files (uploaded via Admin Tool)
- Podcast RSS feed XML (generated by AI Pipeline)
- Waveform data (generated by AI Pipeline)
- Artwork and images

**Access:**
- Admin Tool uploads audio files
- Mobile App and Website fetch audio for playback
- Podcast platforms consume RSS feed

### Livestream & Production
**Primary Repository:** `Thrive_Stream_Controller`
**Support Repository:** `ProPresenter_Automations`
**Stack:**
- Stream Controller: C#/.NET, React, Docker
- Automations: Python, ProPresenter API, OBS API

**Responsibilities:**
- Capture three camera feeds (Canon C100 II) via Blackmagic ATEM switcher
- Stream live to Facebook and YouTube via OBS Studio
- Auto-trigger scene changes and graphics via ProPresenter 7
- Record service for post-production editing
- One-click stream management dashboard

**Data Flow:**
- Hardware: ATEM switcher → OBS Studio → Facebook/YouTube
- Graphics: ProPresenter 7 ↔ OBS (NDI overlay) ↔ Automations service
- Control: Stream Controller dashboard → OBS/ProPresenter APIs

See [Livestream & Production](livestream-production.md) for detailed signal flow.

---

## Data Direction: Content Pipeline

```
Staff ──upload──> Admin Tool ──> Global API ──> MongoDB + S3
                                     |
                                invokes (when sermon created)
                                     |
                                     v
                        AI Processing Pipeline
                        (transcription → enrichment)
                                     |
                                     v
                          Enriched data back to MongoDB
                          (summaries, tags, waveform, RSS)
                                     |
                 ┌───────────────────┴──────────────────┐
                 v                                       v
         Mobile App & Website                   Podcast Platforms
         (read via Global API)              (Apple, Spotify, Google, etc.)
```

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| **Backend API** | C#/.NET 8, MongoDB, Redis, JWT |
| **Frontend (Staff)** | Angular 20, TypeScript |
| **Frontend (Attenders - Mobile)** | React Native 0.81 / Expo |
| **Frontend (Attenders - Web)** | Next.js 16, TypeScript |
| **AI & Processing** | Python 3.11, AWS Lambda, OpenAI API, Azure Speech |
| **Media** | AWS S3 |
| **Cache & Sessions** | Redis (prod), in-memory (dev) |
| **Database** | MongoDB (SermonSeries database) |
| **Livestream** | OBS Studio, ProPresenter 7, Blackmagic ATEM, Python automations |
| **Deployment** | Docker, AWS App Runner, AWS Amplify, AWS Lambda/SAM |

## Key Cross-Repo Integrations

### Global API ↔ Admin Tool
- Admin Tool calls API endpoints to upload and manage content
- Auth: JWT tokens (obtained via login endpoint)
- Protocol: HTTPS REST JSON
- Critical: API must be running for Admin Tool to function

### Global API ↔ Mobile App / Website
- Both call API to fetch sermon lists, series details, and metadata
- Auth: GET endpoints are public (no JWT required)
- Protocol: HTTPS REST JSON
- Caching: Mobile app uses React Query; Website uses Next.js data fetching
- Media links: API returns S3 URLs; clients download directly from S3

### Global API ↔ AI Pipeline
- Global API triggers Lambda on sermon creation (AWS EventBridge or direct invocation)
- Pipeline reads from MongoDB directly or via API
- Pipeline writes enriched data back to MongoDB (bypasses API)
- Scalability: Multiple Lambda instances can run in parallel

### AI Pipeline ↔ S3
- Pipeline downloads raw audio from S3
- Stores transcripts and waveform data in S3
- Generates and uploads podcast RSS feed to S3

### Stream Controller ↔ OBS / ProPresenter
- Communicates via WebSocket API
- Controls scene switching, streaming state
- Receives live status updates

---

## Deployment Units

Each repo deploys independently:

- **Global API** → Docker image → ECR → AWS App Runner (triggered by `master` push)
- **Admin Tool** → Docker image → ECS or standalone Docker
- **Mobile App** → Expo/EAS build → App Stores (iOS/Android)
- **Website** → Next.js build → AWS Amplify (auto-deploy on push)
- **AI Pipeline** → Python SAM → AWS Lambda (via GitHub Actions)
- **Stream Controller** → Docker → self-hosted or ECS
- **Automations** → Python → self-hosted (on streaming machine)

See [Deployment Order](deployment-order.md) for safe update sequences.
