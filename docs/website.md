# Public Website (`thrive-fl.org`)

The **public website** is the main marketing and information hub for Thrive.
It is aimed at both regular attenders and first-time visitors.

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

Non-sermon content (e.g., static info pages, hero banners) is maintained in the website repo itself,
but any sermon-related content comes from the **Global API**.

## Role in the Ecosystem

- First touch point for many new people searching for a church.
- A familiar place for members to re-watch or share sermons.
- A key consumer of the same content pipeline that feeds the app and podcast.

## Typical Visitor Journeys

Some common ways people use the site:

- **First-time visitor**
  - Googles for churches nearby and lands on `thrive-fl.org`
  - Reads about service times, ministries, and what to expect
  - Watches or listens to a recent message to get a feel for the church

- **Regular attender**
  - Replays a message they missed on Sunday
  - Shares a specific sermon link with a friend
  - Uses the site to sign up for groups, serving, or special events

In each case, sermon content shown on the website is **identical** to what appears in the mobile app,
because both read from the same Global API.

## Key Technologies (High-Level)

At a high level, the website:

- Is built with a modern web stack (JavaScript/TypeScript, HTML, CSS)
- Uses the **Global API** over HTTPS to load sermon and series data
- Hosts static assets (images, CSS, JavaScript) via standard web hosting

The exact framework and build setup are documented in the website repo itself. This page focuses on how
the site fits into the larger ecosystem.
