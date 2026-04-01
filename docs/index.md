# Thrive Architecture

Welcome to the **Thrive Architecture** overview.

These pages describe, at a high level, how Thrive's systems work together:

- The **Global API** that stores and serves sermon and church content
- The **Mobile App** used by attenders
- The **Public Website** (`thrive-fl.org`)
- The internal **Admin Tool** used by staff
- The **AI Processing Pipeline** that powers summaries, tags, and the podcast feed
- The **Livestream & Production Infrastructure** that captures, switches, and streams the in-person services

This site is intentionally **overview-focused**. For deep technical details (code, schemas, deployment scripts), see the individual project repositories.

---

## System Diagram

```text
     Visitors / Search              Attenders                Staff
          |                            |                       |
          | browser                    | iOS / Android         | browser
          v                            v                       v
   +----------------+        +----------------+       +-----------------+
   | Public Website |        |  Mobile App    |       |   Admin Tool    |
   | (thrive-fl.org)|        | (Expo / RN)   |       |   (Angular)     |
   +-------+--------+        +-------+--------+       +--------+--------+
           |                         |                          |
           |          HTTPS          |          HTTPS           |
           +------------+------------+-----------+--------------+
                        |                        |
                        v                        v
              +-------------------+     +------------------+
              |    Global API     |     |     AWS S3       |
              |   (C# / .NET)    |---->|  (sermon audio,  |
              |                   |     |   podcast RSS)   |
              +--------+----------+     +--------+---------+
                       |                         |
                 data  |                         |
                       v                         |
              +-------------------+              |
              |     MongoDB       |              |
              | (sermons, config, |              |
              |  podcasts, blogs) |              |
              +-------------------+              |
                       |                         |
                       |  invokes                |
                       v                         v
              +------------------------------------------+
              |        AI Processing Pipeline            |
              |             (Cloud-based)                 |
              |                                          |
              |  Transcription ──> Series Summary        |
              |       |                  |               |
              |       v                  v               |
              +-------------------+----------------------+
                                  |
                                  v
                      Transcript to MongoDB
                                  |
                                  v
                      Mobile App & Website
                      (read via Global API)
```

## Content Creation Flow

At a glance, here is how a sermon becomes searchable:

```text
  Sermon is preached
          |
          v
   Audio/Video edited by
   production team
          |
          v
   Staff upload via
      Admin Tool
          |
          v
   +---------------------------+
   |   AI Processing Pipeline  |
   |  Transcription            |
   |  Series Summaries         |
   +-----------+---------------+
               |
               v
     Mobile App & Website
     (people search & find)
```

---

## Sections

### Component Overview

Use the links below to dive into each part of the system:

- [System Overview](system-overview.md)
- [Global API](global-api.md)
- [Admin Tool](admin-tool.md)
- [Mobile App](mobile-app.md)
- [Public Website](website.md)
- [AI Processing Pipeline](ai-pipeline.md)
- [Livestream & Production](livestream-production.md)

Each page is written to be understandable by both **technical** and **non-technical** team members.

### Cross-Project Reference

When working across multiple repos or planning deployment changes:

- [Repository to Architecture Mapping](repo-to-architecture-mapping.md) — Which GitHub repo owns which component; tech stack for each
- [Integration Matrix](integration-matrix.md) — How components communicate; protocols, auth, and key operations
- [Cross-Project Dependencies](cross-project-dependencies.md) — Which repos depend on which; impact analysis for changes
- [Deployment Order](deployment-order.md) — Safe deployment sequences and rollback procedures
