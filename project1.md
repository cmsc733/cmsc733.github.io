---
layout: page
mathjax: true
title: Color Segmentation using GMM
permalink: /2018/proj/p1/
---
**This article is written by [Chethan Parameshwara](http://analogicalnexus.github.io).**
Please contact **Chethan Parameshwara** if there are any errors.

Table of Contents:
- [Deadline](#due)
- [Introduction](#intro)
- [What you need to do](#problem)
  - [Problem Statement](#pro)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Deadline 
10:59AM, Tuesday, September 18, 2018

<a name='intro'></a>
## Introduction

Have you ever played with these adorable Nao robots? Click on the image to watch a cool demo.  

<a href="http://www.youtube.com/watch?feature=player_embedded&v=Gy_wbhQxd_0
" target="_blank"><img src="http://img.youtube.com/vi/Gy_wbhQxd_0/0.jpg" 
alt=" Nao robot demo " width="480" height="360" border="0" /></a>


Nao robots are star players in RoboCup, an annual autonomous robot soccer competitions. 
We are planning to build the Maryland RoboCup team to compete in RoboCup 2019, we need your help. 
Would you like to help us in Nao's soccer training? We need to train Nao to detect a soccer ball and estimate the depth of the ball to know how far to kick.

Nao's training has two phases:
- Color Segmentation using Gaussian Mixture Model (GMM)
- Ball Distance Estimation

<a name='problem'></a>
## What you need to do
To make logistics easier, we have collected camera data from Nao robot on behalf of you and saved the data in the form of color images. Click [here](https://drive.google.com/file/d/17XiM86JqHqko4JC00-E4w4sPKnzh2iMz/view?usp=sharing) to download. The image names represent the depth of the ball from Nao robot in centimeters. We will release the test dataset 24 hours before the deadline i.e. 10:59AM, Monday, September 17. **Test images are released. Click [here](https://drive.google.com/file/d/17tNn3YIVR-8kqoBgJNK58YY4UBnQmm4q/view?usp=sharing) to download**. 

<a name='pro'></a>
### Problem Statement 

1. Write MATLAB code to cluster the orange ball using [Single Gaussian](https://cmsc426.github.io/colorseg/#gaussian) [30 points] 
2. Write MATLAB code to cluster the orange ball using [Gaussian Mixture Model](https://cmsc426.github.io/colorseg/#gmm) [40 points] and estimate the [distance](https://cmsc426.github.io/colorseg/#distest) to the ball [20 points]. Also, plot all the GMM ellipsoids [10 points]. 

You are **NOT** allowed to use any built-in MATLAB function(s) like `fitgmdist()` or `gmdistribution.fit()` for GMM. To help you with code implementation, we have given the pseudocode :-) 


<div class="fig fighighlight">
  <img src="/assets/proj1/proj1_image.PNG" width="100%">
  <div class="figcaption">
  </div>
  <div style="clear:both;"></div>
</div>

<a name='sub'></a>
## Submission Guidelines

<b> If your submission does not comply with the following guidelines, you'll be given ZERO credit </b>

### File tree and naming

Your submission on Canvas must be a zip file, following the naming convention **YourDirectoryID_proj1.zip**.  For example, xyz123_proj1.zip.  The file **must have the following directory structure**. 

YourDirectoryID_proj1.zip.
 - train_images/.
 - test_images/.
 - results/.
 - GMM.m
 - trainGMM.m
 - testGMM.m
 - measureDepth.m
 - plotGMM.m
 - report.pdf

### Report
For each section of the project, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented.  You must include the following details in your writeup:

- Your choice of color space, initialization method and number of gaussians in the GMM
- Explain why GMM is better than single gaussian 
- Present your distance estimate and cluster segmentation results for each test image
- Explain strengths and limitations of your algorithm. Also, explain why the algorithm failed on some test images

As usual, your report must be full English sentences, **not** commented code. There is a word limit of 1500 words and no minimum length requirement

<a name='coll'></a>
## Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup. For the full honor code refer to the CMSC426 Fall 2018 website.

