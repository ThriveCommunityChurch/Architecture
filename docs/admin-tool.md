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

## Audience

- Tech team
- Communications/creative team
- Staff overseeing sermons, podcasts, and digital content

The goal is to let non‑developers keep all content fresh without needing to touch code.

