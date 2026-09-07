# Image Processing Project — Student Instructions

**Project folder:** `image_processing_examples/`  
**Last updated:** September 2026

---

## Table of Contents

1. [What This Project Does](#1-what-this-project-does)
2. [Project Folder Structure](#2-project-folder-structure)
3. [How to Run the Code — Three Options](#3-how-to-run-the-code--three-options)
4. [Library 1: pic2points](#4-library-1-pic2points)
5. [Library 2: fitmethis](#5-library-2-fitmethis)
6. [The Standard Image Processing Pipeline](#6-the-standard-image-processing-pipeline)
7. [Common Mistakes and Fixes](#7-common-mistakes-and-fixes)
8. [Quick Reference Card](#8-quick-reference-card)

---

## 1. What This Project Does

This project contains four sets of MATLAB scripts for **experimental image analysis**. Each set
processes photographs or image sequences from a physics experiment and extracts quantitative data.

| Sub-project | What it analyses |
|---|---|
| `Drop_edge_detetction/` | Shape of a sessile droplet → polar edge coordinates |
| `Soap_Film_Edge_detection/` | Retraction of a soap film over time → edge velocity |
| `kinematics measurement/` | Deflection of a membrane → displacement, velocity, acceleration |
| `spot_detection/` | Micro-droplet stains on paper → size distribution |

All four sub-projects share the same underlying approach:

> **Load image → clean up → detect edges → extract coordinates → analyse data**

Two reusable library functions make this possible: **`pic2points`** and **`fitmethis`**.

---

## 2. Project Folder Structure

```
image_processing_examples/
│
├── Drop_edge_detetction/
│   └── Final code/
│       ├── code.m                ← main script to run
│       ├── pic2points.m          ← library (image → coordinates)
│       ├── export_fig.m          ← library (publication-quality figures)
│       ├── 1200.jpg              ← example input image
│       └── information.xlsx      ← results output
│
├── Soap_Film_Edge_detection/
│   ├── edge_velocity.m           ← main script (loops over image sequence)
│   ├── edge_position.m           ← post-processing script
│   ├── pic2points.m              ← library copy
│   └── 0.jpg ... 7900.jpg        ← input image sequence
│
├── kinematics measurement/
│   ├── Velocity10KVParallelCap05mm.m  ← main script
│   ├── pic2points.m              ← library copy
│   └── 1.jpg ... 200.jpg         ← input image sequence
│
└── spot_detection/
    ├── water.m                   ← main script
    ├── fitmethis.m               ← library (statistical distribution fitting)
    ├── plotfitdist.m             ← helper for fitmethis
    └── water.tif                 ← input image
```

---

## 3. How to Run the Code — Three Options

### Option A — MATLAB (Full Licence)

If you have access to a MATLAB licence this is the most straightforward option.

**Required Toolboxes** — check via `ver` in the MATLAB Command Window:

| Toolbox | Required for |
|---|---|
| Image Processing Toolbox | All scripts |
| Statistics and Machine Learning Toolbox | `fitmethis` only |

**Setting up the path** (do this once per session):

```matlab
addpath(genpath('C:\path\to\image_processing_examples'));
```

Or use the GUI: *Home tab → Set Path → Add with Subfolders → select project folder → Save*

> **Note:** MATLAB may be free through your institution.
> Check **https://matlab.mathworks.com** — it runs in a browser, no installation needed.

---

### Option B — GNU Octave (Free, Recommended if No MATLAB Licence) ⭐

GNU Octave is a free, open-source program that runs MATLAB `.m` files with minimal changes.
Download from: **https://octave.org/download**

**One-time package installation** (type inside Octave, requires internet):

```octave
pkg install -forge image       % image processing functions
pkg install -forge statistics  % statistical functions (needed for fitmethis)
pkg install -forge io          % xlswrite / xlsread support
```

**Load packages at the start of every session:**

```octave
pkg load image
pkg load statistics
pkg load io
```

**One line to fix in `pic2points.m`** — open the file and make this change:

```matlab
% FIND this line (around line 107):
ImBW = im2bw(Im, TshV);

% REPLACE it with:
ImBW = imbinarize(im2gray(Im), TshV);
```

**One line to fix in scripts using `polarplot`:**

```matlab
% REPLACE:
polarplot(theta, r, '.')

% WITH:
polar(theta, r, '.')
```

**Compatibility summary:**

| Function used in scripts | Works in Octave? |
|---|---|
| `imread`, `imwrite`, `imshow` | Yes, fully |
| `imresize`, `imcrop`, `imadjust` | Yes, fully |
| `edge(..., 'Canny')` | Yes, fully |
| `bwareaopen`, `imcomplement`, `im2uint8` | Yes, fully |
| `imfindcircles` | Yes (minor sensitivity differences) |
| `bwmorph` | Yes, fully |
| `mle` (used inside fitmethis) | Yes, via statistics package |
| `xlswrite` | Yes, via io package |
| `polarplot` | Use `polar()` instead |
| `im2bw` | Use `imbinarize()` instead (see fix above) |

---

### Option C — Python (Free, Requires Rewriting the Scripts)

Python can replicate everything using free libraries. This requires more effort upfront but is
completely independent of MATLAB or Octave.

**Install the required libraries** (one-time, in your system terminal):

```bash
pip install opencv-python scikit-image scipy numpy matplotlib openpyxl
```

**Python equivalent of `pic2points`:**

```python
import cv2
import numpy as np

def pic2points(image_path, threshold=None, thin=False, max_points=None):
    """
    Converts an image into a matrix of (x, y) coordinates of dark pixels.

    Parameters
    ----------
    image_path : str     Path to image (black objects on white background)
    threshold  : float   Binarisation threshold 0-1. Default: auto (Otsu)
    thin       : bool    Thin detected lines to 1-pixel width
    max_points : int     Maximum number of points to return (random sample)

    Returns
    -------
    points : ndarray     N x 2 array of [x, y] coordinates
    """
    img = cv2.imread(image_path, cv2.IMREAD_GRAYSCALE)

    if threshold is None:
        _, bw = cv2.threshold(img, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    else:
        _, bw = cv2.threshold(img, int(threshold * 255), 255, cv2.THRESH_BINARY)

    bw = cv2.bitwise_not(bw)  # invert so dark objects become white

    if thin:
        from skimage.morphology import skeletonize
        bw = (skeletonize(bw > 0).astype(np.uint8)) * 255

    y_coords, x_coords = np.nonzero(bw)
    y_coords = img.shape[0] - y_coords   # flip y-axis to match MATLAB convention

    points = np.column_stack((x_coords, y_coords))

    if max_points is not None and len(points) > max_points:
        idx = np.random.choice(len(points), max_points, replace=False)
        points = points[idx]

    return points


# -- Usage --
import matplotlib.pyplot as plt

pts = pic2points('profile.jpg')
plt.scatter(pts[:, 0], pts[:, 1], s=1)
plt.gca().set_aspect('equal')
plt.show()
```

**Python equivalent of `fitmethis`:**

```python
import scipy.stats as st
import numpy as np

def fitmethis(data, criterion='AIC'):
    """
    Fits multiple statistical distributions to data and ranks them.

    Parameters
    ----------
    data      : array-like   1-D array of values
    criterion : str          'AIC' (default) or 'LL' for Log-Likelihood

    Returns
    -------
    results : list of dicts  Sorted best-first. Each dict has keys:
                             'name', 'params', 'LL', 'AIC'
    """
    distributions = [
        st.norm, st.expon, st.gamma, st.lognorm, st.weibull_min,
        st.beta, st.logistic, st.rayleigh, st.gumbel_r
    ]
    results = []
    data = np.asarray(data).flatten()

    for dist in distributions:
        try:
            params = dist.fit(data)
            ll  = np.sum(dist.logpdf(data, *params))
            aic = 2 * len(params) - 2 * ll
            results.append({'name': dist.name, 'params': params,
                            'LL': ll, 'AIC': aic})
        except Exception:
            pass

    reverse = (criterion.upper() == 'LL')
    results.sort(key=lambda x: x[criterion.upper()], reverse=reverse)

    print(f"\n{'Name':>20}  {'LL':>12}  {'AIC':>12}")
    print("-" * 48)
    for r in results:
        print(f"{r['name']:>20}  {r['LL']:>12.3f}  {r['AIC']:>12.3f}")

    return results


# -- Usage --
results = fitmethis(my_data)
print("Best fit:", results[0]['name'])
print("Parameters:", results[0]['params'])
```

---

## 4. Library 1: `pic2points`

### What it does

`pic2points` reads an image and returns the pixel coordinates (x, y) of all the dark objects it
finds. The output is a matrix where each row is one point, column 1 is x, and column 2 is y.

### The one rule you must follow

> **The image MUST have a WHITE background with BLACK objects.**
> If your image is the other way around, the coordinates will be wrong or empty.
> See Section 6 for how to prepare your image correctly.

### Function signature

```matlab
CoordinateMatrix = pic2points(Im, TshV, ImPlot, maxNum)
```

| Argument | Required? | Description | Example |
|---|---|---|---|
| `Im` | YES | Image loaded with `imread` | `Im` |
| `TshV` | Optional | Binarisation threshold, 0 to 1. Leave empty for auto. | `0.5` |
| `ImPlot` | Optional | Use `'plot'` to thin lines to 1-pixel width | `'plot'` |
| `maxNum` | Optional | Maximum number of points to return | `1000` |

### Usage examples

```matlab
% Simplest form — MATLAB auto-detects the threshold
Im = imread('profile.jpg');
pts = pic2points(Im);
scatter(pts(:,1), pts(:,2), '.');


% Manual threshold — try values between 0.3 and 0.8
Im = imread('profile.jpg');
pts = pic2points(Im, 0.55);
scatter(pts(:,1), pts(:,2), '.');


% For line drawings — thin lines to 1 pixel
Im = imread('profile.jpg');
pts = pic2points(Im, 0.55, 'plot');
scatter(pts(:,1), pts(:,2), '.');


% Limit output to 500 random points (faster for large images)
Im = imread('profile.jpg');
pts = pic2points(Im, 0.55, 'plot', 500);
scatter(pts(:,1), pts(:,2), '.');
```

> **Tip:** Always plot the output immediately after calling `pic2points` to visually verify
> that the coordinates are correct before doing any further calculations.

---

## 5. Library 2: `fitmethis`

### What it does

`fitmethis` takes a vector of numbers (e.g. droplet radii, edge positions) and automatically tries
fitting every available statistical distribution to your data. It prints a ranked table (best fit
first) and produces a plot. This saves you from manually testing each distribution.

### Function signature

```matlab
F = fitmethis(data, 'OptionName', OptionValue, ...)
```

### All options

| Option | Allowed values | Default | Description |
|---|---|---|---|
| `'dtype'` | `'cont'` or `'disc'` | auto | Force continuous or discrete distributions |
| `'ntrials'` | a number | — | Number of trials (for binomial only) |
| `'figure'` | `'on'` or `'off'` | `'on'` | Show plot of the best-fitting distribution |
| `'alpha'` | 0 to 1 | `0.05` | Confidence level for parameter confidence intervals |
| `'criterion'` | `'LL'` or `'AIC'` | `'LL'` | How to rank the distributions |
| `'output'` | `'on'` or `'off'` | `'on'` | Print the results table to the Command Window |
| `'pref'` | distribution name string | — | Force a specific distribution to be plotted |

### Understanding the output structure `F`

```matlab
F(1).name    % best-fitting distribution name, e.g. 'lognormal'
F(1).par     % fitted parameters, e.g. [mu, sigma]
F(1).ci      % confidence intervals for each parameter
F(1).LL      % Log-Likelihood  (higher = better fit)
F(1).aic     % Akaike Information Criterion (lower = better fit)
```

### Usage examples

```matlab
% Basic: fit all distributions, print table, show plot
F = fitmethis(myData);
disp(['Best fit: ' F(1).name]);


% Silent mode — no plot, no printed table
F = fitmethis(myData, 'figure', 'off', 'output', 'off');


% Rank by AIC instead of Log-Likelihood
F = fitmethis(myData, 'criterion', 'AIC');


% Force a preferred distribution to be plotted
F = fitmethis(myData, 'pref', 'lognormal');


% Discrete count data
F = fitmethis(countData, 'dtype', 'disc');
```

> **Warning:** If your data contains negative values, `fitmethis` will only fit the Normal
> distribution. Make sure your data vector is correct before calling the function.

---

## 6. The Standard Image Processing Pipeline

Every script in this project follows the same sequence of steps.
Work through them in order and use `imshow` to visually check the result after each step.

### Step 1 — Load and resize the image

```matlab
II = imread('your_image.jpg');
I  = imresize(II, [1024 1024]);      % adjust size as needed
figure(1); imshow(I);
```

### Step 2 — Crop to the region of interest (optional)

```matlab
% imcrop(image, [x_start  y_start  width  height])
I_cropped = imcrop(I, [100 50 800 700]);
figure(2); imshow(I_cropped);
```

### Step 3 — Increase contrast with intensity thresholding

```matlab
% Pixels below threshold → black (0); pixels above → white (255)
% Adjust the number (e.g. 90, 100, 134) until the edge stands out clearly.
threshold_intensity = 90;
OutputImage = I_cropped;
for i = 1:size(OutputImage, 1)
    for j = 1:size(OutputImage, 2)
        if OutputImage(i,j) < threshold_intensity
            OutputImage(i,j) = 0;
        else
            OutputImage(i,j) = 255;
        end
    end
end
figure(3); imshow(OutputImage);
```

### Step 4 — Detect the edge using Canny

```matlab
[~, threshOut] = edge(OutputImage, 'Canny');   % auto-calculate threshold
BW = edge(OutputImage, 'Canny', threshOut);
figure(4); imshow(BW);
```

### Step 5 — Remove small noise objects

```matlab
% Anything smaller than N pixels is treated as noise and removed.
% Start with 30; increase if too much noise remains.
N = 30;
BW_clean = bwareaopen(BW, N);
figure(5); imshow(BW_clean);
```

### Step 6 — Invert the image (REQUIRED before pic2points)

```matlab
% pic2points needs BLACK objects on WHITE background.
% After Canny the background is black, so invert it here.
BW_inverted = imcomplement(BW_clean);
figure(6); imshow(BW_inverted);
```

### Step 7 — Save, reload, then call pic2points

```matlab
imwrite(im2uint8(BW_inverted), 'profile.jpg', 'jpg');
Im = imread('profile.jpg');
CoordinateMatrix = pic2points(Im);
scatter(CoordinateMatrix(:,1), CoordinateMatrix(:,2), '.');
```

### Step 8 — Analyse the coordinates

```matlab
% Convert pixels to physical units (calibration factor from your experiment)
calibration = 20;   % pixels per mm
coords_mm   = CoordinateMatrix ./ calibration;

% Find the leading edge (leftmost point)
leading_edge = min(CoordinateMatrix(:,1));

% Convert to polar coordinates
x_c = coords_mm(:,1) - x_centre;
y_c = coords_mm(:,2) - y_centre;
r     = sqrt(x_c.^2 + y_c.^2);
theta = atan2(y_c, x_c);
```

---

## 7. Common Mistakes and Fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `pic2points` returns an empty matrix | Background is black, not white | Make sure you ran Step 6 (`imcomplement`) before calling `pic2points` |
| Too many noise points returned | Threshold too low or noise not removed | Increase the `bwareaopen` number in Step 5, or raise the intensity threshold in Step 3 |
| Coordinates look spatially correct but y-axis is flipped | Normal image convention difference | `pic2points` corrects for this automatically; use `axis equal` when plotting to verify |
| `Undefined function 'plotfitdist'` | File not on the path | Add the folder containing `plotfitdist.m` to the path, or use `'figure','off'` |
| `fitmethis` toolbox error | Statistics toolbox missing | Octave: run `pkg install -forge statistics` then `pkg load statistics` |
| Octave error on `im2bw` | Deprecated function | Replace `im2bw(Im, t)` with `imbinarize(im2gray(Im), t)` |
| Octave error on `polarplot` | Different function name | Replace `polarplot(...)` with `polar(...)` |
| Script cannot find the image file | Wrong working directory | Use `cd` to go into the correct sub-project folder first, e.g. `cd('C:\...\Final code')` |
| `xlswrite` fails in Octave | io package not loaded | Run `pkg load io` before the script |

---

## 8. Quick Reference Card

```matlab
%% SETUP — MATLAB
addpath(genpath('/path/to/image_processing_examples'));

%% SETUP — Octave
pkg load image
pkg load statistics
pkg load io

%% FULL PIPELINE: image → coordinates
II  = imread('image.jpg');
I   = imresize(II, [1024 1024]);
% apply intensity thresholding loop here (Step 3 above)
[~, thr] = edge(I, 'Canny');
BW  = edge(I, 'Canny', thr);
BW  = bwareaopen(BW, 30);
BW  = imcomplement(BW);
imwrite(im2uint8(BW), 'profile.jpg', 'jpg');
Im  = imread('profile.jpg');
pts = pic2points(Im);              % N x 2 matrix of [x, y]
scatter(pts(:,1), pts(:,2), '.');  % always verify visually

%% pic2points — argument summary
pts = pic2points(Im);                        % auto threshold
pts = pic2points(Im, 0.55);                  % manual threshold (0–1)
pts = pic2points(Im, 0.55, 'plot');          % thin lines to 1 px
pts = pic2points(Im, 0.55, 'plot', 1000);    % max 1000 random points

%% fitmethis — argument summary
F = fitmethis(myData);                            % fit + print + plot
F = fitmethis(myData, 'figure', 'off');           % no plot
F = fitmethis(myData, 'output', 'off');           % no printed table
F = fitmethis(myData, 'criterion', 'AIC');        % rank by AIC
F = fitmethis(myData, 'pref', 'lognormal');       % preferred distribution
F = fitmethis(countData, 'dtype', 'disc');        % discrete data

disp(F(1).name)    % best distribution name
disp(F(1).par)     % fitted parameters
disp(F(1).LL)      % Log-Likelihood (higher = better)
disp(F(1).aic)     % AIC (lower = better)
```

---

## Reference Scripts to Study

These are fully working examples — read them alongside this guide:

| Script | What it demonstrates |
|---|---|
| `Drop_edge_detetction/Final code/code.m` | Single-image pipeline with circular masking and polar coordinate conversion |
| `Soap_Film_Edge_detection/edge_velocity.m` | Looping over hundreds of images, tracking edge position over time |
| `kinematics measurement/Velocity10KVParallelCap05mm.m` | Full kinematics: displacement → velocity → acceleration from image sequence |
| `spot_detection/water.m` | Circle detection (Hough transform) + size distribution fitting with `fitmethis` |

---

*Work through the pipeline step by step and use `imshow` after every step to check what your
image looks like. Every experimental image is different, and the threshold values will need
adjustment. Good luck!*
