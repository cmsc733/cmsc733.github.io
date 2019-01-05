---
layout: page
mathjax: true
title: Linear Least Squares
permalink: /2018/hw/hw1/
---
**This article is written by [Chethan Parameshwara](http://analogicalnexus.github.io).**

Table of Contents:
- [Due Date](#due)
- [Introduction](#intro)
- [What you need to do](#problem)
  - [Problem Statement](#pro)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Due Date 
10:59AM, Thursday, September 6, 2018

<a name='intro'></a>
## Introduction

This home work is designed to test your understanding of mathematics tutorial discussed in this [link](https://cmsc426.github.io/math-tutorial/). The task is to fit the line to two dimensional data points using different linear least square techniques discussed in the tutorials:

- Line fitting using Linear Least Squares
- Outliers rejection using Regularization
- Outliers rejection using RANSAC

<a name='problem'></a>
## What you need to do

The 2D points data is provided in the form of .mat file (click [here](https://drive.google.com/file/d/1B90tU1-HrR2YMjZC50APiNkQdLtVco6z/view?usp=sharing) to download). The visualization of data with different noise level is shown in the following figure.

<div class="fig fighighlight">
  <img src="/assets/hw1/data.jpg" width="100%">
  <div class="figcaption">
  </div>
  <div style="clear:both;"></div>
</div>

<a name='pro'></a>
### Problem Statement 

- Write matlab code to visualize geometric interpretation of eigenvalues/covariance matrix as discussed in this [link](http://www.visiondummy.com/2014/04/geometric-interpretation-covariance-matrix/) [40 points]  
- Decide the best outlier rejection technique for each of these datasets and write matlab code to fit the line. Also, discuss why your choice of technique is optimal [60 points] 

<a name='sub'></a>
## Submission Guidelines

<b> If your submission does not comply with the following guidelines, you'll be given ZERO credit </b>

### File tree and naming

Your submission on Canvas must be a zip file, following the naming convention **YourDirectoryID_hw1.zip**.  For example, xyz123_hw1.zip.  The file **must have the following directory structure**, based on the starter files

YourDirectoryID_hw1.zip.
 - data/. 
 - plot_eigen.m.
 - least_square.m.
 - report.pdf


### Report
For each section of the homework, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented.  You must include the following details in your writeup:

- Your understanding of eigenvectors and eigenvalues
- Your choice of outlier rejection technique for each dataset
- Limitation of each outliers rejection technique


As usual, your report must be full English sentences,**not** commented code. There is a word limit of 750 words and no minimum length requirement

<a name='coll'></a>
## Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup.  For the full honor code refer to the CMSC426 Fall 2018 website
