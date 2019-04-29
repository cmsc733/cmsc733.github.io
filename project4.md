---
layout: page
mathjax: true
title: Buildings built in minutes - An SfM Approach
permalink: /2019/proj/p4/
---


Table of Contents:

- [1. Deadline](#due)
- [2. Introduction](#intro)
- [3. SfMLearner](#sfmlearner)
   	- [3.1. View Synthesis](#view)
   	- [3.2. Differentiable Depth Image-based Rendering](#render)
   	- [3.3. Explainability Mask](#expl-mask)
   	- [3.4. Gradient Locality Issues](#gradlocal)
- [4. Notes about the dataset](#testset)
- [5. Submission Guidelines](#sub)
  - [5.1. File tree and naming](#files)
  - [5.2. Report](#report)
  - [5.3. Video Presentation](#video)
- [6. Collaboration Policy](#coll)

<i>To be submitted in a group.</i>

<a name='due'></a>

## 1. Deadline 

**11:59PM, May 16, 2019.**

<a name='intro'></a>

## 2. Introduction

We have dealt with reconstructing 3D structure of a given scene using images from multiple views using the traditional (geometric) approach. Though, there is a possibility of achieving more robust results. In this project, we will learn about estimating depth and pose (or ego-motion) from a sequence of images using unsupervised learning methods. In [SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf) paper by David Lowe's team at Google, an unsupervised learning framework was presented for the task of monocular depth and camera motion estimation from unstructured video sequences. 

Your task is to make [SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf) 'better'! When we say better, it means that the error in depth and pose estimation should be less than that of SfMLearner's paper on different empirical evaluation scales. Research papers like [GeoNet](https://arxiv.org/pdf/1803.02276.pdf) might help you in improving the results. Although, research papers in this field are in abundance. Feel free to search through the internet. 

You will be mainly graded on the analysis of your approach and 'your original' implementation to make the SfMLearner better! Along with the standard report, you will be submitting a presentation video of about 5-7 mins long, explaining your approach and the in-depth analysis of your methods and the results. More on submission details are mentioned below in [section 5](#sub).

Note: You don't have to reimplement SfMLearner again! You will be not graded for that. Use the SfMLearner code on [Github](https://github.com/tinghuiz/SfMLearner) by the original authors. Feel free to modify or use any code available online to make the results better but DO NOT forget to cite them. You are restricted to $\sim$ 20K images provided in the dataset for training.


[SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf)

<a name='sfmlearner'></a>

## 3. SfMLearner

One of the trivial ways to solve this problem is to learn rotation and translation from a sequence of data. Although, learning such parameters directly is a weakly constrained problem. Thus, methods like [SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf), jointly train a single-view depth CNN and a camera pose estimation CNN from unlabeled video sequence. 

<b> You are required to read [SfMLearner](https://people.eecs.berkeley.edu/~tinghuiz/projects/SfMLearner/cvpr17_sfm_final.pdf) paper before reading further. </b>

Following the a brief summary of the proposed framework in SfMLearner for jointly training a single-view depth CNN and a camera pose estimation CNN from unlabeled video sequences.  
<b>Assumption: The scene is fairly rigid <i>i.e.</i> the scene appearance change across different frames is dominated by the camera motion.</b>


<a name='view'></a>

### 3.1. View Synthesis
The key supervision signal for our depth and pose prediction CNNs comes from the task of novel view synthesis: given one input view of a scene, synthesize a new image of the scene seen from a different camera pose. We can synthesize a target view given a per-pixel depth in that image, plus the pose and visibility in a nearby view. Thus, synthesizing the target view works as a supervision for training. The view synthesis objective can be formulated as:

$$ L_{vs} = \sum_s \sum_p |I_t(p) - \hat{I_s}(p)|$$ 
where $$p$$ indexes over pixel coordinates and $$\hat{I_s}$$ is the source view $$I_s$$ warped to the target coordinate frame based on a depth imae-based rendering module. See Figure xx for an illustration of SfMLearner learning pipeline for depth and pose estimation. 

<div class="fig fighighlight">
  <img src="/assets/2019/p4/overview.png"  width="80%">
  <div class="figcaption">
  Figure 1: Overview of the supervision pipeline based on view synthesis.
  </div>
  <div style="clear:both;"></div>
</div>


### 3.2. Differentiable Depth Image-based Rendering
A key component of this framework is a differentiable depth image-based renderer (Refer section 3.1 of the paper) that reconstructs the target view $$I_t$$ by sampling pixels from a source view $$I_s$$ based on the predicted depth map $$\hat{D_t}$$ and the relative pose $$\hat{T}_{t\rightarrow s}$$

Fig. xx is an illustration of the differentiable image warping process. 
<div class="fig fighighlight">
  <img src="/assets/2019/p4/image-warp.png"  width="80%">
  <div class="figcaption">
  Figure 2: For each point \(p_t\) in the target view, we first project it onto the source view based on the predicted depth and camera pose, and then use bilinear interpolation to obtain the value of the warped image \(\hat{I}_s\) at location \(p_t\).
  </div>
  <div style="clear:both;"></div>
</div>

To obtain $$I_s(p_s)$$ for populating the value of $$\hat{I}_s(p_t)$$, we use the differentiable bilinear sampling mechanism proposed in the spatial transformer networks(STN) that linearly interpolates the values of the 4-pixel neighboring pixels of $$p_s$$ to approximate $$I_s(p_s)$$. For this project, you may 'use' STN rather than writing your own. A sample tensorflow implementation of STN can be found [here](https://github.com/kevinzakka/spatial-transformer-network). Feel free to use any other implementation.


### 3.3. Explainability Mask

<b>Note</b>: Till now, we assume the scene is static; there are no occlusions between the target and source views; the surface is Lambertian so that the photo-consistency error is meaningful. With any of the these assumptions voilated in the training sequence, the gradients could be corrupted. To improve the robustness of our training process, we separately train a <i>explainability prediction</i> network that outputs a per-pixel soft mask $$\hat{E}_s$$ for each  target-source pair. 

Thus, the view synthesis objective is updated as:

$$ L_{vs} = \sum_s \sum_p \hat{E}_s(p)\ |I_t(p) - \hat{I_s}(p)|$$ 


### 3.4. Gradient Locality Issues

There is still another problem that needs to be dealt with. The gradients in the framework are mainly derived from the pixel intensity difference between the center pixel and its neighbours which will cause problems in training if the correct $$p_s$$ (neighbour pixel) is located in a low-texture region. This is a common issue in motion estimation. To overcome this problem, we can use a encoder-decoder CNN with a small bottleneck for the depth network that implicitly constrains the output to be globally smooth and facilitates gradients to propogate from meaningful regions to nearby regions. 
Thus, for smoothness, we minimize the $$L_1$$ norm of second-order gradients for the predicted depth maps:

$$L_{final} = \sum_l L^l_{vs} + \lambda_s L^l_{smooth} + \lambda_e \sum_s L_{reg}(\hat{E}^l_s)$$

where $$l$$ indexes over different images scales, $$s$$ indexes over source images and $$\lambda_s$$ and $$\lambda_e$$ are weighting for the depth smoothness loss and explainability regularization respectively.

The network architecture is given below:

<div class="fig fighighlight">
  <img src="/assets/2019/p4/network.png"  width="100%">
  <div class="figcaption">
  Figure 3: Network architecture for SfMLearner's depth/pose/explainability prediction modules.
  </div>
  <div style="clear:both;"></div>
</div>


<a name='testset'></a>

## 4. Notes about Data

You are given a multiple sequences from the [KITTI](http://www.cvlibs.net/datasets/kitti/raw_data.php) dataset. You can download the data from [here]().


<a name='sub'></a>

## 5. Submission Guidelines

<b> If your submission does not comply with the following guidelines, you'll be given ZERO credit </b>

<a name='files'></a>

### 5.1. File tree and naming

Your submission on ELMS/Canvas must be a ``zip`` file, following the naming convention ``YourDirectoryID_p4.zip``. If you email ID is ``abc@umd.edu`` or ``abc@terpmail.umd.edu``, then your ``DirectoryID`` is ``abc``. For our example, the submission file should be named ``abc_p1.zip``. The file **must have the following directory structure** because we'll be autograding assignments. You can have any helper functions in sub-folders as you wish, be sure to index them using relative paths and if you have command line arguments for your Wrapper codes, make sure to have default values too. Please provide detailed instructions on how to run your code in ``README.md`` file. Please **DO NOT** include data in your submission.

```
YourDirectoryID_hw1.zip
│   README.md
|   Code 
|   ├── Train.py
|   ├── Test.py
|   ├── Any subfolders you want along with files 
|   Data
|   ├── AnyOutputImagesYouWantToHighlight.png
|   ├── Data2OutputPRNet.mp4
├── Report.pdf
└── PresentationVideo.mp4
```
<a name='report'></a>

### 5.2. Report

For each section of newly developed solution in the project, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented.  You must include the following details in your writeup:

- Your report **MUST** be typeset in LaTeX in the IEEE Tran format provided to you in the ``Draft`` folder and should of a conference quality paper.
- Brief explanation of your approach to this problem.
- Present a set of images for comparison of depth estimation of SfMLearner, YourMethod and Ground Truth (with the input RGB image).
- Present a odometry comparison of pose estimation of SfMLearner, YourMethod and Ground Truth.
- Present a comparison of error of SfMLearner and YourMethod in different error metric scale (Abs, Sq, RMSE and RMSE log) as mentioned in the paper. The scripts for computing error metrics for both pose and depth evaluation can be downloaded from [here](https://github.com/tinghuiz/SfMLearner/tree/master/kitti_eval). For this, train ONLY on the KITTI training set provided here and test it on $$(i)$$ [KITTI testing set]() $$(ii)$$ [CityScape testing set]()
- Present Training Accuracy on the provided KITTI training set. Present Testing accuracy on the provided KITTI testing set and the Cityscape testing set.
- Present an in-depth analysis of your proposed approach. Did you try something specific? If yes, why? Talk about why it performed better or worse?
- Present a 3D structure of the scene, reconstructed from depth and poses estimated. Feel free to directly use parts of your project 3 or any open source code for this.    


<a name='video'></a>

### 5.3 Video Presentation

You are required to submit a video explaining your approach to the given problem. Explain what all problems you tackled during this problem and how you overcame them. Also, give an in-depth analysis of your proposed approach. The video MUST be less 7 mins long. We expect the video to be in somewhere between 5-7 mins.


<a name='coll'></a>

## 6. Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup. For the full honor code refer to the CMSC733 Spring 2019 website.
