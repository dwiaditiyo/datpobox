# Copilot Instructions for datpobox

- This repository is a small single-file Python Qt application: `photobooth.py`.
- The main logic is in the `PhotoBooth` class. It uses `PyQt5` for UI, `cv2` for webcam capture and image I/O, and `numpy` to assemble the collage.
- The current runnable path is webcam-based; the Canon DSLR/GPhoto2 integration is commented out and should not be assumed functional.

## What to change
- Preserve the app state around `self.photos`, `self.capturing`, and button enable/disable logic.
- `update_preview()` is the live webcam refresh loop driven by `QTimer`; do not break that pattern when adding UI updates.
- `capture_photo()` expects up to 4 images and enables `Save Collage` and `Print` only after 4 captures.
- `save_collage()` builds a 2x2 collage from the first four captured frames and writes `collage_<timestamp>.jpg`.
- `print_collage()` is currently a simulated print action; it calls `save_collage()` and only updates status text.

## Run / debug
- Use Python 3 and install dependencies manually: `PyQt5`, `opencv-python`, `numpy`, `Pillow`.
- Run the app with:
  - `python3 photobooth.py --use-webcam`
- There is no additional build or test harness in this repository.

## Project-specific notes
- There is no package structure or configuration file; everything is self-contained in `photobooth.py`.
- The app saves temporary images as `photo_1.jpg` through `photo_4.jpg`.
- The `closeEvent()` handler releases the webcam and will also exit the camera if DSLR support is later restored.
- Do not add unrelated tooling; focus on improving the existing photobooth flow.

## Useful file
- `photobooth.py` - single source of truth for UI, capture, collage assembly, and app startup.
