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

## Why It Matters

This pipeline:

- Saves staff time by automating summaries and tagging.
- Makes it easier for people to **find sermons based on their needs** (e.g., anxiety, marriage, finances).
- Powers a richer listening experience in the **mobile app** and **podcast platforms**.

From the outside, it's invisible—but it's a key part of what makes the Thrive ecosystem feel smart and cohesive.

