---
layout: page
mathjax: true
title: Sexy Semantic Mapping
permalink: /2019/proj/p4/
---

Table of Contents:

- [Introduction](#intro)
- [Extract object of interest from Table-Top Images](#table-top)
  - [Create a Point Cloud from RGB-D images](#create-pcl)  
  - [Build 3D model of object from RGB-D Images](#build-3D-model)  
  - [Extract the ‘Collection of Objects’ from a given scene](#extract)
      
- [Segment the scene using known Objects](#segment)
- [Build a Semantic Map](#semantic)
- [Stuff Given to you](#stuff)
      
- [Submission Guidelines](#guidelines)

<a name='intro'></a>

## 1. Introduction

The aim of this project is to build semantic map of a 3D scene (See Fig. 1). This project gives you a peek into the world of robotics perception <i>i.e.</i> how robots understand and build the model of the world. <i>Excited?</i>

<div class="fig fighighlight"><center>
  <img src="/assets/2019/p4/seg.png" width="60%"></center>
  <div class="figcaption">
    Figure 1: (a) Reconstruction from a samples of RGBD images. (b) Segmentation of a 3D scene.
  </div>
  <div style="clear:both;"></div>
</div>


In the next few sections, we will detail how this can be done along with the specifications of the functions for each part. <b>Just like the previous projects, this project is to be done in groups of two and only one submission is required per group.</b>

<a name='table-top'></a>

## 2. Extract object of interest from Table-Top Images

You are given a set of RGB-D (RGB-Depth) frames of table top images and you need to extract the 'object of interest' from each frame and build a 3D model of the filtered object. When we say 'extract object of interest', you need to remove the table, walls of the room (if any) and any such irrelevant data from the scene. You also might need to filter stray points during the extraction process. But before that we need to align the data from RGB camera and the depth sensor from the Kinect. See Fig xx. Clearly, the RGB image and Depth images are not aligned. 

<div class="fig fighighlight">
  <img src="/assets/2019/p4/rgb.png" width="49%">
  <img src="/assets/2019/p4/depth.png" width="49%">
  <div class="figcaption">
  Figure 3 (a): RGB and Depth images from Kinect.
  </div>
  <br><center>
  <img src="/assets/2019/p4/rgbd.png" width="75%"></center>
  <div class="figcaption">
  Figure 3 (b): Uncalibrated RGB-D images. Thus we need calibration parameters in order to align the two images.
  </div>
  <div style="clear:both;"></div>
</div>

<a name='create-pcl'></a>

### 2.1 Create a Point Cloud from RGB-D images


So, in order to align them we would need calibration parameters <i>i.e.</i> the rotation and translation between the camera centers of RGB camera and the depth sensor as $$(u,v)$$ does not represents $$(u',v')$$, see fig. xx. 

<div class="fig fighighlight"><center>
  <img src="/assets/2019/p4/RotAndTrans1.png" width="90%"></center>
  <div class="figcaption">
    Figure 2: .
  </div>
  <div style="clear:both;"></div>
</div>

To formulate: Given all camera parameters $$(R,t,f)$$ (rotation, translation and focal lengths of both sensors), find the corresponding points of a RGB and a depth image. Thus, we need to generate a point cloud that can be represented as $$(x,y,z,r,g,b)$$. (See Fig. xx)
<div class="fig fighighlight"><center>
  <img src="/assets/2019/p4/RotAndTrans2.png" width="60%"></center>
  <div class="figcaption">
    Figure 2: .
  </div>
  <div style="clear:both;"></div>
</div>


In order to solve this problem, first recall: $$\cfrac{u}{f_u} = \cfrac{x}{z}$$

Now, to generate a point cloud from RGB-D data, follow these steps:
1. Compute 3D coordinate $$X^{IR}$$ in the $$IR$$ camera frame. (IR: Infrared or depth sensor frame)
$$x^{IR} = \cfrac{uz}{f^{IR}}$$; $$\ \ y^{IR} = \cfrac{vz}{f^{IR}}$$; $$\ \ X^{IR} = [x^{IR} \ y^{IR} \ z^{IR}]$$
2. Transform into RGB frame
$$X^{RGB} = RX^{IR} + t$$
3. Reproject them into the image plane
$$u^{RGB} = f^{RGB}\cfrac{x^{RGB}}{z^{RGB}} = f^{RGB}\cfrac{y^{RGB}}{z^{RGB}}$$
4. Read $$(r,g,b)$$ at $$(u,v)^{RGB}$$
$$(r,g,b)$$ is the color of $$X^{IR}$$ point.

Now, once the point clouds are generated from a single view, let us learn how to build 3D model of the scene from multiple views.

<a name='build-3D-model'></a>

### 2.2 Build 3D model of object from RGB-D Images

The RGB-D data provided is recorded from an Asus Xtion Pro sensor. Using the method in the previous stage, you get the 3D view of the object from one single camera view point, use many such views from different frames of the same object to iteratively build a 3D model of the object.

<div class="fig fighighlight">
  <img src="/assets/2019/p4/data-collect.png" width="100%">
  <div class="figcaption">
    Figure 2: .
  </div>
  <div style="clear:both;"></div>
</div>


The idea here is you need to find out 3D translation and rotation parameters between two 3D point clouds of the same object from different frames, use these parameters to back project 3D point cloud from second frame on to the first and now you have little more information of the 3D model of the object as compared to the 3D point cloud from a single frame. You do it iteratively until you build a complete (or more or less complete) 3D model of the object.

You will need to implement any variant of the Iterative Closest Point (ICP) algorithm which gives the relative translation and rotation between two 3D point clouds. We have uploaded a lot of ICP related papers with this description for your reference, please cite the reference you follow.

<a name='extract'></a>

### 2.3 Extract the ‘Collection of Objects’ from a given scene

Here you will pretty much redo the same thing from previous sections, only difference being the set of table top images now have a collection of objects. And you need to extract just the objects without any irrelevant data from the scene.

<a name='segment'></a>

### 3. Segment the scene using known Objects

Now that you have reconstructed the 3D point cloud of the scene you need to segment each individual object of the scene by finding a suitable match in the set of 3D objects that you have already constructed. And while segmenting it you need to color code each object in the scene to the label of that object (feel free to use random color codes for each object such that each object is a different color). Also, note that you cannot assume that you know the number of objects in the scene, this has to be computed by your algorithm.

<a name='semantic'></a>

### 4. Build a Semantic Map

Now try to see if you can derive relationships between the segmented objects, like this object is at this angle with respect to another object or this object is so much smaller than another object or this object is on top of another object and so on. Think of this as modelling information how a human would think of this scene.

<a name='stuff'></a>

### 5. Stuff given to you

Calibration parameters and uncalibrated RGB-D data. Note that you are allowed to use third party code for basic stuff but not for the whole algorithm. You are only allowed to use `pcread`, `pcshow` from the Matlab’s point cloud library. Functions like `KDTreeSearcher` and `knnsearch` can be used for point to point correspondence search. You are also allowed to use any other basic functions built into the computer vision or image processing toolbox in Matlab.

Make sure to check Reference Papers folders for papers regarding ICP (Standard ICP paper is the easiest but the worst performing) and Object Segmentation. Dataset folder contains the data of various scenes containing individual objects and multiple objects.

<a name='guidelines'></a>

### 6. Submission Guidelines

Make a video of all the objects and all the segmented scenes where you pan the 3D point cloud to show it is fully reconstructed and is correctly segmented for each object. You MUST submit a report written in IEEE double column format in L A TEXand should not exceed 6 pages (Template given in Draft folder). The report should be of a conference paper quality. Feel free to add videos you feel are cool or paste YouTube links in your report. You should also include a detailed README file explaining how to run your code.

Submit your .fig files (of 3D point clouds for all individual objects), codes (.m files) with the naming convention YourDirectoryID P5.zip onto ELMS/Canvas (Please compress it to .zip and no other format). If your e-mail ID is `ABCD[at]terpmail.umd.edu` or `ABCD[at]umd.edu` your Directory ID will be `ABCD`.

To summarize, you need to submit these things and in the following strcture: A zip file with the name `YourDirectoryID_P4.zip` onto ELMS/Canvas. A main folder with the name `YourDirectoryID_P4` with the following things (for EACH of the given scene)):

- A video showing a full 3D reconstruction for each object.
- A video showing a full 3D reconstruction and segmentation of each scene.
- <b>`.fig`</b> files for 3D models of all individual objects and scenes (color coded with segmentation labels)
- Code used for this project with a detailed README file.
- A conference paper quality report written in IEEE double column format in LATEX format (Check <b>`Draft`</b> folder for necessary template and class files)


<b>If your submission does not comply with the above guidelines, you’ll be given ZERO credit.</b>

<b>EACH TEAM WILL HAVE TO MAKE A PRESENTATION VIDEO OF ABOUT 7-10MINS AND SUBMIT a `MP4` FORMAT FILE THAT EXPLAINS THE PROBLEMS YOU FACED AND THE ROADBLOCKS YOU OVERCAME OR DIDN'T.</b>

### Collaboration Policy

You can discuss the ideas with any number of people. But the code you turn-in should be from your own team and you SHOULD NOT USE codes from other students. For other honor code refer to the CMSC733 Spring 2019 website.


<b> BUT MOST IMPORTANTLY, DO NOT FORGET TO HAVE FUN AND PLAY AROUND WITH IMAGES! </b> 



### Acknowledgements

We would like to thank [Aleksandrs Ecins](http://users.umiacs.umd.edu/~aecins/) for the dataset. This fun project was inspired from Nitin’s research and a course project at University of Pennsylvania, [GRASP ARCHE](http://www.grasparche.com/).

---
