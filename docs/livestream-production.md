# Livestream & Production Infrastructure

The **livestream and production system** handles everything that happens on Sunday morning from a technical standpoint: capturing audio and video, mixing it, switching cameras, overlaying graphics, and streaming to YouTube and Facebook.

This page documents the physical equipment, signal flow, and software services that make it all work.

## What It Does

On a typical Sunday, the production system:

1. **Captures audio** from instruments, vocals, and the pastor's mic
2. **Mixes audio** for three separate destinations: house speakers, in‑ear monitors, and the livestream
3. **Captures video** from three cameras positioned around the room
4. **Switches cameras** through a centralized video switcher (ATEM)
5. **Overlays graphics** from ProPresenter (lyrics, sermon slides, announcements) via NDI
6. **Streams live** to YouTube and Facebook through OBS Studio
7. **Records a clean copy** of the program output on a separate laptop

## Audio Signal Flow

```text
[Drums] [Guitars] [Bass] [Vocals] [Piano] [Pastor Mic] [Violin] [Etc.]
                                │
                             (XLR)
                                │
                                ▼
                        ┌───────────────┐
                        │ Behringer S32 │  (Stage Box)
                        └───────┬───────┘
                          (AES50 Digital)
                                │
                                ▼
                        ┌───────────────┐
                        │ Behringer X32 │  (Main Mixer)
                        └───────┬───────┘
                                │
            ┌───────────────────┼────────────────────┐
            │                   │                    │
            ▼                   ▼                    ▼
     [House Speakers]  [P16 Distribution Hub]  [Stream Mix Out]
                               │                    │
              ┌────────┬───────┴───────┬────────┐   │
              ▼        ▼               ▼        ▼   │
           [P16]    [P16]           [P16]    [P16]  │
          (IEM 1)  (IEM 2)        (IEM 3)  (IEM n) │
                                                    │
                                                    ▼
                                    ┌──────────────────────────┐
                                    │  ATEM Television Studio  │
                                    │    Pro Studio HD         │
                                    │  (embeds audio + video   │
                                    │   into one HDMI stream)  │
                                    └─────────────┬────────────┘
                                                  │
                                           [Program Out]
                                             (HDMI)
                                                  │
                                                  ▼
                                           [HDMI Splitter]
                                            │           │
                                            ▼           ▼
                                     [AJA U‑TAP]   [AJA U‑TAP]
                                      (USB Video)   (USB Video)
                                            │           │
                                            ▼           ▼
                                     [Desktop PC]   [Laptop]
                                    (OBS Studio)   (Clean Recording)
                                            │
                                            ▼
                                   [Waves VST2 Plugins]
                                   (final broadcast mastering
                                    applied live inside OBS)
                                            │
                                            ▼
                                    (Livestream Output)
```

**Key points:**

- The **Behringer S32** is a 32‑channel digital stage box. All microphones and instruments connect here via XLR.
- The **S32 connects to the X32** over a single AES50 digital snake — one cable replaces dozens of analog runs.
- The **Behringer X32** produces three independent mixes:
  - **House mix** → main speakers in the room
  - **Monitor mixes** → P16 personal monitor system for each musician's in‑ear monitors (IEMs)
  - **Stream mix** → a separate, broadcast‑gain‑staged mix sent to the ATEM
- The **ATEM** embeds the audio and video together into a single HDMI stream. Its program output is split via an **HDMI splitter** to both PCs simultaneously.
- On the **Desktop PC**, OBS applies **Waves VST2 plugins** for final live broadcast mastering before streaming.
- The **Laptop** receives the same embedded audio+video feed for a clean, unprocessed recording.

## Video Signal Flow

```text
[Canon C100 Mk II]    [Canon C100 Mk II]    [Canon C100 Mk II]
   (Camera 1)            (Camera 2)            (Camera 3)
        │                      │                      │
      HDMI                   HDMI                   HDMI
        │                      │                      │
        ▼                      ▼                      ▼
 [HDMI → SDI Converter] [HDMI → SDI Converter] [HDMI → SDI Converter]
        │                      │                      │
        └──────────── SDI Runs ┼──────────────────────┘
                               │
                               ▼
 [HyperDeck Studio Mini] [HyperDeck Studio Mini] [HyperDeck Studio Mini]
        │                      │                      │
     HDMI Out               HDMI Out               HDMI Out
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               │
                               ▼
                ┌──────────────────────────┐
                │  ATEM Television Studio  │
                │    Pro Studio HD         │
                │    (Video Switcher)      │
                └─────────────┬────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
      [Multiview Monitor]        [Program Out (HDMI)]
                                            │
                                            ▼
                                     [HDMI Splitter]
                                      │           │
                                      ▼           ▼
                               [AJA U‑TAP]   [AJA U‑TAP]
                                (USB Video)   (USB Video)
                                      │           │
                                      ▼           ▼
                               [Desktop PC]   [Laptop]
                              (OBS → Stream)  (Clean Recording)
```

**Key points:**

- Three **Canon C100 Mark II** cameras capture the service from different angles.
- Camera HDMI outputs are converted to **SDI** for long cable runs across the room.
- Each SDI feed passes through a **Blackmagic HyperDeck Studio Mini** for local recording and format conversion back to HDMI.
- All three feeds enter the **ATEM Television Studio Pro Studio HD** video switcher.
- The ATEM operator (or automation) selects which camera is "live" at any moment.
- The **Multiview Monitor** shows all inputs and program/preview on a single screen.
- **Program output** is split via an HDMI splitter to two **AJA U‑TAP** USB capture devices:
  - **Desktop PC** — feeds into OBS Studio for the livestream
  - **Laptop** — records a clean, uncompressed copy of the broadcast

## ProPresenter & Graphics Overlay

```text
                ┌──────────────────────────┐
                │  Mac Studio              │
                │  (ProPresenter 7)        │
                └─────────────┬────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
[Audience Display x2]  [Confidence Monitor]    (NDI Output)
                                                    │
                                              (Network / LAN)
                                                    │
                                                    ▼
                                             [Desktop PC]
                                            (OBS NDI Source
                                             → Graphics Overlay)

                  ┌───────────────────────────────────────┐
                  │  Mac Studio                           │
                  │                                       │
                  │  ┌─────────────┐    ┌─────────────┐  │
                  │  │ ProPresenter│    │ ProPresenter │  │
                  │  │      7      │    │ Automations  │  │
                  │  │             │    │  (Python)    │  │
                  │  └──────┬──────┘    └──────┬───────┘  │
                  │         │                  │           │
                  │         │  REST API        │           │
                  │         │ (localhost:56955) │           │
                  │         └──────►───────────┘           │
                  │                    │                   │
                  └────────────────────┼───────────────────┘
                                       │
                                       │ OBS WebSocket
                                       │ (:4455)
                                       │
                                       ▼
                                ┌─────────────┐
                                │  Desktop PC │
                                │ (OBS Studio)│
                                └─────────────┘

  Operator clicks sermon bumper video in ProPresenter
       → Automations detects playback via REST API
       → Triggers OBS scene switch over WebSocket
```

**Key points:**

- **ProPresenter 7** runs on a Mac Studio and drives all on‑screen graphics: lyrics, sermon slides, announcements, and video playback.
- Two **audience‑facing displays** show lyrics and graphics to the congregation.
- A **confidence monitor** (stage‑facing) shows the current and next slides to the worship leader and pastor.
- ProPresenter sends its output over **NDI** (Network Device Interface) across the LAN to the Desktop PC, where OBS composites it as a graphics overlay on top of the camera feed.
- **ProPresenter Automations** is a Python service running on the Mac Studio. It watches ProPresenter's REST API for video playback — when the operator clicks a sermon bumper video in ProPresenter after worship, the service automatically tells OBS to switch to the corresponding scene (e.g., "SermonBumper") via WebSocket. No manual OBS interaction needed.

## Network

All production equipment shares a wired gigabit LAN:

```text
[Netgear Gigabit Switches]
        │
        ├──► Desktop PC        (OBS, Waves, stream output)
        ├──► Mac Studio        (ProPresenter, NDI source)
        ├──► Laptop            (clean recording)
        ├──► ATEM Switcher     (network control)
        └──► Other devices     (HyperDecks, etc.)
```

| Device | Key Ports | Role |
|--------|----------|------|
| Desktop PC | OBS WebSocket `:4455` | Livestream encoding, audio processing, OBS |
| Mac Studio | ProPresenter API `:56955` | Presentation graphics, NDI output |
| ATEM Switcher | — | Video switching |
| Laptop | — | Clean recording |

## Lighting

```text
[ADJ DMX Operator 192] ────► [Stage Lighting Fixtures]
```

Stage lighting is controlled by a standalone **ADJ DMX Operator 192** console via DMX. It is **not networked** — lighting is operated manually and independently from the rest of the production system.

## Power & Redundancy

All critical production equipment is protected by **UPS battery backup**:

```text
[UPS Battery Backup]
        │
        ├──► Behringer X32 (mixer)
        ├──► Behringer S32 (stage box)
        ├──► ATEM Switcher
        ├──► HyperDeck Studio Minis
        ├──► Desktop PC
        ├──► Mac Studio
        └──► Network Switches
```

This ensures that a brief power interruption does not take down the livestream or lose audio mid‑service.

## How It All Fits Together on Sunday

Here is the end‑to‑end flow on a typical Sunday morning:

1. **Volunteers power on** equipment (UPS keeps critical gear ready).
2. **Audio check** — the sound engineer uses the X32 to set house, monitor, and stream mixes.
3. **Cameras are positioned** and the ATEM operator confirms all three feeds on the multiview monitor.
4. **ProPresenter is loaded** on the Mac Studio with the service plan (lyrics, slides, videos).
5. **Thrive Stream Controller** is opened on the Desktop PC — volunteer clicks **Start Stream**.
6. **OBS goes live** — it composites the ATEM program feed (via AJA U‑TAP) with the ProPresenter NDI overlay and the processed stream audio.
7. **During worship and sermon:**
   - When the operator clicks a sermon bumper video in ProPresenter, ProPresenter Automations auto‑switches the OBS scene
   - The ATEM operator switches cameras manually
   - The laptop silently records a clean copy of the program output
8. **Service ends** — volunteer clicks **End Stream** in Thrive Stream Controller.

## Key Technologies

| Category | Equipment / Software |
|----------|---------------------|
| **Audio Mixer** | Behringer X32 + S32 stage box |
| **In‑Ear Monitors** | Behringer P16 system |
| **Audio Processing** | Waves plugins (Desktop PC) |
| **Cameras** | 3× Canon C100 Mark II |
| **Video Transport** | HDMI → SDI converters, SDI cabling |
| **Recording** | 3× Blackmagic HyperDeck Studio Mini |
| **Video Switching** | Blackmagic ATEM Television Studio Pro Studio HD |
| **Capture** | 2× AJA U‑TAP (USB HDMI capture) |
| **Presentation** | ProPresenter 7 on Mac Studio |
| **Streaming** | OBS Studio on Desktop PC |
| **Stream Management** | Thrive Stream Controller (Docker, .NET/React) |
| **Scene Automation** | ProPresenter Automations (Python) |
| **Lighting** | ADJ DMX Operator 192 |
| **Network** | Netgear gigabit switches |
| **Power** | UPS battery backup |
