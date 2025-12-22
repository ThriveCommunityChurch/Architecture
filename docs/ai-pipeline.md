# AI Processing Pipeline

The **AI Processing Pipeline** is a set of cloud‑based services that run behind the scenes to make sermons easier to discover and listen to.

## What It Does

After a sermon is uploaded through the **Admin Tool**, the AI pipeline:

1. **Transcribes Audio**
   - Converts the sermon audio into text using speech‑to‑text technology.
2. **Generates a Summary**
   - Produces a concise, readable summary of the message.
3. **Assigns Topical Tags**
   - Attaches meaningful tags (e.g., "hope", "forgiveness", "marriage") based on the content.
4. **Builds Waveform Data**
   - Prepares the data needed to draw the audio waveform in the mobile app player.
5. **Keeps the Podcast Feed Updated**
   - Adds or updates podcast episodes in a feed that podcast apps can subscribe to.

## Diagram

```text
   Admin uploads audio
   via Admin Tool
           |
           v
   +--------------+      +-----------+
   |  Admin Tool  | ---> | Global API|
   +--------------+      +-----+-----+
                               |
                        data   |   audio links
                               v
                         +-----------+
                         | DB & S3   |
                         +-----+-----+
                               |
                               v
                    +------------------------+
                    |  AI Processing         |
                    |  Pipeline              |
                    +-----------+------------+
                                |
        +-----------------------+------------------------------+
        v                       v                              v
  Transcript & tags     Enriched sermon data          Podcast RSS feed
                        (summary, tags, waveform)            |
                               |                            v
                               v                     Podcast platforms
                      Mobile App & Website
                    (read enriched data)
```

## Where It Runs

- Implemented as **serverless functions** in the cloud.
- Triggered automatically when new or updated content is ready.
- Communicates with the **Global API's** database to read and update sermon records.

## Example End-to-End Flow

1. A staff member uploads a new sermon through the **Admin Tool**.
2. The **Global API** stores basic details and an audio URL.
3. An **AWS Lambda** (or similar serverless function) is triggered to:
   - Download the audio
   - Call a speech-to-text service to create a transcript
4. Another step in the pipeline uses the transcript to:
   - Generate a concise summary in plain language
   - Suggest topical tags based on the content
5. The pipeline writes the enriched data (summary, tags, waveform info) back to the **Global API**'s database.
6. A final step updates the **podcast RSS feed**, which podcast apps read on their own schedule.

From the church's point of view, this all happens automatically after the single upload step.

## Why It Matters

This pipeline:

- Saves staff time by automating summaries and tagging.
- Makes it easier for people to **find sermons based on their needs** (e.g., anxiety, marriage, finances).
- Powers a richer listening experience in the **mobile app** and **podcast platforms**.

## Key Technologies (High-Level)

At a high level, the pipeline uses:

- **AWS Lambda** or equivalent serverless compute
- **AWS S3** for storing and reading audio files
- A **speech-to-text service** (e.g., Whisper API) to create transcripts
- **LLM-based summarization and tagging** to enrich the content
- The **Global API** and its database as the single source of truth for enriched sermon data

The detailed implementation may evolve over time, but the core idea remains the same: automatically turn raw audio into rich, searchable content and keep all downstream experiences in sync.

From the outside, it's invisible—but it's a key part of what makes the Thrive ecosystem feel smart and cohesive.

