---
layout: page
mathjax: true
title: Image Features and Warping 
permalink: /2018/hw/hw2/
---

Table of Contents:
- [Due Date](#due)
- [Introduction](#intro)
- [Part 1: Coding](#part1)
- [Part 2: Short Answer](#part2)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Due Date 
10:59AM, Tuesday, October 2nd, 2018

<a name='intro'></a>
## Introduction

This homework reviews some important concepts related to Project 2 (Panorama Stitching).  It is composed
of a coding problem and two short answer questions.

<a name='part1'></a>
## Part 1: Coding - Harris Corner Detection (50 pts)

In Project 2, you'll be using Matlab's built-in corner detector, via the `cornermetric` function.  In this coding assignment, we'll ask you to implement a simple corner detector yourself: the "Harris" corner detector.

Check out <a href="http://www.cse.psu.edu/~rtc12/CSE486/lecture06.pdf">these slides from Penn State's CSE486</a> for an excellent overview of how the Harris Corner
Detector works.

### What you need to do.

Write a matlab function `corner_response=myharris(I, window_size, corner_thresh)` which implements Harris
corner detection.  To make things a bit easier, you don't need to implement the final non-maximal suppression step.  So, it should return a heatmap of corner response, for a given image, window size, and corner threshold. 

See "Submission Guidelines" for instructions on what to include in your report.

<a name='part2'></a>
## Part 2: Short Answer

### Question 1: Harris Corner Detector Properties (15 pts)

Corners are useful for matching features between different images of the same scene, which might have changes in lighting, viewpoint, etc.  Consider the following simple changes that we might expect to occur between two images of a given scene:

- image rotation
- image scaling
- increment all pixel values by a constant.
- multiply all pixel values by a constant.

Based on your understaning of the Harris Corner Detector, how robust do you think it would be to each of these changes?  e.g., if an image were rotated, would the Harris corner detector identify the same corners before and after rotation?

Answer briefly-- two paragraphs or fewer.


### Question 2:  Image Warping and Invertible Transformations (35 pts)
Given a digital image, and an invertible transformation $$\tilde{\textbf{H}}$$ of the form
$$
\tilde{p}' \equiv \tilde{H} \tilde{p}
$$
we would like to compute the warped image whereby each point in the original image is transformed to
its new location.

This type of image warping is exactly what the Matlab `imwarp` function does, for
example.

We could envision a somewhat straightforward algorithm for performing this image warp:
for each location $$\tilde{p}$$ in the original image, compute the nearest pixel location of the
transformed point $$\tilde{p}'$$ in the warped image, and copy the color found in $$\tilde{p}$$ into the
warped image at location $$\tilde{p}'$$.

However, the vastly preferable algorithm is to loop over the *destination* pixels
$$\tilde{p}'$$ in the warped image, and use the inverse transformation $$\tilde{H}^{-1}$$ to identify
the nearest pixel $$\tilde{p}$$ in the source image and copy the color from that source pixel to the
destination.

What is the difference between the two approaches? Why is the second one preferable?  Please answer
in no more than a paragraph.


<a name='sub'></a>
## Submission Guidelines

<b> We will deduct points if your homework doesn't follow these specifications. </b>

### File tree and naming

Your submission on Canvas must be a zip file, following the naming convention **YourDirectoryID_hw2.zip**.  For example, xyz123_hw2.zip.  The file **must have the following directory structure**, based on the starter files

YourDirectoryID_hw2.zip/

 - myharris.m
 - report.pdf


### Report

Please run your corner detector on <a
href="https://drive.google.com/file/d/11MJ_qPpmQwQ-kgnTrxsnfGQAqmHvQqZ6/view?usp=sharing">these images</a> (also used in project 2), for several different window sizes and corner threshold values, and include these results in your report.

Also include your responses to the short answer questions from part 2.

As usual, your report must be full English sentences,**not** commented code.

<a name='coll'></a>
## Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup.  For the full honor code refer to the CMSC426 Fall 2018 website
