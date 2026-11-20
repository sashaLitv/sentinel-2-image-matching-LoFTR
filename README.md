# Sentinel-2 image matching using LoFTR
This repository represents a Proof-of-Concept (PoC) demonstrating a custom LoFTR-based CV pipeline for Sentinel-2 image matching.


## Task Overview
The algorithm finds matching keypoints between two images and outputs all detected points, the total point count, the number of USAC_MAGSAC-verified inliers, an overall confidence score.
<div align="center">
  <img width="979" height="504" alt="Example of Sentinel-2 image matching on same location with transformation and season changes" src="https://github.com/user-attachments/assets/5e2d3146-fd06-47bb-9f8b-6e46a6864bb4" />
  <p><em>Example 1. same location - seasonal change with 15° rotation (only 200 inliers demonstration). More examples are available in <a href="notebooks/demo.ipynb">notebooks/demo.ipynb</a>.</em></p>
</div>

## Algorithm Explanation 
```
[Input: img0, img1]
       │
       ▼
[1. Coarse аlignment: multi-anchor global homography]
  ├── Select k high-detail, spread anchor tiles on img0 (Mertens + Laplacian variance)
  ├── Spiral search candidate tiles for k selected on img1 using LoFTR
  ├── Pool verified correspondences across anchors
  └── Fit global homography H (USAC_MAGSAC) ──► [Fallback to H = I if unaligned]
       │
       ▼
[2. Aligned еiling & geometric warping]
  ├── Slice img0 into overlapping tiles (512×512, 64px overlap)
  ├── Project tile footprint onto img1 via global H
  ├── Crop from img1 and warp back using H_local⁻¹ into img0 perspective (derived from global H + offsets)
  └── Build binary validity masks to mask padding and warp borders (mask_pad & mask_warped)
       │
       ▼
[3. Fine feature matching (LoFTR)]
  ├── Run LoFTR on aligned tile pairs
  ├── Discard invalid/edge keypoints using validity masks 
  └── Lift surviving points to global space: 
        shift pts0 by (x, y) to img0 global space; shift pts1 by (x, y) and project via H to unwarped img1 space
       │
       ▼
[4. Outlier filtering & decision scoring]
  ├── Aggregate keypoints across all tiles
  ├── Filter global outliers with USAC_MAGSAC
  └── Compute overlap confidence:
        P(overlap) = 1 / (1 + 10^(-slope * (log10(inliers + 1) - midpoint)))
        Decision: P >= 0.5 → Same Location | P < 0.5 → Different Locations
```
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

## 📁 Project Structure

The project is structured to separate the algorithm_creation, inference pipeline, evaluation. 

```text
└── sentinel-2-image-matching-LoFTR/      
    ├── data/                               
    │   ├── demo-1                          # Same-day shifted pair (July 30, 2026) — vertical offset detection & alignment check
    │   ├── demo-2                          # Seasonal pair (January vs March) — robustness to season/lighting
    │   ├── demo-3                          # Non-overlapping pair 
    │   ├── demo-4                          # Rotational/scale offset pair 
    │   └── demo-5                          # Rotational/scale offset + challenging conditions pairs
    ├── notebooks/
    │   ├── demo.ipynb                      # Scratch notebook for single-pair experiments and visual checks
    │   └── evaluation.ipynb                # Batch evaluation over a dataset: pair building, confusion matrix, metrics
    ├── src/
    │   ├── model_inference.py              # Runnable pipeline: CLI, consolidated reporting, match visualization
    │   └── algorithm_creation.py           # Core library: multi-anchor aligner, tiling, tile matcher...
    ├── README.md                           # Project documentation
    ├── requirements.txt                    # Project dependencies
    └── .gitignore                       
```


## How to Run
The project ships two entry points. Both take the same two image paths and must be run from inside `src/`, using the project virtual env.

### 1. Base algorithm script — `algorithm_creation.py`

A minimal smoke test of the core classes (GlobalAligner, ImagePreprocessor, SatelliteMatcher). It aligns the pair, matches all tiles and prints the counts. Use it to confirm the library works on your machine — it produces **no images**.

```text
usage: algorithm_creation.py [-h] --img0 IMG0 --img1 IMG1 [--tile_size TILE_SIZE]
                             [--overlap OVERLAP] [--device DEVICE]

Test Satellite Image Matching classes

options:
  -h, --help             show this help message and exit
  --img0 IMG0            Path to the first satellite image
  --img1 IMG1            Path to the second satellite image
  --tile_size TILE_SIZE  Tile size for processing (default: 512)
  --overlap OVERLAP      Overlap between tiles (default: 64)
  --device DEVICE        Type of device (default: 'cpu')
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
                          [--confidence_threshold CONFIDENCE_THRESHOLD] [--tile_size TILE_SIZE]
                          [--magsac_threshold MAGSAC_THRESHOLD] [--show_outliers]
                          [--max_points MAX_POINTS] [--device DEVICE]

Run Satellite Image Matching Inference

options:
  -h, --help                                    show this help message and exit
  --img0 IMG0                                   Path to the first satellite image
  --img1 IMG1                                   Path to the second satellite image
  --confidence_threshold CONFIDENCE_THRESHOLD   LoFTR confidence threshold (default: 0.8)
  --tile_size TILE_SIZE                         Tile size for processing (default: 512)
  --magsac_threshold MAGSAC_THRESHOLD           MAGSAC threshold (default: 3.0)
  --show_outliers                               Flag to visualize outlier matches (red lines)
  --max_points MAX_POINTS                       Maximum number of points to visualize (default: 150)
  --device DEVICE                               Type of device (default: 'cpu')
```

```text
python3 src/model_inference.py --img0 "data/demo-1/2026-07-30-00_00_2026-07-30-23_59_Sentinel-2_L2A_True_color-2.jpg" --img1 "data/demo-1/2026-07-30-00_00_2026-07-30-23_59_Sentinel-2_L2A_True_color.jpg"
```

### Input constraints

- Both images must be at least `tile_size × tile_size` pixels (512 × 512 by default).
- Images do **not** need to be square, or the same size as each other.

### What you will see

#### `algorithm_creation.py` prints, in order:

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

#### `model_inference.py` prints a header, then — only when co-registration cannot be estimated — one consolidated notice block:

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

The alignment line has two states: 
 - `OK (aligned)` when a homography was found,
 - `OK (co-registered via grid / identity fallback)` when no correction was needed because the images were already aligned,
 - `NOTE (distinct locations or insufficient overlap)`.

**Read overlap confidence for the decision** >= 0.5 means the two images likely show the same place.

Finally a 16 × 8 matplotlib figure opens: the two images side by side, joined by match lines — green for accepted inliers, red for rejected matches when `--show_outliers` is passed. If no match survives, no figure is drawn.


## Algorithm Evaluation

