---
layout: page
mathjax: true
title: MyAutoPano
permalink: /2019/proj/p1/
---

Table of Contents:
- [Deadline](#due)
- [Problem Statement](#prob)
- [Phase 1: Traditional Approach](#ph1)
- [Phase 2: Deep Learning Approach](#ph2)
	- [Data Generation](#datagen)
	- [Supervised Approach](#ph2sup)
	- [Unsupervised Approach](#ph2unsup)
- [Notes about Test Set](#testset)
- [Extra Credit](#extra)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Deadline 
11:59 PM, Thursday, February 21, 2019


<a name='prob'></a>
## Problem Statement 



<a name='ph1'></a>
## Traditional Approach

<div class="fig fighighlight">
  <img src="/assets/2019/p1/TraditionalOverview.png" width="100%">
  <div class="figcaption">
  	Overview of panorama stitching using traditional method.
  </div>
  <div style="clear:both;"></div>
</div>

<a name='ph2'></a>
## Deep Learning Approach

You are going to be implementing two deep learning approaches to estimate the homography between two images. The deep model effectively combines corner detection, ANMS, feature extraction, feature matching, RANSAC and estimate homography all into one. This not only makes the approach faster but also makes it robust if the network is generalizable (More on this [later](#ph2unsup)).

<a name='datagen'></a>
## Data Generation
To train a Convolutional Neural Network (CNN) to estimate homography between a pair of images (we call this network *HomographyNet* and the original paper can be found [here](https://arxiv.org/pdf/1606.03798.pdf)) we need data (pairs of images) with the known homogaraphy between them. This is in-general hard to obtain as we would need the 3D movement between the pair of images to obtain the homography between them. An easier option is to generate synthetic pairs of images to train a network. But what images do we use so that the network is not baised? Simple, use images from [MSCOCO dataset](http://cocodataset.org/#home) which contains images of a lot of objects in natural scenes. MSCOCO is quite large and it'll take forever to train on these images. Hence, we provide a small subset of MSCOCO for you to train your HomographyNet on. This dataset can be downloaded from [here](). 

Now that you've downloaded the dataset, we need to generate synthetic data, i.e., pairs of images with known homography between them. Before, we generate image pairs, we need all the image pairs to be of the same size (as HomographyNet is not fully convolutional, it cannot accept image sizes of arbitrary shape). First step in generating data is to obtain a random crop of the image (called patch). Then the original image will be warped using a random homography then the respective patch is extracted. While we perform this operation we need to ensure that we are not extracting the patch from outside the image after warping. An illustration is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p1/ActiveRegion.png" width="100%">
  <div class="figcaption">
  	Patch is shown as dashed blue box. Active region is shown as a blue highlight -- this is the region where the top left corner of the patch can lie such that all the pixels in the patch will lie within the image after warping the random extracted patch. Red line shows the maximum perturbation \(\rho\). 
  </div>
  <div style="clear:both;"></div>
</div>

Let's go through the steps of generating data now. 

In step 1, obtain a random patch (\\(P_A\\) of size \\(M_P\times N_P\\)) from the image (\\( I_A\\)) of size \\( M\times N\\)) with \\( M>M_P\\) and \\(N>N_P \\) such that all the pixels in the patch will lie within the image after warping the random extracted patch. Think about where you have to extract the patch \\(P_A\\) from in \\( I_A\\) if maximum possible perturbation is \\([-\rho, \rho]\\). 

In step 2, perform a random perturbation in the range \\([-\rho, \rho]\\) of the corner points (top left corner, top right corner, left bottom corner and right bottom corner -- *not the corners in computer vision sense*) of \\(P_A\\) in \\( I_A\\). This is illustrated in the figures below.

<div class="fig fighighlight">
  <img src="/assets/2019/p1/I1Patch.png" width="100%">
  <div class="figcaption">
  	Extracted random patch \(P_A\) from original image \(I_A\) is shown as dashed blue box. Corners of the patch \(P_A\) denoted by \(C_A\) are are shown as blue circles.   
  </div>
  <div style="clear:both;"></div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p1/PerturbPts.png" width="100%">
  <div class="figcaption">
  	Random perturbation applied to corners of the patch \(P_A\) denoted by \(C_A\) are shown as blue circles to obtain corners of patch \(P_B\) denoted by \(C_B\) are shown as red circles. 
  </div>
  <div style="clear:both;"></div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p1/PerturbPtsImg.png" width="100%">
  <div class="figcaption">
  	Red dashed lines show the patch formed by perturbed point corners. Notice how the shape is not rectangular making it hard to extract data without adding extra data or masking.
  </div>
  <div style="clear:both;"></div>
</div>

As mentioned in the last figure, we want to extract data such that we add minimal amount of extra data and we don't want to mask anything or add black pixels (pixels outside image assuming zero padding). In order to do this, we'll warp the image \\(I_A\\) with the inverse of homography between \\(C_A\\) and \\(C_B\\) (denoted by \\(H_B^A\\)), i.e., which is the homography between \\(C_B\\) and \\(C_A\\) (denoted by \\(H_A^B\\)). Refer to ``cv2.getPerspectiveTransform`` and ``np.linalg.inv`` to implement this part. 

In step 3, use the value of \\(H_A^B\\) to warp \\(I_A\\) and obtain \\(I_B\\). Refer  to ``cv2.warpPerspective`` to implement this part. Now, we can extract the patch \\(P_B\\) using the corners in \\(C_A\\) (work the math out and convince yourself why this is true). This is shown in the figure below.

<div class="fig fighighlight">
  <img src="/assets/2019/p1/I2Patch.png" width="100%">
  <div class="figcaption">
  	Red dashed lines show the patch \(P_B\) extracted from warped image \(I_B\).
  </div>
  <div style="clear:both;"></div>
</div>

Now, the extracted patches are shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p1/Patches.png" width="100%">
  <div class="figcaption">
  	Extracted Patches \(P_A\) and \(P_B\) with known homography between them. 
  </div>
  <div style="clear:both;"></div>
</div>

Note that, we generated labels (ground truth homography between the two patches) as is given by \\(H_A^B\\). However, the authors in [this paper](https://arxiv.org/pdf/1606.03798.pdf) found that regressing the 9 values of the homography matrix directly yielded bad results. Instead, another way the authors found was to regress the amount the corners of the patch \\(P_A\\) denoted by \\(C_A\\) need to be moved so that they are aligned with \\(P_B\\). This is denoted by \\(H_{4Pt}\\) and will be used as our labels. Remember,  \\(H_{4Pt}\\) is given by \\(H_{4Pt} = C_B - C_A\\). 

Now, we stack the image patches \\(P_A\\) and \\(P_B\\) depthwise to obtain an input of size \\(M_P\times N_P \times 2*K\\) where \\(K\\) is the number of channels in each patch/image (3 if RGB image and 1 if grayscale image). 

The final output of data generation are these stacked image patches and the homography between them given by \\(H_{4Pt}\\). An overview of panorama stitching using Deep Learning (both supervised and unsupervised have the same overview) is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p1/DLOverview.png" width="100%">
  <div class="figcaption">
  	Overview of panorama stitching using deep learning based method.
  </div>
  <div style="clear:both;"></div>
</div>


<a name='ph2sup'></a>
## Supervised Approach

The network architecture and the overview is shown below. *However, note that you are free to change the network as you wish.*

<div class="fig fighighlight">
  <img src="/assets/2019/p1/HomographyNetSup.png" width="100%">
  <div class="figcaption">
  	Top: Overview of the supervised deep learning system for homography estimation. Bottom: Architecture of the network which mimics the network given in <a href="https://arxiv.org/pdf/1606.03798.pdf">this paper</a>.
  </div>
  <div style="clear:both;"></div>
</div>

The loss function used is simple L2 loss between the predicted 4-point homography \\(\widetilde{H_{4Pt}}\\) and ground truth 4-point homography \\(H_{4Pt}\\). The loss function \\(l\\) is given by \\(l = \vert \vert \widetilde{H_{4Pt}} - H_{4Pt} \vert \vert_2\\). 
 
<a name='ph2unsup'></a>
## Unsupervised Approach
Though the supervised deep homography estimation works well, it needs a lot of data to generalize well and is generally not robust if good data augmentation is not provided as shown in [this paper](https://arxiv.org/abs/1709.03966). Now, we'll implement an unsupervised approach to estimate homography between image pairs using a CNN as given in [this paper](https://arxiv.org/abs/1709.03966). 

The data generation is exactly the same for this part (as we want to evaluate the algorithm later using the labels), note that the labels \\(H_{4Pt}\\) are not used for learning in the loss function. For the unsupervised method, we want to predict the value of \\(H_{4Pt}\\) which when used to warp the patch \\(P_A\\) should look exactly like the patch \\(P_B\\). This can be written as the photometric loss function \\(l\\) given by \\(l = \vert \vert w(P_A, H_{4Pt}) - P_B\vert \vert_1\\). Here \\(w(\cdot)\\) represents a generic warping function and here specifically warping using the Homography value and bilinear interpolation. Let's see how we can convert our supervised method into unsupervised. 


<div class="fig fighighlight">
  <img src="/assets/2019/p1/HomographyNetUnsup.png" width="100%">
  <div class="figcaption">
    Overview of the unsupervised deep learning system for homography estimation.
  </div>
  <div style="clear:both;"></div>
</div>

Notice that, now we have two new parts (shown in light blue and yellow), namely, TensorDLT and Spatial Transformer Network. The TensorDLT part takes as input the predicted 4-point homography \\(\widetilde{H_{4Pt}}\\) and corners of patch \\(P_A\\) to estimate the full \\(3\times 3\\) estimated homgraphy matrix \\(\widetilde{\mathbf{H}}\\) which is used to warp the patch \\(P_A\\) using a differentiable warping layer (using bilinear interpolation) called the Spatial Transformer Network. To implement the TensorDLT layer, refer to [Section IV B in this paper](https://arxiv.org/abs/1709.03966). A generic Spatial Transformer Network is given to you with the starter code (you need to modify it as descibed in [Section IV C of this paper](https://arxiv.org/abs/1709.03966)). Finally, implement the photometric loss function we discussed earlier. 

Note that, we didn't talk about the network architecture here, feel free to use the network you implemented for the supervised version here or be creative.

<a name='testset'></a>
## Notes about Test Set
Six hours before the deadline, a test set will be released on which we expect you to run your code from both the parts and present the results in a report (more on this [later](#sub)). For the deep learning part, your algorithm will only run on the image size you chose during training, i.e., \\(M_P\times N_P\\). A simple way to deal with this is to resize the test image to \\(M_P\times N_P\\) to obtain the homography and then warp the original image or crop a central region of \\(M_P\times N_P\\) or obtain random crops of size \\(M_P\times N_P\\) and average all the predicted homography values. *Feel free to be creative here.*

<a name='extra'></a>
## Extra Credit
Implementing the extra credit can give you upto 25% on the bonus score. As we discussed [earlier](#testset), there is no optimal way to obtain the best homography between two images as we trained our networks on a small patch size. A good way would be to obtain homographies from a lot of patches and then use RANSAC to obtain the best method. We want you to implement this method using deep learning and present a high quality analysis. Refer to the [DSAC paper](https://arxiv.org/abs/1611.05705) which presents ideas on how to implement RANSAC in a differentiable way for a neural network. You are free to use the DSAC [code from the authors](https://hci.iwr.uni-heidelberg.de/vislearn/research/scene-understanding/pose-estimation/#DSAC) to implement this part. 

<a name='sub'></a>
## Submission Guidelines

<b> If your submission does not comply with the following guidelines, you'll be given ZERO credit </b>

### File tree and naming

Your submission on Canvas must be a zip file, following the naming convention **YourDirectoryID_proj1.zip**. **YourDirectoryID** is the name your username which is used to login to UMD resources such as Testudo or ELMS. For example, if you use "xyz" to login to Testyudo or ELMS then the zip file must be called "xyz_proj1.zip".  The file **must have the following directory structure**. 

YourDirectoryID_proj1.zip.
 - results/.
 - GMM.m
 - trainGMM.m
 - testGMM.m
 - measureDepth.m
 - plotGMM.m
 - report.pdf

### Report
For each section of the project, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented.  You must include the following details in your writeup:


<a name='coll'></a>
## Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup. For the full honor code refer to the CMSC733 Spring 2019 website.

