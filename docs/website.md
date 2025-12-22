# Public Website (`thrive-fl.org`)

The **public website** is the main marketing and information hub for Thrive.  
It's aimed at both regular attenders and first‑time visitors.

## What It Shows

- **About and Visitor Information**
  - Service times, locations, "Plan a Visit", beliefs, and ministries
- **Sermons and Media**
  - Lists of recent messages and series
  - Detail pages where visitors can listen to sermons and read summaries
- **Next Steps**
  - Links to giving, small groups, serving, and contact forms
- **News and Events**
  - Announcements, special services, and other timely information

## Diagram

```text
   +------------+
   |  Visitor   |
   +-----+------+ 
         |
         | Browser
         v
   +--------------------+
   |  Public Website    |
   |  (thrive-fl.org)   |
   +----------+---------+
              |
              | HTTPS (sermons, media)
              v
        +-------------+
        |  Global API |
        +------+------+ 
               |
         data  |
               v
        +-------------+
        | DB & Media  |
        +-------------+
```

## How It Connects

The website:

- Uses the **Global API** to pull:
  - Sermon series and message details
  - Summaries, artwork, and tags
- Shares the **same data** as the mobile app so content is consistent across platforms.

Non‑sermon content (e.g., static info pages, hero banners) is maintained in the website repo itself, but any sermon‑related content comes from the **Global API**.

## Role in the Ecosystem

- First touch point for many new people searching for a church.
- A familiar place for members to re‑watch or share sermons.
- A key consumer of the same content pipeline that feeds the app and podcast.

