# AI Processing Pipeline

The **AI Processing Pipeline** is a set of cloud‑based services that automatically transcribe sermons so people can search, find, and access sermon content.

## What It Does

After a sermon is uploaded through the **Admin Tool**, the AI pipeline automatically:

1. **Transcribes Audio** — Converts sermon audio into searchable text
2. **Generates Series Summaries** *(when series is complete)* — Summarizes completed series

## How It Works

```text
   Staff uploads sermon
   via Admin Tool
           |
           v
   +------------------+
   | AI Processing    |
   | Pipeline         |
   +--------+---------+
            |
   Transcript stored
   (searchable, archived)
            |
            v
   Mobile App & Website
   (people search & find)
```

## Processing Stages

The pipeline orchestrates these processing stages automatically:

1. **Transcription Stage** — Converts audio to searchable text.
2. **Series Processing Stage** *(conditional)* — When a series is complete, synthesizes all message content into a series summary.

## Technical Foundation

The pipeline runs on a **cloud-based, serverless architecture**:

- **Automatically triggered** when sermons are uploaded or updated
- **Scalable** — handles demand without manual server management
- **Integrated** with the Global API database to read and update sermon records
- **Distributed** — processes audio, text, and metadata in parallel for speed

## From a Practical View

From the staff and congregation's perspective:

1. **Staff uploads** a sermon through the Admin Tool.
2. **Pipeline runs automatically** in the background—no manual steps needed.
3. **Within minutes**, the transcript is searchable and available.

The technology handles the complexity so your team can focus on ministry.

## Why It Matters

This pipeline creates real value:

- **Saves staff time** — No manual transcription needed.
- **Makes sermons discoverable** — People can search for sermon content using text.

## Why Cloud-Based?

The pipeline uses cloud services so it can:

- **Scale automatically** — Handle any number of sermons without manual intervention.
- **Stay reliable** — Retry automatically if something goes wrong.
- **Improve over time** — Get better as AI models improve, without disrupting your workflow.

