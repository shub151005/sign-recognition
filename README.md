# Real-Time Sign Language Recognition (ASL fingerspelling prototype)

This repository contains a webcam application that tracks hand landmarks and attempts to recognize a subset of static American Sign Language (ASL) fingerspelling poses. It is a prototype for experimentation and demonstration. It is **not** a validated ASL translator, does not recognize words or sentences as sign language, and should not be used as an accessibility or interpretation service.

## Will it run on my laptop?

**Yes, the Python application should run on a Ryzen 7 laptop with 16 GB RAM and an RTX 3050 4 GB.** It supports CPU inference, and the classifier is small. The GPU is optional; 4 GB VRAM is ample for this landmark classifier if your installed PyTorch has CUDA support, but hand detection and camera/video work still consume CPU. Expect actual speed to depend on your Ryzen model, camera, lighting, and MediaPipe/PyTorch versions. A built-in/USB webcam is required. The project has not been benchmarked on every Ryzen 7/3050 configuration.

## What is in this folder

| File | Purpose |
| --- | --- |
| `main.py` | Main runnable Python webcam application. Contains landmark features, a rule-based classifier, a small PyTorch classifier, smoothing, UI, recording and exports. |
| `main.cpp` | Separate C++ prototype using OpenCV and MediaPipe. It is not wired up as a working build in the current folder. |
| `CMakeLists.txt` | CMake configuration for that C++ prototype; currently references a missing `sign_language_recognition.cpp` instead of `main.cpp`. |
| `sign.png` | Image asset; the Python app does not load it. |
| `LICENSE` | MIT license. |

At first Python launch, the app creates `models/` and `output/` directories. These are runtime-generated and are not currently checked into the repository.

## Features in the Python app

- Webcam capture on camera index 0, requested at 1280×720, in a background thread.
- MediaPipe Hands tracking (configured for up to two hands); the app classifies only the first detected hand.
- 21 three-dimensional landmarks and normalized shape features.
- Rule-based pose heuristics for some static letters, with a neural-network fallback when the heuristic returns `UNKNOWN`.
- Prediction confidence display and temporal majority/stability filtering before adding a character.
- Sentence buffer with capitalization at sentence start/after spaces, delete gesture label, manual clear, automatic space after the hand leaves view, clipboard copy and text export.
- Dark OpenCV HUD with hand box, skeleton/points toggles, prediction, confidence, stability, FPS, status, and sentence/history.
- Screenshot capture, annotated video recording, session log, and JSON session summary.
- CPU inference with automatic CUDA selection when available.

The code includes letter labels A–Z plus `SPACE`, `DELETE`, `NOTHING`, `UNKNOWN`, and `IDLE`, but the heuristic rules only explicitly identify a subset. Several static poses are visually similar, and movement-based letters such as J and Z cannot be reliably inferred from a single-frame pose. Gesture labels and confidence values are heuristic estimates, not calibrated probabilities.

## Requirements

- Windows, macOS, or Linux with Python supported by your installed versions of MediaPipe and PyTorch (Python 3.10 or 3.11 is a practical starting point; package compatibility changes over time).
- Webcam and a graphical desktop session (OpenCV opens a window).
- Internet access on first run if required Python packages are missing, because `main.py` attempts to install them with pip.
- A writable project directory for model, recordings, screenshots, and logs.

The script checks/imports OpenCV, MediaPipe, NumPy, PyTorch, scikit-learn, Pillow, SciPy, Matplotlib, and tqdm. Several of these are not used by the current execution path (SciPy, Matplotlib, tqdm, Pillow, and `queue` are unnecessary for the visible app flow), so automatic installation is broader than needed. Prefer a virtual environment to avoid altering system Python.

## Start the Python app

From PowerShell in this project directory:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install opencv-python mediapipe numpy torch scikit-learn pillow scipy matplotlib tqdm
python main.py
```

If `py -3.11` is unavailable, use an installed Python version compatible with the current MediaPipe and PyTorch wheels. If PowerShell blocks activation, use `\.venv\Scripts\python.exe` directly for pip and launch commands. The app also attempts to install missing packages itself, but an explicit environment makes setup and troubleshooting easier.

On first run, the program generates 400 **synthetic** landmark examples per label and trains for 40 epochs, then saves `models/gesture_classifier.pt`. This can take a while and requires no internet once dependencies are installed. Later launches load that checkpoint. This synthetic training is not training on a real ASL dataset and does not establish real-world recognition accuracy. The rule-based classifier is tried first and usually takes precedence.

## Use the webcam window

1. Start the app and allow desktop camera access if prompted.
2. Face the camera toward one hand with the palm visible, keep the hand well lit and inside the image, and try static fingerspelling poses.
3. Watch the current label and confidence. Hold a pose steady; the app requires a stable prediction before committing it once per hold.
4. Move the hand out of view briefly to trigger a space after a committed word.
5. Use the keys below while the OpenCV window is focused.

| Key | Action |
| --- | --- |
| `P` | Pause or resume processing. |
| `S` | Save the current annotated frame to `output/screenshots/`. |
| `R` | Start or stop annotated MP4 recording in `output/recordings/`. |
| `L` | Toggle landmark points. |
| `K` | Toggle hand skeleton lines. |
| `H` | Toggle the HUD. |
| `C` | Clear the sentence. |
| `X` | Copy the sentence to the system clipboard. |
| `E` | Export the sentence as a `.txt` file in `output/`. |
| `Q` | Quit and write the session summary. |

Keys are handled as lowercase OpenCV key codes; they should work without Shift. Clipboard support on Linux may require `xclip`. Recording depends on the OpenCV video codec available on the system. The app mirrors the camera image before processing/display.

## Files created while running

```text
models/gesture_classifier.pt       generated PyTorch checkpoint
output/screenshots/*.png           saved annotated still images
output/recordings/*.mp4            saved annotated video
output/sentence_*.txt               exported sentence
output/logs/session.log             application log
output/logs/session_summary_*.json  per-session statistics and final sentence
```

The session JSON includes duration, average FPS, frame count, recognition event count, most recognized letters, and the final sentence. Logs and captured media may contain sensitive camera content; manage these files as you would other personal recordings.

## How recognition works (developer notes)

1. `WebcamStream` reads the camera in a daemon thread and exposes the latest frame.
2. `SignLanguageRecognitionApp.process_frame` mirrors the frame, converts it to RGB, and calls MediaPipe Hands.
3. The first detected hand's 21 normalized landmarks are converted to five finger-extension estimates, thumb/pinch/spread relationships, and a 72-value feature vector.
4. `classify_gesture_rule_based` applies hand-written threshold rules. If it returns a label other than `UNKNOWN`, that result is used directly. Otherwise the features are standardized using the generated checkpoint and passed to a small fully connected PyTorch classifier.
5. `PredictionSmoother` uses a 12-item rolling majority buffer, 0.5 confidence threshold, at least eight stable updates, and 0.6 majority ratio. A label is committed once per continuous hold.
6. `SentenceBuilder` appends the committed letter, handles `SPACE`/`DELETE`, and the app renders a HUD and stores session artifacts.

The vector consists of 63 landmark coordinates + 5 curl ratios + 4 adjacent fingertip distances = **72 values**. Use the implementation in `extract_feature_vector` as the source of truth if changing the model or docs.

Important source-level caveats:

- The synthetic generator creates landmarks from simplified finger curl/spread templates, not camera recordings, and includes crude/overlapping labels. Its validation accuracy measures only a held-out split of that synthetic generator.
- The rule-based implementation does not cover all 26 letters or reliable dynamic recognition. `SPACE` is principally generated by hand removal; the current rule set does not emit `DELETE`.
- `max_num_hands=2`, but the recognition path uses only `multi_hand_landmarks[0]`; it does not fuse two hands.
- The Python window has HUD toggles, not a separate interactive analytics dashboard. Some README claims in older copies may describe aspirational behavior.
- Python dependencies are installed at module import time, before the application starts. Installation failures stop startup.
- PyTorch checkpoint loading currently uses `weights_only=False`; only load checkpoints you trust.
- The project has no dependency lock file, automated tests, packaging metadata, or camera selection command-line option.

## C++ prototype status

`main.cpp` contains a separate MediaPipe/OpenCV implementation with a similar static-pose heuristic and UI. It does not use the Python PyTorch classifier. The current `CMakeLists.txt` refers to absent `sign_language_recognition.cpp`, so a normal CMake configure/build from this checkout will not build the target. It also requires a MediaPipe C++ installation exported as a CMake package, OpenCV development files, and generated/build integration matching that installation. Treat this path as incomplete source, not a supported way to run the app.

## Troubleshooting

- **Camera not detected / black screen:** close other camera apps, check OS camera permissions, confirm webcam works, and try setting the camera source in `WebcamStream()` if the device is not index 0.
- **Slow or laggy:** lower the requested camera size in `WebcamStream` (currently 1280×720); close GPU/CPU-heavy apps. The small classifier is not usually the main workload.
- **PyTorch reports CPU:** this is supported. CUDA acceleration requires a CUDA-enabled PyTorch build and compatible NVIDIA driver/runtime; the installed default wheel may be CPU-only.
- **Import/install error:** use a clean supported Python environment and install packages individually to identify compatibility issues. MediaPipe/PyTorch version and Python wheel support can change.
- **Poor labels:** use bright even light, show the palm clearly, avoid cropping fingers, hold still, and remember this prototype has no real-data-trained model.
- **No screenshot saved from key:** focus the OpenCV window and press `S`.
- **Recording file is missing/empty:** start recording with `R`, ensure frames are being processed, stop with `R` or quit cleanly, and check codec support.

## Contributors

This project was developed collaboratively. We appreciate the contributions and efforts of everyone involved. Thanks to everyone who contributed to this project!

## License and attribution

The repository is licensed under the MIT License. The source identifies `tubakhxn` as creator; the license names Tuba Khan as copyright holder.
