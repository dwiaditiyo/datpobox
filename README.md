# Photobooth Touchscreen App

A touchscreen photobooth application for Canon DSLR (1200D/600D) using `pygame`, `gphoto2`, `qrcode`, and `Pillow`.

## What it does

- Idle screen with `Touch to Start`
- Frame selection from 4 PNG overlays
- 3-second countdown
- Canon DSLR capture via `gphoto2`
- Automatic photo resize + frame overlay using Pillow
- QR code generation for a download placeholder link
- Preview screen with final photo and QR code
- `Done` button or auto-reset after 15 seconds

## Files

- `photobooth.py` - main touchscreen photobooth application
- `requirements.txt` - Python dependencies
- `.github/copilot-instructions.md` - AI guidance for this repo
- `photobooth.py.bak` - backup of the original Qt/openCV version

## Dependencies

Install the required Python packages first:

```bash
python3 -m pip install -r requirements.txt
```

You must also have `gphoto2` installed and a Canon DSLR connected.

## Run the app

```bash
cd /Users/datisroot/Desktop/4funcorner/datpobox
python3 photobooth.py
```

If you want to validate the Python file quickly:

```bash
python3 -m py_compile photobooth.py
```

## Notes

- The app creates an `assets/` folder for placeholder frame PNGs and a `captures/` folder for captured images.
- The generated QR code currently encodes a placeholder download link.
- If the camera fails, the app displays an error screen with a `Try Again` button.

## Troubleshooting

- `gphoto2` not found: install it using your package manager, e.g. `brew install gphoto2` on macOS.
- DSLR not detected: ensure the camera is powered on, USB-connected, and set to the correct remote control mode.
- Camera init failure: check `photobooth.py` output for `Camera init failed:` and verify your camera is not locked by another app.
- Pygame display issues: use a desktop session that supports fullscreen or run in a normal window if needed.
- Missing frame assets: the app will generate placeholder PNGs in `assets/` automatically.
