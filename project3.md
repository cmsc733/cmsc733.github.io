---
layout: page
mathjax: true
title: Rotobrush
permalink: /2018/proj/p3/
---

Table of Contents:
- [Deadline](#due)
- [Introduction](#intro)
- [Implementation Overview](#system_overview)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Deadline 
10:59:59AM, Thursday, November 15, 2018

<a name='intro'></a>
## Introduction

The aim of this project is to segment deformable object from a given video sequence. 
This document just provides an overview of what you need to do.  For a full breakdown of how each step in the pipeline works, see <a href="https://cmsc426.github.io/rotobrush/">the course notes for this project</a>.

<a name='system_overview'></a>
## Implementation Overview

Download the starter code and data <a href="https://www.dropbox.com/s/32fa11l5pnakcph/CMSC426_P3.zip"> here </a>


A brief description of each step (you'll implement the steps **in bold**):
- **`myRotobrush.m`:  Wrapper function.**
- `initLocalWindows.m`:  Creates local windows on boundary of mask. 
- **`initColorModels.m`:   Initializes color models.** 
- **`initShapeConfidences.m`: Initializes shape confidences.** 
- **`localFlowWarp.m`:  Calculates local window movement based on optical flow between frames.**
- **`calculateGlobalAffine.m`:  Finds affine transform between two frames.** 
- **`updateModels.m`:  Update shape and color models.** 
- `showLocalWindows.m`:  Plots local windows. 
- `showColorConfidences.m`: Plots the color confidence for each local window. 
- `equidistantPointsOnPerimeter.m`: Find equally spaced points along the perimeter of a polygon


### Point Distribution:
- Setup Local Windows: 5 pts
- Initialize Color Models: 10pts
- Compute Color Model Confidence: 5 pts
- Initialize Shape Model: 10 pts
- Compute Shape confidence: 5 pts
- Estimate Entire-Object Motion: 5 pts
- Estimate Local Boundary Deformation: 10 pts
- Update Color Model (and color confidence): 15 pts
- Combine Shape and Color Models: 5 pts
- Merge Local Windows: 10 pts
- Extract final foreground mask: 20 pts

For **extra credit**, implement any remaining part of the SnapCut paper (other than mentioned above). Be innovative!
Try to make your system robust as much as possible! Hint: Try to have additional image features, something beyond shape and color!


## Project Files

When running `MyRotobrush.m`, it must load a set of frames from the folder `Frames/`. Assume
that the frames will be `jpeg` images of the form `1.jpg`, `2.jpg`, etc. Your program must then
prompt the user to specify a region of interest in the first frame (for example using the
`roipoly` tool), and then track the specified object through the rest of the frames. For each
frame, draw the tracked boundary in red and save the result with the same filename in
`Output/`.

#### Functions Allowed 

For this project, roipoly, fitgmdist, estimateGeometricTransform, opticalFlowFarneback, vl_sift (http://www.vlfeat.org/overview/sift.html), imfilter, conv2, imrotate, im2double, rgb2gray, fspecial, imtransform, imwarp (and imref2d), meshgrid, sub2ind, ind2sub and all other plotting and matrix operation/manipulation functions are allowed.

<a name='sub'></a>
## Submission Guidelines
<b> We will deduct points if your submission does not comply with the following guidelines.</b>

### File tree and naming

Your submission on Canvas must be a zip file, following the naming convention **YourDirectoryID_proj3.zip**.  For example, xyz123_proj3.zip.  The file **must have the following directory structure**: 

- `YourDirectoryID_proj3.zip/`
    - `Code/`
        - `MyRotobrush.m`
        - *(any dependencies of MyRotobrush.m)
    - `Input/`
    - `Output/`
    - `result1.mp4`
    - `result2.mp4`
    - `result3.mp4`
    - `result4.mp4`
    - `result5.mp4`
    - `report.pdf`


When run, `MyRotobrush.m` must load a set of images from *Images/Input/*, and return
the resulting panorama.

## Output Videos

You have been provided the frames from five video clips. Run your code on each set of
frames and create videos, named as indicated above. For each video track the following:

- Frames1: Track the turtle. <a href="">Source</a>: An underwater video captured by Chahat at Lakshadweep islands, India.
- Frames2: Track the motorcycle and rider. <a href="https://www.youtube.com/watch?v=jhxCJLm4C-0">Source</a>.
- Frames3: Track the gymnast. <a href="https://www.youtube.com/watch?v=gMNCwgXtQIw">Source</a>.
- Frames4: Track the powerlifter, without the weights. <a href="https://www.youtube.com/watch?v=EiqEFUFM-KI">Source</a>.
- Frames5: Track the lizard (including tail). <a href="https://www.youtube.com/watch?v=Mp1R4Lxoj5c">Source</a>.


### Report
**You will be graded primarily based on your report.**  
**The most important part is to understand and explain why your algorithm is working or not working on a given dataset. What do you think are the drawbacks of your code!?**

We want you to demonstrate an understanding of the concepts involved in the project, and to show the output produced by your code.

Include visualizations of the output of each stage in your pipeline (as shown in the system diagram
on page 2), and a description of what you did for each step.  Assume that we're familiar with the
project, so you don't need to spend time repeating what's already in the course notes.  Instead, focus on any interesting problems you encountered and/or solutions you implemented.

As usual, your report must be full English sentences, **not** commented code. There is a word limit of 1500 words and no minimum length requirement.

<a name='coll'></a>
## Collaboration Policy
We encourage you to work closely with your groupmates, including collaborating on writing code.  With students outside your group, you may discuss methods and ideas but may not share code.  

For the full collaboration policy, including guidelines on citations and limitations on using online resources, see <a href="http://prg.cs.umd.edu/cmsc426">the course website</a>.
