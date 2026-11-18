# Samyukta Hasta PoC - train -> recognise teacher -> save JSON -> recognise student -> compare

```
samyukta_poc/
├── run.py                      # single entry point (python run.py <command>)
├── server.py                   # live webcam FastAPI server
├── config.yaml                 # your settings (paths, ROI, thresholds)
├── requirements.txt
├── frontend/                   # live webcam browser UI
│   ├── index.html
│   ├── style.css
│   └── app.js
├── reference/
│   ├── samyukta_hasta.mp4      # <- PUT YOUR TEACHER VIDEO HERE
│   └── teacher_segments.json   # your 23 segments (Dola/Utsanga are dropped automatically)
├── data/samyukta/<Class>/*.jpg # <- dataset goes here (step 2)
├── samyukta/
│   ├── model.py  dataset.py  train.py     # EfficientNet-B0 transfer learning
│   ├── infer.py                           # classifier -> probabilities + embeddings
│   ├── video.py  hand_crop.py             # frame sampling, two-hand crop (MediaPipe -> ROI -> full)
│   ├── labels.py                          # segment name <-> dataset class matching
│   ├── segmentation.py                    # smoothing + frame predictions -> movements (student)
│   ├── pipeline.py                        # teacher / student recognition -> JSON, comparison
│   ├── compare.py  report.py              # alignment, scoring, timeline plot
│   └── cli.py  utils.py
├── scripts/prepare_hf_dataset.py  scripts/download_hand_model.py
├── tests/                      # test_pure.py, test_e2e_stub.py (no torch / dataset needed)
└── outputs/                    # created by the run
    ├── model/    model.pt  class_names.json  training_report.json  classification_report.txt  confusion_matrix.png
    ├── teacher/  teacher_reference.json  crops/
    └── students/<name>/  student_performance.json  comparison_report.json  comparison_timeline.png  crops/
```

## Steps to run

**1. Install** (Python 3.10+, GPU optional but training on CPU is slow)
```
pip install -r requirements.txt
python tests/test_pure.py && python tests/test_e2e_stub.py     # sanity check, ~10 s
```

**2. Get the dataset into `data/samyukta/<ClassName>/*.jpg`** (double-hand classes only)
- Option A, HuggingFace copy (columns: image, label, gesture_type): `python run.py prepare-hf`
- Option B, the GitHub/original download: arrange the Samyukta (double-hand) images into one folder per class.
  Any path containing the word "double" is picked automatically; otherwise set `dataset.classes_file`.
- Dataset licence is CC-BY-SA-4.0: credit it and share derived data alike.

**3. Check data and segment mapping** - must show 21 classes, and Dola/Utsanga excluded
```
python run.py check-data
```
If a segment maps to the wrong class (or an expected one is excluded), fix it under `teacher.aliases` in `config.yaml`.

**4. Train**
```
python run.py train
```
Writes `outputs/model/model.pt` plus test accuracy, per-class report and confusion matrix.
Read `classification_report.txt` before trusting anything downstream.

**5. Recognise the teacher -> `teacher_reference.json`**
```
python run.py teacher --roi 0.5,0.15,0.5,0.7
```
`--roi` is the normalised box (x,y,w,h) where the dancer's hands are. Open `outputs/teacher/crops/` to check the crops are
the two hands; adjust and rerun. The command prints where the classifier disagrees with your segment labels.

**6. Recognise a student and compare**
```
python run.py evaluate --video students/student1.mp4 --name student1
```
(or separately: `python run.py student --video ...` then `python run.py compare --student outputs/students/student1/student_performance.json`).
The teacher JSON is reused for every student - teacher recognition never has to be rerun.

## Live webcam frontend

The separate five-round webcam game has its own setup instructions in
[game/README.md](game/README.md). It uses the same trained model and dataset
described above.

### First-time setup

Open PowerShell in the project directory, then run:

```powershell
cd D:\Cdac_internship\Samyukta_hasta_training\samyukta_poc
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Train the model before starting the frontend if `outputs/model/model.pt` does not exist:

```powershell
python run.py train
```

### Start the frontend

From the project directory:

```powershell
cd D:\Cdac_internship\Samyukta_hasta_training\samyukta_poc
.\.venv\Scripts\python.exe server.py
```

Open **http://localhost:8000** in a browser that has webcam access enabled.

To verify the server without opening the page:

```powershell
curl.exe http://localhost:8000/health
```

The health response should contain `"ready": true`.

### Stop the frontend

Return to the PowerShell window running the server and press `Ctrl+C`.

To stop it from another PowerShell window, find the process using port 8000 and
stop the matching Python process:

```powershell
Get-NetTCPConnection -LocalPort 8000 -State Listen |
  Select-Object OwningProcess
Stop-Process -Id <OwningProcess>
```

The page sends downscaled JPEG frames to the backend at 200 ms by default. The
`HandCropper` detects the hands and the EfficientNet classifier returns the top
five mudras. Hand detection, the crop preview, latency, FPS, and a five-frame
temporal probability average are shown in the UI.

Required files:
- `outputs/model/model.pt` from `python run.py train`
- `hand_landmarker.task` in the project root, or `models/hand_landmarker.task`
- `fastapi`, `uvicorn`, and `python-multipart` from `requirements.txt`

## Repository contents

The repository contains the training/inference code, configuration, tests,
web-game source, and dependency lockfiles. It intentionally does not include
the dataset, recorded videos, generated outputs/model weights, MediaPipe task
binary, local virtual environments, installed Node packages, or `.env` files.
Follow the setup steps above to obtain the dataset and train the model; run
`python scripts/download_hand_model.py` to download the MediaPipe task model.
Install the game JavaScript dependencies with `npm ci` in both `game/backend`
and `game/web`.

Install and run:

```cmd
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe server.py
```

Useful checks while the server is running:

```cmd
curl.exe http://localhost:8000/health
curl.exe -I http://localhost:8000/
```

`/health` should report `"ready": true` and list all 21 classes. If the browser
shows `connection error`, make sure the server is still running and open the
page through `http://localhost:8000`, not directly through `frontend/index.html`.

## What gets saved
- `teacher_reference.json`: video + model metadata, per-movement label (human segment label and classifier prediction, both kept),
  mean probabilities, mean embedding, and every frame (timestamp, top-3, all class probabilities, hands detected, crop path).
- `student_performance.json`: same schema, but movements are discovered from the video (no timings needed).
- `comparison_report.json`: per teacher movement -> `correct | wrong_mudra | missing`, plus extra student movements,
  start-timing error, tempo-adjusted duration ratio, hand-shape similarity, text feedback, overall score, frame-level DTW similarity.

## How comparison works
1. Student frames are smoothed, low-confidence/no-hand frames dropped, short blips removed, same-label runs merged -> movements.
2. Teacher and student label sequences are aligned (Needleman-Wunsch) -> correct / wrong / missing / extra.
3. Tempo = median duration ratio of the correct pairs, so a uniformly slower student isn't punished twice.
4. Timing = start offset relative to the previous adjacent correct mudra; duration = tempo-adjusted ratio;
   hand shape = cosine of mean classifier embeddings. Anything that can't be assessed is left out and weights renormalised.
5. Score = mean over teacher movements (0 for wrong/missing) minus a small penalty per extra movement. Weights are in `config.yaml`.

## Known limitations (be aware before demoing)
- **Hand shape score is a proxy** (classifier embeddings), not finger-level geometry. Add HaMeR / 3D joints later for real feedback.
- **Domain gap**: the dataset is studio photos; your videos are not. Expect lower accuracy on video than the test set shows.
  The first fix is a good ROI/crop; then fine-tune on crops taken from your own labelled frames.
- **Random split is optimistic** unless the file names carry a volunteer id (set `dataset.group_regex`).
- The teacher video is a YouTube screen recording: keep the ROI tight so thumbnails/end-screen cards aren't read as hands.
- Download `hand_landmarker.task` into the project root; the code also accepts `models/hand_landmarker.task`.
- MediaPipe >= 0.10.3x removed `mp.solutions`; then run `python scripts/download_hand_model.py` once (needs internet).
  Without any detector the code falls back to ROI/full-frame crops and records that in the JSON (`crop_method`).
- Dola and Utsanga are not in the dataset and are excluded; they need their own labelled images to be supported.
- Class names come from the folder names, so spellings follow the dataset, not your video.
