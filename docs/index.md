# Thrive Architecture

Welcome to the **Thrive Architecture** overview.

These pages describe, at a high level, how Thrive's systems work together:

- The **Global API** that stores and serves sermon and church content
- The **Mobile App** used by attenders
- The **Public Website** (`thrive-fl.org`)
- The internal **Admin Tool** used by staff
- The **AI Processing Pipeline** that powers summaries, tags, and the podcast feed

This site is intentionally **overview-focused**. For deep technical details (code, schemas, deployment scripts), see the individual project repositories.

---

## Quick System Diagram

```text
        Attenders                          Staff
            |                               |
            v                               v
   +----------------+              +-----------------+
   |  Mobile App    |              |   Admin Tool    |
   +--------+-------+              +--------+--------+
            \                           /
             \                         /
              \       HTTPS          /
               v                     v
                 +---------------------+
                 |      Global API     |
                 | (sermons, config)   |
                 +----+-----------+----+
                      |           |
                data  |           | audio
                      v           v
                +-----------+  +--------+
                | Database  |  |  S3    |
                +-----------+  +--------+
                      |
                      v
            +------------------------+
            | AI Processing Pipeline |
            | (summary, tags, RSS)   |
            +------------------------+
                      |
                      v
                Podcast Platforms
```

## Content Creation Flow

At a glance, here is how a sermon moves from the stage to people's phones, browsers, and podcast apps:

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
   +-------------------+
   |    Global API     |
   | (store details &  |
   |  audio location)  |
   +---------+---------+
             |
             | triggers / data
             v
   +---------------------------+
   |   AI Processing Pipeline  |
   |  • transcript & summary   |
   |  • tags & waveform data   |
   |  • podcast RSS updates    |
   +-----------+---------------+
               |
     +---------+-----------------------------+
     |                                       |
     v                                       v
Mobile App & Website                  Podcast Platforms
(read enriched data                    (subscribe to RSS
  from Global API)                         feed)
```

---

## Sections

Use the links below to dive into each part of the system:

- [System Overview](system-overview.md)
- [Global API](global-api.md)
- [Admin Tool](admin-tool.md)
- [Mobile App](mobile-app.md)
- [Public Website](website.md)
- [AI Processing Pipeline](ai-pipeline.md)

Each page is written to be understandable by both **technical** and **non-technical** team members.
