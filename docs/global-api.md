# Global API

The **ThriveChurchOfficialAPI** is the central "brain" of the system.  
All other components talk to it when they need official data.

## What It Does

The Global API is responsible for:

- **Sermon and Series Data**
  - Stores all sermon series and individual messages
  - Keeps track of titles, dates, speakers, artwork, and audio links

- **Search and Discovery**
  - Supports searching by topic tags and speaker
  - Provides paginated lists of series and messages

- **Configuration**
  - Stores key‑value settings used by the mobile app and website  
    (e.g., contact info, "I'm New" link, social media URLs)

- **Analytics Hooks**
  - Exposes simple endpoints used to track which messages are played

## Diagram

```text
   +-------------+          +--------------------+
   | Mobile App  |  HTTPS   |                    |
   +-------------+ <------> |                    |
                             |    Global API     |
   +-------------+  HTTPS   | (sermons, config)  |
   |  Website    | <------> |                    |
   +-------------+          +---------+----------+
                                      |
                               data   |
                                      v
                             +----------------+
                             | Database & S3  |
                             +----------------+
```

## Who Uses It

- **Mobile App** - to browse and play sermons, show summaries and tags, and build menus.
- **Public Website** (`thrive-fl.org`) - to display sermon content and stay in sync with the app.
- **Admin Tool** - to create and update sermon content and church configuration.
- **AI Processing Pipeline** - to read and update enriched sermon data behind the scenes.

## Why It Matters

Having a single Global API means:

- Content is **consistent** everywhere (app, website, podcast).
- New front‑ends can be added without duplicating logic.
- Security, validation, and business rules live in one well‑defined place.

