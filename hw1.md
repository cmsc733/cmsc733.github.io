---
layout: page
mathjax: true
title: AutoCalib
permalink: /2019/hw/hw1/
---

Table of Contents:
- [Due Date](#due)
- [Data](#data)
- [Initial Parameter Estimation](#init)
  - [Solving for approximate $$K$$ or camera intrinsic matrix](#solveK)
  - [Estimate approximate $$R$$ and $$t$$ or camera extrinsics](#solveRT)
  - [Approximate Distortion $$k_c$$](#solvedist)
- [Non-linear Geometric Error Minimization](#nonlinmin)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Due Date 
10:59AM, Thursday, September 6, 2018

<a name='intro'></a>
## Introduction

Estimating parameters of the camera like the focal length, distortion coefficients and principle point is called **Camera Calibration**. It is one of the most time consuming and important part of any computer vision research involving 3D geometry. An automatic way to perform efficient and robust camera calibration would be wonderful. One such method was presented Zhengyou Zhang of Microsoft in [this paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr98-71.pdf) and is regarded as one of the hallmark papers in camera calibration. Recall that the camera calibration matrix $$K$$ is given as follows


$$
K = \begin{bmatrix} 
f_x & 0 & c_x\\
0 & f_y & c_y\\
0 & 0 & 1\\
\end{bmatrix}
$$

and radial distortion parameters are denoted by $$k_1$$ and $$k_2$$ respectively. Your task is to estimate $$f_x, f_y, c_x, c_y, k_1, k_2$$.

<a name='data'></a>
## Data
The Zhang's paper relies on a calibration target (checkerboard in our case) to estimate camera intrinsic parameters. The calibration target used can be found in the file ``checkerboardPattern.pdf``. This was
printed on an A4 paper and the size of each square was 21.5mm. Note that the $$Y$$ axis has odd number of squares and $$X$$ axis has even number of squares. It is a general practice to neglect
the outer squares (extreme square on each side and in both directions). Thirteen images taken from a Google Pixel XL phone with focus locked are given in the folder ``./Code/Imgs`` which you will use to calibrate.


<a name='init'></a>
## Initial Parameter Estimation
We are trying to get a good initial estimate of the parameters so that we can feed it into the non-linear optimizer. We will define the parameters we are using in the code next.

$$x$$ denotes the image points, $$X$$ denotes the world points (points on the checkerboard), $$k_s$$ denotes the radial distortion parameters, $$K$$ denotes the camera calibration matrix, $$R$$ and $$t$$ represent the rotation matrix and the translation of the camera in the world frame.

<a name='solveK'></a>
### Solving for approximate $$K$$ or camera intrinsic matrix
Refer to Section 3.1 in [this paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr98-71.pdf) for a solution of parameters in $$K$$. Use ``cv2.findChessboardCorners`` function in OpenCV to find the corners of the Checker board with appropriate parameters here.


<a name='solveRT'></a>
### Estimate approximate $$R$$ and $$t$$ or camera extrinsics
Refer to Section 3.1 in [this paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr98-71.pdf)  for details on how to estimate $$R$$ and $$t$$. Note that the author mentions a method to convert a normal matrix to a rotation matrix in Appendix C, this can be neglected most of the times.

<a name='solvedist'></a>
### Approximate Distortion $$k_c$$
Because we assumed that the camera has minimal distortion we can assume that $$k_c = [0, 0]^T$$ for a good initial estimate.

<a name='nonlinmin'></a>
## Non-linear Geometric Error Minimization
We have the initial estimates of $$K, R, t, k_s$$, now we want to minimize the geometric error defined as given below

$$
\sum_{i=1}^N \sum_{j=1}^M \vert \vert x_{i,j} - \hat x_{i,j}\left(K, R_i, t_i, X_j, k_s \right)\vert \vert
$$

Here $$x_{i,j}$$ and $$\hat x_{i,j}$$ are an inhomogeneous representation. Feel free to use ``scipy.optimize`` to minimize the loss function described above. Refer to Section 3.3 in [this paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr98-71.pdf) for a detailed explanation of the distortion model, you'll need this part for the minimization function.


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
