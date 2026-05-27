PROJECT: Robotics Infrastructure Lecturer Tracking Camera System
LAST UPDATED: May 2026

═══════════════════════════════════════════════════════════
CONFIRMED BOM (robu.in order placed)
═══════════════════════════════════════════════════════════

| Component        | Part                                    | Price (INR) |
|------------------|-----------------------------------------|-------------|
| Compute          | Raspberry Pi 5 4GB RAM                  | ₹12,331     |
| Camera           | RPi Camera Module 3                     | ₹3,099      |
| Pan/Tilt         | Pan-Tilt HAT for RPi and Jetson Nano    | ₹2,160      |
| Display          | 7" Official RPi Touch Display 2         | ₹7,229      |
| Power            | Official 27W USB-C PD PSU for RPi 5     | ₹1,235      |
| Storage          | SanDisk Ultra microSD 32GB 120MB/s (A1) | ₹1,619      |
| Cooling          | Official RPi 5 Active Cooler            | ₹488        |
| Microphones      | 4× ICS-43434 MEMS (preferred, sourcing) | ~₹800 TBD   |
| Mic fallback     | 4× INMP441 (drop-in replacement)        | ~₹960       |
| Housing          | 3D printed PLA                          | ~₹300       |
| Misc             | Wires, standoffs, connectors            | ~₹300       |
| TOTAL            |                                         | ~₹29,561    |

IMPORTANT NOTES ON BOM:
- SanDisk Ultra is A1 not A2 — acceptable since recordings go to cloud not SD
- ICS-43434 currently unavailable in India — INMP441 is confirmed drop-in 
  replacement (same I2S interface, same wiring, same code, no changes needed)
- Always prefer ICS-43434 if it becomes available
- Microphones not yet ordered — sourcing separately
- Active Cooler is essential — pipeline runs CPU hard during lectures

═══════════════════════════════════════════════════════════
SYSTEM ARCHITECTURE
═══════════════════════════════════════════════════════════

CONCEPT:
A ceiling-mountable self-contained pod that automatically tracks a lecturer
using audio localisation and controls a pan/tilt camera to follow them,
with live streaming and cloud recording.

PHYSICAL DESIGN:
- Single rigid 3D printed housing (~11cm diameter cylinder)
- 4x MEMS mics fixed in a circular ring on the housing (NEVER rotate)
- Mic ring diameter: 8.5cm, adjacent mic spacing: ~6cm
- Camera sits OUTSIDE and BELOW the housing on the pan/tilt mechanism
- Only the camera head rotates — mic geometry is permanently fixed
- Ceiling mount bracket with two screw holes at top
- Single cable to wall (power + ethernet)

KEY DESIGN DECISIONS (do not re-debate these):
- Mics are FIXED to housing, camera rotates independently
- No encoder needed on servos — position saved to disk file instead
- Servo power goes to HAT VIN terminal directly, NOT from Pi GPIO rail
- No external SSD — recordings stream to cloud
- Display is a separate monitoring station, NOT mounted on ceiling pod

═══════════════════════════════════════════════════════════
AUDIO LOCALISATION STACK
═══════════════════════════════════════════════════════════

VAD: Silero VAD
- Gates the localisation pipeline — only runs SRP-PHAT when speech detected
- Requires 16kHz mono audio, 512 sample chunks
- Threshold: 0.5 (adjustable)

LOCALISATION: SRP-PHAT (Steered Response Power - PHAT weighted)
- NOT GCC-PHAT — SRP-PHAT runs all mic pairs simultaneously voting on
  a common angle grid, far more robust in reverberant classrooms
- d < λ/2 constraint does NOT apply to SRP-PHAT (applies to beamforming)
- Practical mic spacing limit for SRP-PHAT in classroom: ~10cm
- Search grid: azimuth -90° to +90°, elevation 0° to 60°
- All 6 mic pairs (4 choose 2) vote on every candidate direction
- Outputs: azimuth (deg), elevation (deg), confidence (0-1)

SMOOTHING: 2D Kalman Filter
- State: [azimuth, elevation, az_velocity, el_velocity]
- Predict every audio frame, update only when speech detected
- Tuned for smooth over fast (lecture recording priority)
- High R (measurement noise) relative to Q (process noise)

MIC GEOMETRY:
- Circular ring, 8.5cm diameter, radius = 0.0425m
- M0: North  (0,    +R)
- M1: East   (+R,   0 )
- M2: South  (0,    -R)
- M3: West   (-R,   0 )
- Wired as 2x stereo I2S pairs to Pi GPIO
- L/R select pin determines channel within each pair

I2S WIRING (Pi 5 GPIO):
- Pin 12 (BCK)  → SCK  on all 4 mics (shared)
- Pin 35 (LRCK) → WS   on all 4 mics (shared)
- Pin 38 (DIN)  → SD   on pair 1 (M0 + M1)
- Pin 40 (DIN)  → SD   on pair 2 (M2 + M3)
- L/R: M0→GND, M1→3.3V, M2→GND, M3→3.3V
- No extra ADC board needed — ICS-43434/INMP441 are native I2S digital mics

═══════════════════════════════════════════════════════════
SERVO + CAMERA CONTROL
═══════════════════════════════════════════════════════════

HARDWARE:
- Waveshare Pan-Tilt HAT (PCA9685 over I2C)
- Pan servo: MG90S (metal gear, channel 1)
- Tilt servo: SG90 (plastic gear, channel 0)
- Library: Adafruit ServoKit (NOT Waveshare bcm2835 — incompatible with Pi 5 Bookworm)

CRITICAL ASSEMBLY NOTE:
- DO NOT assemble servos into bracket before running test code
- Run servos to 0° (home) first, then physically assemble
- Prevents servo jam and gear damage

POSITION PERSISTENCE:
- No encoder needed — last commanded position saved to:
  /var/lib/camera/last_position.json
- On reboot: load saved position, command servo there immediately
- Dead zone: 2° — ignore movements smaller than this to prevent jitter

SERVO POWER WIRING:
- Pi 5 USB-C ← 27W PSU (logic + compute power)
- HAT VIN pin ← 27W PSU directly (servo power, bypasses Pi GPIO rail)
- Common GND between Pi and HAT
- NEVER power servos from Pi GPIO 5V rail (max 1A, servos need up to 1.4A combined)

ANGLE MAPPING:
- Azimuth -90° to +90° → Pan servo PAN_MIN to PAN_MAX (20° to 160°)
- Elevation 0° to 60°  → Tilt servo TILT_MAX to TILT_MIN (ceiling mount inverted)
- Confidence threshold: 0.35 — below this hold position, do not chase noise

═══════════════════════════════════════════════════════════
STREAMING + STORAGE
═══════════════════════════════════════════════════════════

NETWORK: Use ethernet for pod (not WiFi) — classroom WiFi congestion
         degrades stream quality mid-lecture

STREAMING STACK:
- libcamera-vid → hardware H.264 encode (Pi 5 ISP, free compute)
- MediaMTX (rtsp-simple-server) → RTSP + HLS endpoints
- RTSP: rtsp://[pi-ip]:8554/lecture  (VLC, low latency ~1s)
- HLS:  http://[pi-ip]:8888/lecture  (browser, ~3-6s latency)
- Bitrate: 2Mbps, 720p, 25fps
- Recording segments: 15 minutes each → /tmp/recordings/

CLOUD STORAGE: Google Drive via rclone — TWO-PHASE RECORDING

PHASE 1 — DURING LECTURE (fault tolerance):
- rclone watches /tmp/recordings/ via inotifywait
- Uploads completed 15min segments to Drive/LecturerTracker/YYYY-MM-DD/segments/
- Deletes local copy after confirmed upload (SD card never fills)
- If Pi loses power mid-lecture, all uploaded segments are safe in the cloud

PHASE 2 — END OF LECTURE (triggered by "End Lecture" button on 7" display):
- ffmpeg concat merges all segments losslessly (~5 seconds, no re-encode)
- Merged file uploaded to Drive/LecturerTracker/YYYY-MM-DD/
  as Lecture_HH-MM_to_HH-MM.mp4
- segments/ folder deleted from Drive after merge confirmed
- Result: one clean video file per lecture in Google Drive

FOLDER STRUCTURE (final state after merge):
  LecturerTracker/
    2026-05-27/
      Lecture_09-00_to_10-05.mp4
      Lecture_11-00_to_12-00.mp4

TRIGGER METHOD: Manual button on 7" touch display
- Rejected: auto silence detection (breaks between slides = false end)
- Rejected: scheduled timetable (not flexible enough)
- Rejected: SSH command (requires terminal access)

REMOTE ACCESS: Tailscale (WireGuard VPN)
- Free tier, up to 100 devices
- Works through university NAT/firewall, no port forwarding needed
- Pi gets stable Tailscale IP (100.x.x.x)
- Access stream from anywhere: rtsp://[tailscale-ip]:8554/lecture

STORAGE ESTIMATE:
- 720p H.264 @ 2Mbps ≈ 900MB/hour
- 5 lectures/day ≈ 4.5GB/day
- Google One 200GB covers ~44 days of lectures

═══════════════════════════════════════════════════════════
SOFTWARE STACK
═══════════════════════════════════════════════════════════

OS: Raspberry Pi OS Bookworm (64-bit)
    NOTE: bcm2835 and wiringPi do NOT work on Bookworm — use lgpio

PYTHON LIBRARIES:
- sounddevice      — 4-channel I2S audio capture
- numpy / scipy    — SRP-PHAT signal processing
- torch            — Silero VAD model
- adafruit-circuitpython-servokit — PCA9685 servo control
- libcamera / picamera2 — camera capture

SYSTEM SERVICES (all auto-start on boot via systemd):
- lecturer-tracker.service  — main VAD + SRP-PHAT + servo pipeline
- mediamtx.service          — RTSP/HLS stream server
- rclone-upload.service     — cloud upload watcher
- tailscaled.service        — remote access VPN

CODE: Full pipeline written — vad_srp_phat_pipeline.py
- SileroVAD class
- SRPPhatLocalizer class (with precomputed TDOA grid)
- KalmanFilter2D class
- ServoController class (with position persistence)
- LecturerTracker main pipeline (threaded audio queue)

═══════════════════════════════════════════════════════════
FUTURE / OPTIONAL UPGRADES
═══════════════════════════════════════════════════════════

- Vision fusion: YOLOv8-nano face detection for fine tracking confirmation
  (audio gives coarse direction, vision locks precise position)
  Needs: 4GB RAM is sufficient, no compute upgrade required
- Speaker lock-on: SpeechBrain ECAPA-TDNN voiceprint of lecturer
  (ignore student questions, coughing, door slams)
- Acoustic zone calibration: room impulse response measurement at deployment
- PoE++ HAT: replace wall PSU + ethernet with single Cat6 cable
- Motor noise notch filter: servo whine at 1-3kHz, filter before SRP-PHAT
- Auto-framing: crop/zoom so lecturer fills 60% of frame height

═══════════════════════════════════════════════════════════
OPEN DECISIONS
═══════════════════════════════════════════════════════════

- [ ] 3D housing design not yet finalised
- [ ] ICS-43434 availability — check before ordering mic stage
- [ ] systemd service files not yet written
- [ ] Tailscale + rclone not yet configured
- [ ] End-of-lecture merge script not yet written (ffmpeg + rclone cleanup)
- [ ] Monitoring display UI not yet designed (must include "End Lecture" button)

RESOLVED DECISIONS:
- [x] Recording strategy: two-phase (15min safety chunks → single merged file)
- [x] End-of-lecture trigger: manual "End Lecture" button on 7" touch display