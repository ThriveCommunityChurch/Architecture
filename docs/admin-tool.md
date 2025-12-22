# Admin Tool

The **Admin Tool** is a web‑based dashboard used by church staff and volunteers to manage content.

## What It Does

Through a simple web interface, staff can:

- **Create and manage series**
  - Start new sermon series and update artwork and descriptions
- **Add and edit messages**
  - Upload weekly sermon audio
  - Set titles, dates, speakers, and scripture passages
- **Trigger processing**
  - Uploads and updates feed into the **AI Processing Pipeline** so summaries, tags, and waveforms can be generated
- **Manage configuration**
  - Update key links and settings that drive the mobile app and website (e.g., "Give" link, "I'm New" page, small group sign‑ups)

## Diagram

```text
   +-----------+
   |  Staff    |
   |  User     |
   +-----+-----+
         |
         | Web browser
         v
   +-----------------+
   |   Admin Tool    |
   |   (Dashboard)   |
   +--------+--------+
            |
            | HTTPS
            v
   +-----------------+
   |   Global API    |
   +--------+--------+
            |
     data   |   audio
            v
   +------------------------+
   | Database & Media Store |
   +------------------------+
            |
         triggers
            v
   +------------------------+
   | AI Processing Pipeline |
   +------------------------+
```

## How It Connects

- Talks to the **Global API** for all operations (no direct database access).
- Uses the same configuration system that the **Mobile App** reads from, so changes made here appear across platforms.

## Typical Workflows

Common ways staff use the Admin Tool include:

- **Weekly sermon upload**
  - Drag and drop the new audio file
  - Confirm or edit title, date, speaker, and series
  - Save and let the AI pipeline handle summaries, tags, waveforms, and podcast updates

- **Fixing or improving details**
  - Update a typo in a title or description
  - Swap out artwork for a series
  - Adjust tags if the auto-generated ones need tweaking

- **Keeping the app and website current**
  - Change the "Give" or "I'm New" links
  - Update links for small groups, serving, or events

In all of these cases, staff work in one place, and the changes automatically flow to the mobile app, website, and podcast feed.

## Key Design Choices

- **Browser-based** - Runs in a normal web browser, so staff do not need special software installed.
- **Guardrails by design** - Uses the Global API for validation and security, reducing the risk of broken content.
- **Future-proof** - New fields or flows can be added to the Admin Tool without changing the underlying data model too often.

## Audience

- Tech team
- Communications/creative team
- Staff overseeing sermons, podcasts, and digital content

The goal is to let non‑developers keep all content fresh without needing to touch code.

