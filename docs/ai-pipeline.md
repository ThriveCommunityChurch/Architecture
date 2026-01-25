# AI Processing Pipeline

The **AI Processing Pipeline** is a set of cloud‑based services that run behind the scenes to make sermons easier to discover and listen to.

## What It Does

After a sermon is uploaded through the **Admin Tool**, the AI pipeline:

1. **Transcribes Audio**
   - Converts the sermon audio into text using Azure Speech-to-Text.
   - Stores the full transcript in Azure Blob Storage for downstream processing.
2. **Generates a Message Summary**
   - Produces a TLDR-style summary (130-180 words) using straightforward, educational tone.
   - Uses first person plural (we/us/our) perspective, leading with the topic or lesson.
3. **Assigns Topical Tags**
   - Selects from 90+ predefined tags across 17 categories (Theological, Spiritual Disciplines, Personal Growth, Relationships, etc.).
   - Tags are mapped to C# enums and stored as integers for efficient querying.
4. **Builds Waveform Data**
   - Creates a 200-point waveform representation for audio player visualization in the mobile app.
5. **Keeps the Podcast Feed Updated**
   - Generates podcast-friendly descriptions (two paragraphs, 130-180 words) written for non-church audiences.
   - Descriptions name specific tensions, avoid speaker names and rhetorical questions, and end with curiosity.
   - Updates the PodcastEpisodes collection and regenerates the RSS XML feed.
6. **Generates Series Summaries** *(when series is complete)*
   - When a series has an end date and all messages are processed, generates a cohesive series-level summary.
   - Summaries state insights as present-tense timeless truths with varied openings.

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
                    |  Pipeline (Lambdas)    |
                    +-----------+------------+
                                |
        +-----------------------+------------------------------+
        v                       v                              v
  Transcript            Enriched sermon data          Podcast RSS feed
  (Azure Blob)          (summary, tags, waveform)            |
                               |                            v
                               v                     Podcast platforms
                        Series Summary
                        (when complete)
                               |
                               v
                      Mobile App & Website
                    (read enriched data)
```

## Lambda Architecture

The pipeline consists of four AWS Lambda functions that work together:

```text
┌──────────────────┐     ┌──────────────────────┐     ┌────────────────────┐
│   .NET API       │     │  Transcription       │     │  Sermon Processor  │
│  (App Runner)    │────▶│  Processor           │───▶│                    │
│                  │     │                      │     │  • Summary (GPT)   │
│                  │     │  • Download audio    │     │  • Tags (GPT)      │
│                  │     │  • Azure Speech API  │     │  • Waveform        │
│                  │     │  • Store transcript  │     │  • Update MongoDB  │
└──────────────────┘     └──────────┬───────────┘     └─────────┬──────────┘
                                    │                           │
                                    │                           │ (if series complete)
                                    │                           ▼
                                    │                 ┌────────────────────┐
                                    │                 │  Series Summary    │
                                    │                 │  Processor         │
                                    │                 │                    │
                                    │                 │  • Aggregate msgs  │
                                    │                 │  • Series summary  │
                                    │                 │  • Update MongoDB  │
                                    │                 └────────────────────┘
                                    │
                                    │                 ┌────────────────────┐
                                    └────────────────▶│  Podcast RSS       │
                                                      │  Generator         │
                                                      │                    │
                                                      │  • Description     │
                                                      │  • Update episodes │
                                                      │  • Update RSS XML  │
                                                      └────────────────────┘
```

| Lambda | Purpose |
|--------|---------|
| **Transcription Processor** | Downloads audio from S3, transcribes with Azure Speech API, stores transcript in Azure Blob Storage, triggers downstream Lambdas |
| **Sermon Processor** | Generates TLDR-style message summary (130-180 words), assigns tags from 90+ categories, creates waveform data. Triggers Series Summary Processor for completed series |
| **Series Summary Processor** | Aggregates all message summaries in a series and generates a present-tense timeless truths summary |
| **Podcast RSS Generator** | Generates two-paragraph podcast descriptions for non-church audience (no speaker names), updates PodcastEpisodes collection, regenerates RSS XML |

## Where It Runs

- Implemented as **AWS Lambda** serverless functions.
- Triggered automatically when new or updated content is ready.
- Communicates with the **Global API's** MongoDB database to read and update sermon records.
- Uses **Azure Blob Storage** for transcript storage.
- Uses **AWS S3** for audio files and podcast RSS feed.

## Example End-to-End Flow

1. A staff member uploads a new sermon through the **Admin Tool**.
2. The **Global API** stores basic details and an audio URL, then triggers the pipeline.
3. The **Transcription Processor** Lambda:
   - Downloads the audio from S3
   - Calls Azure Speech-to-Text to create a transcript
   - Stores the transcript in Azure Blob Storage
   - Invokes the Sermon Processor and Podcast RSS Generator in parallel
4. The **Sermon Processor** Lambda:
   - Generates a TLDR-style summary (straightforward, educational, we/us/our perspective)
   - Assigns topical tags from 90+ predefined categories
   - Creates waveform data for the audio player
   - Saves all enriched data to MongoDB
   - If the series has an end date, triggers the Series Summary Processor
5. The **Series Summary Processor** Lambda *(conditional)*:
   - Checks that all messages in the series have summaries
   - Aggregates all message summaries
   - Generates a series summary as present-tense timeless truths
   - Saves to the Series document in MongoDB
6. The **Podcast RSS Generator** Lambda:
   - Generates a two-paragraph description for spiritual seekers (no speaker names)
   - Updates the PodcastEpisodes collection
   - Regenerates the RSS XML feed on S3

From the church's point of view, this all happens automatically after the single upload step.

## Why It Matters

This pipeline:

- Saves staff time by automating summaries and tagging.
- Makes it easier for people to **find sermons based on their needs** (e.g., anxiety, marriage, finances) with 90+ topical tags.
- Provides **series-level context** so users can understand what an entire series covers.
- Powers a richer listening experience in the **mobile app** and **podcast platforms**.
- Reaches **new audiences** through thoughtfully crafted podcast descriptions that resonate with spiritual seekers.

## Key Technologies

At a high level, the pipeline uses:

- **AWS Lambda** for serverless compute
- **AWS S3** for storing audio files and podcast RSS feed
- **Azure Speech-to-Text** for transcription
- **Azure Blob Storage** for transcript storage
- **Azure OpenAI (GPT-5-mini)** for summarization, tagging, and description generation
- **MongoDB** as the single source of truth for enriched sermon data

The detailed implementation may evolve over time, but the core idea remains the same: automatically turn raw audio into rich, searchable content and keep all downstream experiences in sync.

From the outside, it's invisible—but it's a key part of what makes the Thrive ecosystem feel smart and cohesive.

