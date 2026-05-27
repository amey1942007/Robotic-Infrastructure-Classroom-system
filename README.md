# SURGE — The Lecturer-Following Camera

> A small box on the ceiling that listens for the teacher's voice, points a
> camera at them, and quietly puts the whole lecture on Google Drive.

---

## What is this?

Imagine you walk into a classroom. A small round pod sits on the ceiling.
The teacher starts talking. The pod's camera quietly turns to face them.
As the teacher walks across the room, the camera follows. Every word and
every slide is recorded and waiting on Google Drive by the time class ends.

That's SURGE. No camera operator. No buttons to press. It just works.

---

## How it works — the simple version

```
 Teacher speaks   →   4 tiny mics hear the sound
                  →   The Pi figures out the direction
                  →   The camera turns to that direction
                  →   The video goes live on the internet
                  →   The recording lands on Google Drive
```

That's the whole story. Everything below is the detail.

---

## The five jobs the pod does

| # | Job | What's doing it |
|---|---|---|
| 1 | **Hear the room** | Four tiny MEMS microphones around the rim |
| 2 | **Find the voice** | A small AI (Silero VAD) ignores silence; an algorithm called SRP-PHAT figures out *where* the speech came from |
| 3 | **Move the camera** | Two little servo motors — one for left/right, one for up/down |
| 4 | **Stream the lecture** | The Pi's built-in video chip encodes it; MediaMTX serves it like a TV channel |
| 5 | **Save the recording** | Every 15 minutes a chunk is uploaded to Google Drive and deleted from the Pi |

---

## What's inside the pod

| Part | What it does |
|---|---|
| Raspberry Pi 5 | The brain |
| RPi Camera Module 3 | The eye |
| Pan-Tilt HAT | The neck — lets the camera look around |
| 4× MEMS microphones | The ears, fixed in a ring around the pod |
| 27W USB-C PSU | The power supply |
| Active cooler | A little fan so the Pi doesn't overheat during long lectures |
| 32GB microSD | Holds the operating system; recordings don't live here |
| 3D-printed shell | Holds everything together, ~11 cm wide |

Cost so far: **about ₹29,500.**

A separate 7" touch display sits on a desk — that's the *monitoring* screen.
It is **not** mounted on the pod. The pod itself has no screen.

---

## How you watch it

- **In VLC, on the same network:** open `rtsp://<pi-ip>:8554/lecture`
- **In a browser:** open `http://<pi-ip>:8888/lecture`
- **From anywhere in the world:** turn on Tailscale and use the same links

---

## Where the recordings go

```
Google Drive
└── LecturerTracker/
    └── 2026-05-27/
        ├── 09-00.mp4
        ├── 09-15.mp4
        ├── 09-30.mp4
        └── ...
```

Each file is a 15-minute slice of the lecture. The Pi's SD card never fills up because every chunk is deleted after Google Drive confirms it was uploaded.

About **900 MB per hour** of lecture. **₹130/month** for Google One 200 GB covers ~44 days of lectures.

---

## A few choices we already made (and why)

- **Mics don't move, only the camera does.** If the mics rotated with the camera, the "direction the voice came from" would keep changing meaning. Fixed mics keep the geometry honest.
- **Wired ethernet, not WiFi.** Classroom WiFi gets crowded and the stream stutters. A cable just works.
- **No hard drive, no SSD.** Recordings stream to the cloud the moment they're finished. Less hardware, less to fail.
- **Power the servos from the HAT, not from the Pi's pins.** The Pi's 5V pins can only push so much current; the servos want more. Skipping this rule fries things.

---

## What's done, what's left

**Done**
- [x] Bill of materials finalised and ordered
- [x] Software pipeline written (`vad_srp_phat_pipeline.py`)
- [x] Hardware architecture locked

**To do**
- [ ] 3D-print the housing
- [ ] Order the microphones (ICS-43434 if available, INMP441 otherwise)
- [ ] Write the systemd service files so everything boots on its own
- [ ] Hook up Google Drive (rclone) and Tailscale
- [ ] Design what the monitoring display actually shows

---

## Things we might add later (not now)

- **Face detection** to double-check the camera is aimed right
- **Voice fingerprinting** so the camera ignores students asking questions
- **Room calibration** for buildings with weird echo
- **Power over Ethernet** so the pod only needs one cable instead of two
- **Auto-zoom** so the teacher fills the frame nicely

---

## A note on the name

**SURGE** — because the project surged from "we need a way to record lectures" to a full ceiling-mounted tracking pod in one semester.

---

*Last updated: May 2026.*
