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

## Typical Requests (High Level)

Some examples of how other systems use the API:

- **Mobile App**
  - "Give me the newest messages for the home screen."
  - "Search for messages tagged with 'hope' from the last year."
- **Website**
  - "Show all messages in the current series."
  - "Load details for this specific message page."
- **Admin Tool**
  - "Create a new message with this title, date, speaker, and audio URL."
  - "Update the artwork for this series."
- **AI Processing Pipeline**
  - "Read the raw transcript for this message."
  - "Write back the generated summary, tags, and waveform data."

All of these go through the same Global API, which keeps rules and validation in one place instead of scattering them across multiple apps.

## Key Technologies (High‑Level)

At a technology level (without going into implementation detail), the Global API:

- Is built with **C#/.NET**
- Uses **MongoDB** as its primary data store
- Exposes **JSON over HTTPS** for both internal and external clients
- Uses **JWT authentication** for non‑public operations (e.g., admin changes)

The exact endpoints and schemas are documented in the API repo itself. This page focuses on the role of the API in the overall system.
