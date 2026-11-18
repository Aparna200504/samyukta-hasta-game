Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\README.md"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\server.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\run.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\frontend\\app.js"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\frontend\\index.html"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\frontend\\style.css"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\pipeline.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\infer.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\hand_crop.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\model.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\segmentation.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\compare.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\config.yaml"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\cli.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\train.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\dataset.py"
}

Read 
Completed with input: {
  "path": "D:\\Cdac_internship\\Samyukta_hasta_training\\samyukta_poc\\samyukta\\utils.py"
}

# Samyukta Hasta: architecture and end-to-end pipeline

There are **two related but separate ways to use the project**:

1. **Live webcam demo:** the browser sends individual camera frames to the Python backend, which returns a mudra prediction. It displays the prediction and crop preview; it does **not** create a movement sequence or compare the performer with the teacher.
2. **Offline video evaluation:** command-line workflows analyze teacher and student videos, save movement-level JSON, and compare the student’s sequence and timing with the teacher.

The main entry points are `server.py` for the live demo and `run.py` for offline workflows.

## 1. High-level architecture

```text
                         TRAINING / OFFLINE EVALUATION

Dataset images ──> Discover classes & split data ──> Train EfficientNet-B0
                                                        │
                                                        └──> outputs/model/model.pt

Teacher video + labeled segments ──> sample frames ──> crop hands ──> classify
                                                                      │
                                                                      └──> teacher_reference.json

Student video ──> sample frames ──> crop hands ──> classify ──> smooth predictions
                                                                    │
                                                                    └──> movements
                                                                          │
Teacher JSON + Student JSON ──> sequence alignment + scoring ──> report JSON


                             LIVE WEBCAM DEMO

Browser camera ──> downscale frame ──> POST /predict ──> decode image
                                                            │
                                                    crop around hands
                                                            │
                                                    EfficientNet-B0
                                                            │
Browser UI <── JSON prediction + crop preview + timing ─────┘
```

The shared core is the **hand cropper and classifier**. The live path processes one image per request. The offline path processes sampled video frames and adds temporal segmentation and comparison.

## 2. Model and image-processing basics

### Training

The model is an **EfficientNet-B0 image classifier**. Its final layer is replaced with one output per mudra class; the project expects 21 Samyukta classes when using the intended dataset. The model also exposes an intermediate feature vector called an **embedding**, later used as a rough hand-shape similarity signal.

Training discovers image files under class-named folders, creates train/validation/test splits, and fine-tunes the model in two phases: first the classification head, then the feature extractor. It saves the checkpoint and evaluation reports under `outputs/model/`. See `model.py`, `dataset.py`, and `train.py`.

Before classification, images are letterboxed to preserve their aspect ratio, resized to the model’s input size, converted to tensors, and normalized. Training adds image augmentations; inference uses a consistent evaluation transform.

### Cropping

The `HandCropper` optionally applies a configured region of interest (ROI), then uses MediaPipe to locate up to two hands. It makes a bounding box around the detected hand landmarks, adds margin, and returns that crop.

If no hand is detected, the cropper falls back to the ROI or the full frame. Each result includes crop metadata such as the method used, number of hands detected, and bounding box. This fallback keeps inference running, but a full-frame crop may produce poor predictions.

## 3. Live frontend/backend pipeline

The browser UI is served by FastAPI from `frontend/`. The HTML defines the video and prediction panel; the JavaScript controls camera capture, requests, smoothing, and display. See `index.html` and `app.js`.

### Startup

When `server.py` starts, it:

1. Loads `config.yaml` and the model checkpoint at `outputs/model/model.pt`.
2. Initializes the classifier and hand cropper.
3. Exposes `GET /health` so the page can check whether the service is ready.
4. Serves the frontend and accepts `POST /predict` requests.

The browser checks `/health` before requesting webcam access. If the model is missing or the server cannot be reached, the page displays an error status.

### One live prediction

```text
Browser                                      FastAPI backend
   │                                                │
   ├─ capture camera frame                          │
   ├─ resize to 640px wide and encode as JPEG       │
   ├─ POST /predict (multipart field "file") ──────>│
   │                                                ├─ decode JPEG to RGB
   │                                                ├─ crop hand region
   │                                                ├─ run classifier
   │                                                ├─ select top five classes
   │                                                └─ return JSON
   │<──────────── JSON: top classes, crop, timing ──┤
   ├─ update prediction, probability bars, crop     │
   ├─ draw returned bounding box                    │
   └─ wait for the next interval                    │
```

Simplified pseudocode:

```text
BACKEND_START:
    config = load_config()
    classifier = load_checkpoint("outputs/model/model.pt")
    cropper = initialize_hand_cropper(config.crop)
    ready = classifier exists AND cropper initialized

GET /health:
    return ready, model version, class names

POST /predict(image_file):
    if not ready:
        return service unavailable

    image = decode_uploaded_jpeg(image_file)
    if image cannot be decoded:
        return bad request

    lock shared inference resources
    crop, crop_info = cropper(image)
    probabilities, embedding = classifier.predict(crop)
    unlock

    return:
        top five {label, probability}
        top-one label and probability
        crop metadata and crop preview
        input dimensions and elapsed time
```

The live endpoint returns top-five probabilities, not just a label. The browser’s optional temporal smoothing averages the most recent five responses and displays the five highest results from that average. It also shows latency, estimated FPS, detected-hand count, crop method, crop preview, and bounding box. The interval slider controls how often frames are sent; while one request is still in flight, the page does not send another.

**Important distinction:** that browser-side smoothing is only for the live display. The live frontend does not generate student movement records, run sequence alignment, or calculate a comparison score.

## 4. Offline video evaluation pipeline

The offline commands are dispatched by `cli.py`, with video recognition and JSON generation in `pipeline.py`.

### A. Prepare and train the model

```text
DATASET:
    find image files under dataset class folders
    assign integer labels from sorted class names
    split data into train, validation, and test sets

TRAIN:
    initialize EfficientNet-B0 with a classification head
    train head while feature extractor is frozen
    fine-tune the feature extractor
    retain best validation checkpoint
    evaluate on test data
    save checkpoint, class names, and reports
```

A random split can put images from the same performer in both training and testing. If filenames contain performer IDs, `dataset.group_regex` can request performer-disjoint splits; otherwise, the reported accuracy may be optimistic. Check the classification report before relying on model results.

### B. Build the teacher reference

The teacher video has a separate JSON file containing labeled time segments. The pipeline maps segment labels to model class names, using aliases and name matching; configured exclusions such as Dola and Utsanga are omitted when those classes are unavailable.

```text
TEACHER(video, segments):
    load classifier and cropper
    read video at configured sample FPS
    for each sampled frame:
        crop hands
        classify crop -> probabilities and embedding
        retain timestamp and crop metadata

    for each mapped teacher segment:
        collect frames whose timestamps fall in the segment
        average probabilities and embeddings
        choose reference label:
            segment label, by default
            or classifier prediction, if configured
        save movement and frame details

    write outputs/teacher/teacher_reference.json
```

The default sample rate is 6 FPS. The teacher pipeline skips the configured initial portion of each segment when calculating movement statistics to avoid transition frames. The reference keeps both the segment’s expected label and the classifier’s prediction, so disagreements can be inspected. This is the reusable reference for subsequent student evaluations.

### C. Recognize student movements

Unlike the teacher, the student video does not need labeled segment timings. The pipeline predicts each sampled frame, smooths probabilities over time, and groups consecutive confident predictions into movements. See `segmentation.py`.

```text
STUDENT(video):
    sample frames from video
    for each frame:
        crop hands
        classify -> probabilities and embedding

    smooth frame probabilities over time
    for each frame:
        choose most likely class
        mark as uncertain if confidence is below threshold
        optionally mark uncertain if no hands are detected

    combine neighboring frames with the same class into runs
    remove runs shorter than the minimum duration
    merge same-class runs separated by short gaps

    for each remaining run:
        calculate time range, mean probabilities, and mean embedding
        save movement and per-frame details

    write student_performance.json
```

Thresholds and behavior are configurable. Defaults include a 5-frame smoothing window, minimum confidence of 0.45, minimum movement duration of 1 second, and gap tolerance of 1 second; values in `config.yaml` override defaults. See `utils.py` for the defaults.

### D. Compare student and teacher

The comparison step reads the two saved JSON files, so it does not need to run the model again. It aligns the teacher and student label sequences using Needleman–Wunsch alignment, then evaluates matches, errors, and timing. See `compare.py`.

```text
COMPARE(teacher, student):
    align ordered teacher and student movement labels

    for each aligned pair:
        teacher only       -> missing movement
        student only       -> extra movement
        both, labels differ -> wrong mudra
        both, labels match  -> correct:
            estimate overall tempo from correct movement durations
            compare start timing relative to preceding adjacent correct movement
            compare tempo-adjusted duration
            compare mean embeddings as a hand-shape proxy
            combine available components using configured weights

    average movement scores
    subtract capped penalty for extra student movements
    optionally calculate frame-level DTW similarity
    write comparison_report.json and comparison timeline
```

The score’s label, timing, duration, and configuration weights are configurable. Timing is assessed relative to a preceding adjacent correct movement; unavailable components are omitted and the remaining weights are renormalized. The “hand configuration” value is **embedding similarity**, not finger-joint measurement. It should not be interpreted as precise geometry feedback.

## 5. How to run each path

### Live webcam

Train the model first if `outputs/model/model.pt` is missing, then start the server:

```powershell
python run.py train
python server.py
```

Open `http://localhost:8000`. The README has setup details and a health-check example: `README.md`.

### Offline teacher/student comparison

Typical sequence:

```powershell
python run.py check-data
python run.py train
python run.py teacher
python run.py evaluate --video students\student1.mp4 --name student1
```

`evaluate` recognizes the student and compares its output with the saved teacher reference. The major outputs are:

```text
outputs/
  model/
    model.pt
    class_names.json
    training_report.json
    classification_report.txt
  teacher/
    teacher_reference.json
    crops/
  students/<name>/
    student_performance.json
    comparison_report.json
    comparison_timeline.png
    crops/
```

## 6. Practical limitations to keep in mind

- **Good crops matter:** verify `outputs/teacher/crops/` or the live crop preview. A detector fallback may classify a large ROI or the full frame instead of the hands.
- **Dataset-to-video domain gap:** studio photos differ from webcam and performance video frames, so video accuracy may be lower than test-set accuracy.
- **Live and offline behavior differ:** the live UI presents frame predictions; offline processing discovers movements and compares performances.
- **The comparison is approximate:** embeddings are a proxy for hand shape, not anatomical finger-level feedback.
- **Unsupported classes stay unsupported:** absent classes cannot be reliably recognized just by adding an alias; they need representative training images.

In short: **train one image classifier; reuse it for live frame predictions and offline video analysis; only the offline path converts frame predictions into movements and scores them against the teacher reference.**