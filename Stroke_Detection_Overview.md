# Tennis Stroke Detection Overview

## Executive Summary

This project is one element within a broader AI tennis coaching pipeline. It automatically identifies tennis strokes in video and returns the stroke type, the start and end frames of each stroke, the player's handedness, and the left/right court side of the stroke. Its primary goal is to convert unstructured match or training footage into structured, searchable tennis events that support downstream coaching analysis and feedback.

Potential applications include:

- automated match and practice indexing;
- player workload and stroke-volume analysis;
- rapid creation of coaching clips;
- technical and tactical performance analysis;
- searchable media archives; and
- downstream statistics, highlights, and data products.

The system performs **temporal action localization**. It does not merely decide whether a video contains a stroke. It identifies where each stroke occurs on the video timeline and classifies the stroke.

## Model Purpose and Output

For each detected event, the model produces:

```json
{
  "label": "forehand",
  "start_frame": 1240,
  "end_frame": 1382,
  "score": 0.91,
  "handedness": "right",
  "court_side": "left"
}
```

The core outputs are the stroke label, temporal boundaries, and confidence score. When trained with the corresponding annotations, the model also predicts the player's handedness and the left/right court side associated with each detected stroke.

The full system supports the following taxonomy:

- non-action;
- serve;
- forehand;
- one-handed backhand;
- two-handed backhand;
- forehand slice;
- backhand slice;
- forehand volley;
- backhand volley; and
- overhead.

Due to limitations in the dataset, the current focused experiment trains and evaluates three stroke types:

- serve;
- forehand; and
- two-handed backhand.

Other stroke annotations are treated as non-action in this experiment. This focus avoids drawing strong conclusions from rare classes that currently have insufficient training examples, which could otherwise produce less reliable predictions and more false-positive detections. The broader taxonomy remains supported for future dataset expansion.

## Dataset

The latest reported experiment combines two annotated video collections:

- 213 minutes from the 2025 US Open quarter final Alcaraz vs Lehecka, 60 fps, camera following player;
- 198 minutes from the 2025 ITF15K, 60 fps, fixed camera; and
- 411 minutes in total.

The videos are divided into one-minute clips. At 60 fps, each clip contains approximately 3,600 frames. The clips are randomized and divided into training and evaluation sets using a 3:1 ratio. The reported experiment used:

- 308 training clips; and
- 103 held-out evaluation clips.

All players in the current dataset are right-handed and use a two-handed backhand. The dataset also contains relatively few examples of slices, volleys, and overheads.

## Annotation

![Stroke annotation example](images/Screenshot%20annotation%205958.png)

Each annotation contains:

- stroke label;
- inclusive start frame;
- inclusive end frame;
- whether the start boundary is visible;
- whether the end boundary is visible;
- optional handedness; and
- optional court side.

Example:

```json
{
  "video_id": "match_001",
  "video_path": "videos/match_001.mp4",
  "num_frames": 3600,
  "fps": 60,
  "annotations": [
    {
      "label": "serve",
      "start_frame": 420,
      "end_frame": 690,
      "start_visible": true,
      "end_visible": true,
      "handedness": "right",
      "court_side": "left"
    }
  ]
}
```

Stroke boundaries are intended to include the complete action:

- serve begins with the final preparation or ball bounces before the service motion and ends after follow-through and recovery;
- groundstrokes, slices, volleys, and overheads begin at stroke initiation and end after follow-through and recovery; and
- all remaining periods are treated as non-action.

Consistent annotation policy is critical. Boundary definitions that vary by annotator directly increase localization error even when the stroke class is correct.

## Evaluation Method

Evaluation is performed on held-out videos that are not used for training. Predictions are compared with ground-truth segments using temporal Intersection over Union, or temporal IoU.

### Temporal mAP

Mean Average Precision measures both classification and temporal overlap:

- `mAP@0.3` accepts relatively coarse overlap;
- `mAP@0.5` requires moderate overlap; and
- `mAP@0.7` requires close boundary alignment.

A small decline from `mAP@0.5` to `mAP@0.7` indicates that the model generally finds not only the correct event but also reasonably accurate boundaries.

### Precision, Recall, and F1

- **Precision** asks: of all events predicted as a class, how many were correct?
- **Recall** asks: of all real events of a class, how many were found?
- **F1** balances precision and recall.

### Boundary Error

Boundary metrics report mean absolute error for:

- start frame;
- end frame; and
- event duration.

Errors are reported in canonical frames, original source frames, and seconds. Seconds are the clearest unit for comparing mixed-frame-rate videos.

### Non-Action False Positives

The report measures false stroke detections during non-action periods:

- false-positive events;
- false-positive events by stroke type;
- false-positive frames;
- false positives per 1,000 non-action frames; and
- false positives per minute.

This is operationally important because a system that detects real strokes but also creates many false events during dead time produces a poor user experience.

### Macro and Dataset Results

Two summary views are produced:

- **Macro summary** calculates metrics per video and gives each video equal weight.
- **Dataset summary** pools predictions and ground truth across the complete evaluation set, so videos with more events contribute more strongly.

Dataset metrics are useful for overall production accuracy. Macro metrics help identify whether performance is consistent across videos rather than driven by a subset of easier or event-rich videos.

## Latest Reported Three-Class Performance

The following results are from the held-out set of 103 videos.

### Overall Detection

| Metric | Macro | Dataset |
|---|---:|---:|
| mAP@0.3 | 0.874 | 0.893 |
| mAP@0.5 | 0.868 | 0.889 |
| mAP@0.7 | 0.819 | 0.842 |

The dataset-level `mAP@0.5` of 0.889 shows strong detection performance on the focused three-class task. The dataset-level `mAP@0.7` of 0.842 indicates that most correct detections retain good temporal overlap under a stricter boundary requirement.

### Per-Class Results at Temporal IoU 0.5

| Stroke | Precision | Recall | F1 | TP | FP | FN |
|---|---:|---:|---:|---:|---:|---:|
| Serve | 0.956 | 0.956 | 0.956 | 109 | 5 | 5 |
| Forehand | 0.690 | 0.955 | 0.801 | 149 | 67 | 7 |
| Two-handed backhand | 0.784 | 0.853 | 0.817 | 87 | 24 | 15 |

Serve detection is the strongest and most balanced class. Forehand recall is high, meaning very few forehands are missed, but lower precision indicates over-detection. Two-handed backhand performance is more balanced, with some remaining misses and false detections.

### Boundary Accuracy

Dataset-level mean errors:

| Boundary Metric | Canonical Frames | Source-Video Frames | Seconds |
|---|---:|---:|---:|
| Start error | 7.52 | 15.05 | 0.251 |
| End error | 4.21 | 8.42 | 0.140 |
| Duration error | 8.36 | 16.71 | 0.279 |

Start boundaries are currently less accurate than end boundaries. This is consistent with ambiguity around when preparation becomes the beginning of a stroke and suggests that annotation consistency and start-boundary refinement are important improvement areas.

### False Positives During Non-Action

Across the complete evaluation set:

- 65 unmatched false-positive stroke events overlapped non-action;
- 42 were forehands;
- 19 were two-handed backhands;
- 4 were serves;
- 3,407 non-action frames were covered by these unmatched predictions; and
- the rate was approximately 3.40 false positives per minute of non-action.

Forehand is therefore the main source of false detections. Threshold calibration, additional hard-negative examples, and more representative non-action footage are direct opportunities for improvement.

## Current Strengths

- Strong overall detection performance for the focused three-class task.
- High serve precision and recall.
- High forehand recall.
- Frame-rate-independent modeling for mixed 30 fps and 60 fps inputs.
- Original-video frame coordinates in user-facing output.
- Reusable feature cache for efficient retraining and dataset growth.
- Multiple independent decoding strategies for comparison and refinement.
- Explicit measurement of non-action false positives and temporal boundaries.

## Current Limitations

- Results currently support a three-class deployment claim, not yet a reliable nine-stroke deployment claim.
- Rare strokes such as volleys, overheads, slices, and one-handed backhands need substantially more labeled examples.
- Forehand false positives remain the main precision issue.
- Performance may change with camera angle, zoom, resolution, player level, occlusion, doubles play, broadcast editing, or environmental differences.
- The reported 103-video evaluation set is meaningful for development but is not yet a substitute for a larger, independently sourced acceptance test.
- Random train/evaluation splits measure generalization to held-out videos from the combined collections. A stronger commercial validation should also hold out entire venues, camera systems, tournaments, or data providers.

## Recommended Next Phase

1. Collect and annotate substantially more video samples across the full stroke taxonomy, including one-handed backhands, forehand and backhand slices, forehand and backhand volleys, and overheads. The expanded dataset should balance stroke types, players, camera views, venues, and playing conditions so the model can progress from the current three-class focus to reliable detection of all supported stroke types.
2. Add hard-negative non-action footage, especially examples currently mistaken for forehands.
3. Establish a locked external test set that is never used for model or threshold tuning.
4. Define product acceptance targets, such as maximum false events per minute and minimum recall for each supported stroke.
