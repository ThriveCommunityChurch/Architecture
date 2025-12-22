# Thrive Architecture

This repository hosts the **high‑level architecture documentation** for Thrive Church's digital ecosystem.

It backs the public GitHub Pages site:

- **Thrive Architecture (GitHub Pages)**: https://thrivecommunitychurch.github.io/Architecture/

Use this repo when you want to understand how the **Global API**, **mobile app**, **website**, **admin tool**, and **AI processing pipeline** fit together.

## Scope

This documentation is intentionally **overview‑focused**:

- Explains *what* each major component does
- Shows *how* data flows between components
- Avoids deep internal implementation details (code, schemas, deployment scripts)

For low‑level technical details, see the individual project repositories listed below.

## Contents

- `docs/`
  - Markdown pages that describe each major part of the system.
  - Rendered as the GitHub Pages site linked above.
- `README.md`
  - This file, which explains the purpose of the Architecture docs and how to find them.

> Note: This repo was originally based on the AWS Lambdas project structure. Any remaining Lambda‑related files here are **historical** and not the source of truth for production code. See the `AWSLambdas` repo below for the active Lambda implementation.

## Related Repositories

These repositories contain the actual application and infrastructure code that this Architecture site describes:

- **Global API** – Main C# API for sermons and configuration  
  `https://github.com/ThriveCommunityChurch/ThriveChurchOfficialAPI`
- **Admin Tool / Media Tool** – Web UI for staff to upload and manage content  
  `https://github.com/ThriveCommunityChurch/ThriveAPIMediaTool`
- **Mobile App** – Thrive Church mobile app (Expo/React Native)  
  `https://github.com/ThriveCommunityChurch/ThriveChurchExpo`
- **Public Website** – Public marketing site (`thrive-fl.org`)  
  `https://github.com/ThriveCommunityChurch/Thrive-FL.org`
- **AWS Lambdas** – Serverless processing for transcription, enrichment, and podcast feeds  
  `https://github.com/ThriveCommunityChurch/AWSLambdas`

## Contributing

When you change how systems connect (new data flows, new services, or major refactors), update the relevant page(s) in `docs/` so that:

- New team members can quickly understand the big picture
- Architects and leads can discuss trade‑offs using a shared, up‑to‑date view of the system

After updating docs, commit and push to the `master` branch. GitHub Pages will automatically rebuild the site from the `docs/` folder.