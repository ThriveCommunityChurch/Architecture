# System Overview

The Thrive ecosystem is built around a simple idea:

> **Have one central source of truth for sermon and church content, and publish it out to every place people engage.**

To make that work, we use a small set of focused components:

- A **Global API** that stores and serves all sermon and configuration data
- A **Mobile App** that uses the API to show content to attenders
- A **Public Website** that also uses the API to stay in sync
- An **Admin Tool** that staff use to manage content
- An **AI Processing Pipeline** that enriches sermons with summaries, tags, and podcast feeds

## Diagram

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
            | (summary, 90+ tags,    |
            |  waveform, podcast)    |
            +------------------------+
                      |
                      v
                Podcast Platforms
```

## High‑Level Flow

1. **Sermon is recorded**  
   Audio is edited and prepared by the production team.

2. **Staff upload the sermon**  
   Using the **Admin Tool**, staff upload audio and fill in basic details (title, date, series, speaker, etc.).  
   The **Global API** securely stores this information and manages links to the audio file.

3. **AI enhances the content**
   The **AI Processing Pipeline**:
   - Generates a TLDR-style summary (130-180 words) with straightforward, educational tone
   - Assigns topical tags from 90+ categories (e.g., hope, marriage, anxiety, finances)
   - Prepares waveform data for audio player visualization
   - Generates podcast descriptions written for non-church audiences
   - Creates series-level summaries when a series is complete

4. **Content goes live everywhere**  
   - The **Mobile App** and **Public Website** call the **Global API** to:
     - List sermon series
     - Show individual messages
     - Display summaries, tags, and artwork
   - Podcast platforms use the generated **podcast feed** to stay in sync.

5. **Attenders engage**  
   - In the **Mobile App**, people can:
     - Browse and search sermons
     - Play audio (online or downloaded)
     - View summaries and tags
     - Access "Connect", "Give", and other church resources
   - On the **Website**, people see the same core content, plus marketing and visitor‑focused pages.

## Why This Design?

This architecture is intentionally:

- **Centralized** - One place (the Global API) is responsible for the actual data.
- **Flexible** - Multiple front‑ends (app, website, admin tool) can evolve independently.
- **Future‑friendly** - New experiences (a TV app, kiosk, etc.) can plug into the same API without redoing the content pipeline.

## Example Weekly Workflow

Here is a typical rhythm for a normal week:

1. **Sunday** – The sermon is preached and recorded.
2. **Early week (Mon–Tue)** – Audio is edited and exported by the production team.
3. **Mid‑week** – A staff member uses the **Admin Tool** to:
   - Upload the audio file
   - Attach the message to the right series
   - Enter title, date, speaker, and passage
4. **Shortly after upload** – The **AI Processing Pipeline** runs automatically to:
   - Transcribe the message using Azure Speech-to-Text
   - Generate a TLDR-style summary and assign tags from 90+ categories
   - Prepare waveform data and generate a podcast description for non-church audiences
5. **Same day** – The **Mobile App** and **Website** automatically show the new message via the **Global API**; podcast apps receive the updated feed.

From the team's perspective, the main manual task is **one upload in the Admin Tool**. Everything else fans out automatically.

## Key Technologies (High‑Level)

Without going deep into implementation details, the system uses:

- A **C#/.NET API** for the Global API layer
- **MongoDB** for storing sermon and podcast data
- **AWS** (including S3 and Lambda) for media storage and serverless processing
- A **web‑based Admin Tool** for staff (Angular)
- A **mobile app** built with modern cross‑platform tooling (Expo/React Native)
- A **public website** that consumes the same API as the app
- **GitHub Pages** for hosting this architecture documentation

The rest of this documentation looks at each part of the system in more detail.

