# AI Processing Pipeline

The **AI Processing Pipeline** is a set of cloud‑based services that run behind the scenes to make sermons easier to discover and listen to.

## What It Does

After a sermon is uploaded through the **Admin Tool**, the AI pipeline:

1. **Transcribes Audio**
   - Converts the sermon audio into text using Azure Speech-to-Text.
2. **Generates a Message Summary**
   - Produces a concise, readable summary of the individual message.
3. **Assigns Topical Tags**
   - Attaches meaningful tags (e.g., "hope", "forgiveness", "marriage") based on the content.
4. **Builds Waveform Data**
   - Prepares the data needed to draw the audio waveform in the mobile app player.
5. **Keeps the Podcast Feed Updated**
   - Adds or updates podcast episodes in a feed that podcast apps can subscribe to.
6. **Generates Series Summaries** *(when series is complete)*
   - When a series has an end date and all messages are processed, automatically generates a cohesive summary of the entire sermon series.

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
| **Sermon Processor** | Generates message summary, tags, and waveform data using GPT. Triggers Series Summary Processor for completed series |
| **Series Summary Processor** | Aggregates all message summaries in a series and generates a cohesive series-level summary |
| **Podcast RSS Generator** | Generates podcast episode descriptions, updates PodcastEpisodes collection, and regenerates RSS XML |

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
   - Generates a concise summary using GPT
   - Generates topical tags using GPT
   - Creates waveform data for the audio player
   - Saves all enriched data to MongoDB
   - If the series has an end date, triggers the Series Summary Processor
5. The **Series Summary Processor** Lambda *(conditional)*:
   - Checks that all messages in the series have summaries
   - Aggregates all message summaries
   - Generates a cohesive series-level summary
   - Saves to the Series document in MongoDB
6. The **Podcast RSS Generator** Lambda:
   - Generates a podcast-friendly description
   - Updates the PodcastEpisodes collection
   - Regenerates the RSS XML feed on S3

From the church's point of view, this all happens automatically after the single upload step.

## Why It Matters

This pipeline:

- Saves staff time by automating summaries and tagging.
- Makes it easier for people to **find sermons based on their needs** (e.g., anxiety, marriage, finances).
- Provides **series-level context** so users can understand what an entire series covers.
- Powers a richer listening experience in the **mobile app** and **podcast platforms**.

## Key Technologies

At a high level, the pipeline uses:

- **AWS Lambda** for serverless compute
- **AWS S3** for storing audio files and podcast RSS feed
- **Azure Speech-to-Text** for transcription
- **Azure Blob Storage** for transcript storage
- **Azure OpenAI (GPT)** for summarization and tagging
- **MongoDB** as the single source of truth for enriched sermon data

The detailed implementation may evolve over time, but the core idea remains the same: automatically turn raw audio into rich, searchable content and keep all downstream experiences in sync.

From the outside, it's invisible—but it's a key part of what makes the Thrive ecosystem feel smart and cohesive.

