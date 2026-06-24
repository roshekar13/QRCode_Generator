# CMU Booth Photo System — Spring 2025

An interactive photo-and-QR-code system built for a CMU Spring Carnival booth. Visitors press a physical button, get a 3-second countdown, and receive their photo as a scannable QR code for instant download to their phone.

## How it works

The system runs across three Raspberry Pi nodes connected on a local Wi-Fi network:

```
[Visitor presses button]
        │
        ▼
  Pi 5 (ScaryFaceTracker)          ← camera capture node
  • 3-second countdown
  • Captures photo via OpenCV
  • Computes a "scary score" (face/hand/mouth analysis via MediaPipe)
  • POSTs image to capture server
        │
        ▼
  Pi Zero (QRCode)                 ← capture + relay node
  • Receives uploaded image
  • Generates a QR code linking to a download page
  • Notifies display Pi of the new QR URL
        │
        ▼
  Pi Zero / display Pi (Display)   ← e-ink display node
  • Receives the QR URL
  • Renders QR as BMP (800×480)
  • Drives the Waveshare e-ink display via C (`./epd`)
  • Visitor scans QR → download page → saves photo
```

Photos expire automatically after 30 minutes.

## Repository layout

```
ScaryFaceTracker/        # Pi 5 — button + camera node
  app.py                 #   Flask app: GPIO button → countdown → capture → upload
  static/                #   Frontend assets

QRCode/                  # Pi Zero — upload + QR generation node  
  app.py                 #   Flask app: receives image, generates QR, notifies display Pi
  index.html / style.css #   Web UI

Display/                 # Display Pi — e-ink driver node
  booth-local-server/
    app.py               #   Flask app: receives QR URL, generates BMP, triggers e-ink
    epd                  #   Compiled C binary — drives the Waveshare e-ink panel
  c/                     #   C source for e-ink display driver (Waveshare EPD library)
  eink/                  #   Additional e-ink utilities
  grok/                  #   Experimental / prototype code
```

## Setup

### Requirements

- Raspberry Pi 5 (camera node) + Raspberry Pi Zero(s) (QR/display nodes)
- Waveshare e-ink display (7.5" V2, 800×480, connected to display Pi via SPI)
- Physical button wired to GPIO pin 7 on the Pi 5 (pull-down, active-high)
- All three Pis on the same Wi-Fi network

### Python dependencies

Each node runs a Flask server. Install on the respective Pi:

**ScaryFaceTracker (Pi 5):**
```bash
pip install flask opencv-python mediapipe RPi.GPIO requests
```

**QRCode (Pi Zero):**
```bash
pip install flask flask-cors qrcode pillow requests qrcode-terminal
```

**Display (display Pi):**
```bash
pip install flask flask-cors qrcode pillow
```

### Configuration

Update the hardcoded IP addresses to match your network before deploying:

| File | Variable | Description |
|------|----------|-------------|
| `ScaryFaceTracker/app.py` | `upload_url` (line ~100) | IP of the QRCode Pi Zero |
| `QRCode/app.py` | `QR_DISPLAY_PI` | IP of the display Pi |
| `Display/booth-local-server/app.py` | `get_server_url()` | Returns your Render deploy URL for external QR links |

All servers listen on port **3000**.

### Building the e-ink driver (display Pi)

The `Display/c/` directory contains the Waveshare EPD C library. To build:

```bash
cd Display/c
# Install bcm2835: download from http://www.airspayce.com/mikem/bcm2835/
# or use the included bcm2835-1.60.tar.gz
tar xzf ../bcm2835-1.60.tar.gz
cd ../bcm2835-1.60 && ./configure && make && sudo make install
cd ../c && make
cp booth-display ../booth-local-server/epd
```

The Flask app in `Display/booth-local-server/` calls `sudo ./epd` whenever a new QR code is ready.

### Running

Start each server on its respective Pi:

```bash
# Pi 5 (ScaryFaceTracker)
cd ScaryFaceTracker && python app.py

# Pi Zero (QRCode)
cd QRCode && python app.py

# Display Pi
cd Display/booth-local-server && python app.py
```

### Optional: Render deployment

The QRCode server optionally mirrors images to a Render-hosted instance (`qrcodegeneration2.onrender.com`) so that QR codes work outside the booth's local network. You can replace this with any publicly accessible server or remove the Render-specific logic and use only local-network links.

## Scary Score

The Pi 5 runs a lightweight computer-vision scorer on each captured frame using MediaPipe:

- **Hands above face** — +5 pts per key point
- **Finger spread** — bonus up to +10 pts for wide spread (>100px)
- **Mouth opening** — bonus proportional to jaw drop (>20px opening)

Score is clamped and normalized to 0–100. It's displayed on the booth's web UI but does not affect the photo or QR code.
