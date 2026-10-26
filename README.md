# Sentinel-2 image matching using LoFTR

## ⚙️ Setup Instructions

To run this project locally, it is recommended to use a virtual environment.
1. Clone the repository:
```text
git clone https://github.com/sashaLitv/sentinel-2-image-matching-LoFTR
cd "sentinel-2-image-matching-LoFTR"
```
2. Create and activate a virtual environment:
```text
python3 -m venv venv_cv
source venv_cv/bin/activate  
```
3. Install dependencies:
```text
pip install -r requirements.txt
```

## How to Run
The project ships two entry points. Both take the same two image paths and must be run from inside `src/`, using the project virtual env.

### 1. Base algorithm script — `algorithm_creation.py`

A minimal smoke test of the core classes (`GlobalAligner`, `ImagePreprocessor`, `SatelliteMatcher`). It aligns the pair, matches all tiles and prints the counts. Use it to confirm the library works on your machine — it produces **no images**.

```text
usage: algorithm_creation.py [-h] --img0 IMG0 --img1 IMG1 [--tile_size TILE_SIZE]
                             [--overlap OVERLAP] [--device DEVICE]

Test Satellite Image Matching classes

options:
  -h, --help            show this help message and exit
  --img0 IMG0           Path to the first satellite image
  --img1 IMG1           Path to the second satellite image
  --tile_size TILE_SIZE
                        Tile size for processing (default: 512)
  --overlap OVERLAP     Overlap between tiles (default: 64)
  --device DEVICE       Type of device (default: 'cpu')
```

```text
python3 src/algorithm_creation.py --img0 "data/demo-1/2026-07-30-00_00_2026-07-30-23_59_Sentinel-2_L2A_True_color-2.jpg" --img1 "data/demo-1/2026-07-30-00_00_2026-07-30-23_59_Sentinel-2_L2A_True_color.jpg"
```

### 2. Inference CLI pipeline — `model_inference.py`

The full pipeline. It aligns the pair, matches every tile, aggregates the
correspondences, filters outliers with MAGSAC, reports an **overlap confidence**, and
finally draws the surviving matches. This is the script to use for real runs.

```text
usage: model_inference.py [-h] --img0 IMG0 --img1 IMG1
                          [--confidence_threshold CONFIDENCE_THRESHOLD]
                          [--tile_size TILE_SIZE]
                          [--ransac_threshold RANSAC_THRESHOLD]
                          [--show_outliers] [--max_points MAX_POINTS]
                          [--device DEVICE]

Run Satellite Image Matching Inference

options:
  -h, --help            show this help message and exit
  --img0 IMG0           Path to the first satellite image
  --img1 IMG1           Path to the second satellite image
  --confidence_threshold CONFIDENCE_THRESHOLD
                        LoFTR confidence threshold (default: 0.8)
  --tile_size TILE_SIZE
                        Tile size for processing (default: 512)
  --ransac_threshold RANSAC_THRESHOLD
                        RANSAC threshold in pixels (default: 3.0)
  --show_outliers       Flag to visualize outlier matches (red lines)
  --max_points MAX_POINTS
                        Maximum number of points to visualize (default: 150)
  --device DEVICE       Type of device (default: 'cpu')
```

```text
python3 src/model_inference.py --img0 "data/demo-1/2026-07-30-00_00_2026-07-30-23_59_Sentinel-2_L2A_True_color-2.jpg" --img1 "data/demo-1/2026-07-30-00_00_2026-07-30-23_59_Sentinel-2_L2A_True_color.jpg"
```

### Input constraints

- Both images must be at least `tile_size × tile_size` pixels (512 × 512 by default).
- Image dimensions must be divisible by 8 for LoFTR.
- Images do **not** need to be square, or the same size as each other.

### What you will see

**`algorithm_creation.py`** prints, in order:

```text
Testing base classes for:
  img0: <path>
  img1: <path>
```
then either the estimated matrix, or the reason alignment was skipped:
```text
Global Homography Matrix:
 [[ ... ]]
```
```text
Global homography estimation failed: too_few_inliers
```
and finishes with a single summary line:
```text
Test passed! Found 4941 total matches across tiles
```
It draws **no figure**.

**`model_inference.py`** prints a header, then — only when co-registration cannot be
estimated — one consolidated notice block:

```text
============================================================
 ALIGNMENT NOTICE — T36UYA_..._TCI.jp2 vs T36UYA_..._TCI.jp2
  Standard geometric correction was skipped; evaluating direct co-registration.
------------------------------------------------------------
  anchors selected ................ 8
  anchors matched ................. 1
  candidate tiles scored .......... 841
  candidates passing sanity ....... 1
  best inlier count ............... 4   (required: 15)
  pooled points ................... 4
  pooled inliers .................. 4
  reason .......................... no correction needed (already co-registered) or locations differ
============================================================
```

and always one final conclusion block:

```text
============================================================
 RESULT
  overlap confidence .............. 0.997  (HIGH — likely the same location)
  inliers ......................... 4853
  total matches ................... 4941
  alignment ....................... OK (aligned)
============================================================
```

The `alignment` line has two states: `OK (aligned)` when a homography was found,
`OK (co-registered via grid / identity fallback)` when no correction was needed
because the images were already aligned, and `NOTE (distinct locations or
insufficient overlap)` otherwise. **Read `overlap confidence` for the decision** —
`>= 0.5` means the two images likely show the same place.

Finally a `16 × 8` matplotlib figure opens: the two images side by side, joined by
match lines — green for accepted inliers, red for rejected matches when
`--show_outliers` is passed. If no match survives, no figure is drawn.


## 📁 Project Structure

The project is structured to separate the algorithm_creation, inference pipeline, evaluation. 

```text
└── sentinel-2-image-matching-LoFTR/      
    ├── data/                               
    │   ├── demo-1                          # Same-day pair — baseline sanity check
    │   ├── demo-2                          # Seasonal pair (January vs March) — robustness to season/lighting
    │   ├── demo-3                          # Non-overlapping pair — regression guard for the failure path
    ├── notebooks/
    │   ├── demo.ipynb                      # Scratch notebook for single-pair experiments and visual checks
    │   └── evaluation.ipynb                # Batch evaluation over a dataset: pair building, confusion matrix, metrics
    ├── src/
    │   ├── model_inference.py              # Runnable pipeline: CLI, consolidated reporting, match visualization
    │   └── algorithm_creation.py           # Core library: multi-anchor aligner, tiling, tile matcher...
    ├── README.md                           # Project documentation
    └── requirements.txt                    # Project dependencies
    └── .gitignore                          
```

## Algorithm Explanation 

## Model Evaluation
