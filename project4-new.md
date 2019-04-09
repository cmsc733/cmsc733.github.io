---
layout: page
mathjax: true
title: Buildings built in minutes - An SfM Approach
permalink: /2019/proj/p4-new/
---


Table of Contents:

- [4. Phase 2: Deep Learning Approach (SfMLearner)](#sfmlearner)
   	- [4.1. View Synthesis](#view)
   	- [4.2. Differentiable Depth Image-based Rendering](#render)
   	- [4.3. Explainability Mask](#expl-mask)
   	- [4.4. Gradient Locality Issues](#gradlocal)
- [5. Notes about Test Set](#testset)
- [6. Submission Guidelines](#sub)
  - [6.1. File tree and naming](#files)
  - [6.2. Report](#report)
- [7. Collaboration Policy](#coll)

<a name='due'></a>
## 1. Deadline 
**11:59PM, Thursday, April 22, 2019.**

<a name='intro'></a>
## 2. Introduction
We have been playing with images for so long, mostly in 2D scene. Recall [project 1](/2019/proj/p1) where we stitched multiple images with about 30-50% common features between a couple of images. Now let's learn how to **reconstruct a 3D scene and simultaneously obtain the camera poses** of a monocular camera w.r.t. the given scene. This procedure is known as Structure from Motion (SfM). As the name suggests, you are creating the entire **rigid** structure from a set of images with different view points (or equivalently a camera in motion). A few years ago, Agarwal et. al published [Building Rome in a Day](http://grail.cs.washington.edu/rome/rome_paper.pdf) in which they reconstructed the entire city just by using a large collection of photos from the Internet. Ever heard of Microsoft [Photosynth?](https://en.wikipedia.org/wiki/Photosynth) _Facinating? isn't it!?_ There are a few open source SfM algorithm available online like [VisualSFM](http://ccwu.me/vsfm/). _Try them!_ 


<a name='sfmlearner'></a>
## 4. Phase 2: Deep Learning Approach (SfMLearner)

To achieve more robust results, we are going to implement an unsupervised deep learning framework ([SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf)) to estimate monocular depth and camera motion from continuous set of frames. This is an end-to-end learning approach that will basically replace all the steps from feature matching to non linear PnP.   

One of the trivial ways to do it is to learn rotation and translation from a sequence of data. Although, learning such parameters directly is weakly constrained. Thus, methods like SfMLearner, jointly train a single view depth CNN and a camera pose estimation CNN from unlabeled video sequence. 

Assumption: The scene is fairly rigid <i>i.e.</i> the scene appearance change across different frames is dominated by the camera motion.

<b> You are required to read [SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf) paper before reading further. </b>

<a name='view'></a>
### 4.1. View Synthesis
The key supervision signal for our depth and pose prediction CNNs comes from the task of novel view synthesis: given one input view of a scene, synthesize a new image of the scene seen from a different camera pose. We can synthesize a target view given a per-pixel depth in that image, plus the pose and visibility in a nearby view. Thus, synthesizing the target view works as a supervision for training. The view synthesis objective can be formulated as:

$$ L_{vs} = \sum_s \sum_p |I_t(p) - \hat{I_s}(p)|$$ 
where $$p$$ indexes over pixel coordinates and $$\hat{I_s}$$ is the source view $$I_s$$ warped to the target coordinate frame based on a depth imae-based rendering module. See Figure xx for an illustration of SfMLearner learning pipeline for depth and pose estimation. 

<div class="fig fighighlight">
  <img src="/assets/2019/p3/overview.png"  width="80%">
  <div class="figcaption">
  Figure xx: Overview of the supervision pipeline based on view synthesis.
  </div>
  <div style="clear:both;"></div>
</div>


### 4.2. Differentiable Depth Image-based Rendering
A key component of this framework is a differentiable depth image-based renderer (Refer section 3.1 of the paper) that reconstructs the target view $$I_t$$ by sampling pixels from a source view $$I_s$$ based on the predicted depth map $$\hat{D_t}$$ and the relative pose $$\hat{T}_{t\rightarrow s}$$

Fig. xx is an illustration of the differentiable image warping process. 
<div class="fig fighighlight">
  <img src="/assets/2019/p3/image-warp.png"  width="80%">
  <div class="figcaption">
  Figure xx: For each point \(p_t\) in the target view, we first project it onto the source view based on the predicted depth and camera pose, and then use bilinear interpolation to obtain the value of the warped image \(\hat{I}_s\) at location \(p_t\).
  </div>
  <div style="clear:both;"></div>
</div>

To obtain $$I_s(p_s)$$ for populating the value of $$\hat{I}_s(p_t)$$, we use the differentiable bilinear sampling mechanism proposed in the spatial transformer networks(STN) that linearly interpolates the values of the 4-pixel neighboring pixels of $$p_s$$ to approximate $$I_s(p_s)$$. For this project, you may 'use' STN rather than writing your own. A sample tensorflow implementation of STN can be found [here](https://github.com/kevinzakka/spatial-transformer-network). Feel free to use any other implementation.


### 4.3. Explainability Mask

<b>Note</b>: Till now, we assume the scene is static; there are no occlusions between the target and source views; the surface is Lambertian so that the photo-consistency error is meaningful. With any of the these assumptions voilated in the training sequence, the gradients could be corrupted. To improve the robustness of our training process, we separately train a <i>explainability prediction</i> network that outputs a per-pixel soft mask $$\hat{E}_s$$ for each  target-source pair. 

Thus, the view synthesis objective is updated as:

$$ L_{vs} = \sum_s \sum_p \hat{E}_s(p)\ |I_t(p) - \hat{I_s}(p)|$$ 


### 4.4. Gradient Locality Issues

There is still another problem that needs to be dealt with. The gradients in the framework are mainly derived from the pixel intensity difference between the center pixel and its neighbours which will cause problems in training if the correct $$p_s$$ (neighbour pixel) is located in a low-texture region. This is a common issue in motion estimation. To overcome this problem, we can use a encoder-decoder CNN with a small bottleneck for the depth network that implicitly constrains the output to be globally smooth and facilitates gradients to propogate from meaningful regions to nearby regions. 
Thus, for smoothness, we minimize the $$L_1$$ norm of second-order gradients for the predicted depth maps:

$$L_{final} = \sum_l L^l_{vs} + \lambda_s L^l_{smooth} + \lambda_e \sum_s L_{reg}(\hat{E}^l_s)$$

where $$l$$ indexes over different images scales, $$s$$ indexes over source images and $$\lambda_s$$ and $$\lambda_e$$ are weighting for the depth smoothness loss and explainability regularization respectively.

The network architecture is given below:

<div class="fig fighighlight">
  <img src="/assets/2019/p3/network.png"  width="100%">
  <div class="figcaption">
  Figure xx: Overview of the supervision pipeline based on view synthesis.
  </div>
  <div style="clear:both;"></div>
</div>


<a name='testset'></a>
## 5. Notes about Test Set
One day (24 hours) before the deadline, a test set will be released with details of what faces to replace. We'll grade on the completion of the project and visually appealing results.

<a name='sub'></a>
## 6. Submission Guidelines

<b> If your submission does not comply with the following guidelines, you'll be given ZERO credit </b>

<a name='files'></a>
### 6.1. File tree and naming

Your submission on ELMS/Canvas must be a ``zip`` file, following the naming convention ``YourDirectoryID_p3.zip``. If you email ID is ``abc@umd.edu`` or ``abc@terpmail.umd.edu``, then your ``DirectoryID`` is ``abc``. For our example, the submission file should be named ``abc_p1.zip``. The file **must have the following directory structure** because we'll be autograding assignments. The file to run for your project should be called ``Wrapper.py``. You can have any helper functions in sub-folders as you wish, be sure to index them using relative paths and if you have command line arguments for your Wrapper codes, make sure to have default values too. Please provide detailed instructions on how to run your code in ``README.md`` file. Please **DO NOT** include data in your submission.

```
YourDirectoryID_hw1.zip
│   README.md
|   Your Code files 
|   ├── Any subfolders you want along with files
|   Wrapper.py 
|   Data
|   ├── Data1.mp4
|   ├── Data2.mp4
|   ├── Data1OutputTri.mp4
|   ├── Data1OutputTPS.mp4
|   ├── Data1OutputPRNet.mp4
|   ├── Data2OutputTri.mp4
|   ├── Data2OutputTPS.mp4
|   ├── Data2OutputPRNet.mp4
└── Report.pdf
```
<a name='report'></a>
### 6.2. Report

For each section of the project, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented.  You must include the following details in your writeup:

- Your report **MUST** be typeset in LaTeX in the IEEE Tran format provided to you in the ``Draft`` folder and should of a conference quality paper.
- Present the Data you collected in ``Data`` folder with names ``Data1.mp4`` and ``Data2.mp4`` (Be sure to have the format as ``.mp4`` **ONLY**).
- Present the output videos for Triangulation, TPS and PRNet as ``Data1OutputTri.mp4``, ``Data1OutputTPS.mp4`` and ``Data1OutputPRNet.mp4`` for Data 1 respectively in the ``Data`` folder. Also, present outputs videos for Triangulation, TPS and PRNet as ``Data2OutputTri.mp4``, ``Data2OutputTPS.mp4`` and ``Data2OutputPRNet.mp4`` for Data 2 respectively in the ``Data`` folder. (Be sure to have the format as ``.mp4`` **ONLY**).
- For Phase 1, present input and output images for two frames from each of the videos using both Triangulation and TPS approach.
- For Phase 2, present input and output images for two frames from each of the videos using PRNet approach.
- Present failure cases for both Phase 1 and 2 and present your thoughts on why the failure occurred. 

<a name='coll'></a>
## 7. Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup. For the full honor code refer to the CMSC733 Spring 2019 website.
