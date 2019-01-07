---
layout: page
mathjax: true
title: FaceSwap
permalink: /2019/proj/p2/
---

Table of Contents:
- [Deadline](#due)
- [Introduction](#intro)
- [Problem Statement](#prob)
- [Data Collection](#data)
- [Phase 1: Traditional Approach](#ph1)
    - [Facial Landmarks detection](#landmarks)
    - [Face Warping using Triangulation](#tri)
    - [Face Warping using Thin Plate Spline](#tps)
    - [Replace Face](#replace)
    - [Blending](#blending)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Deadline 
11:59PM, Sunday, March 17, 2019

<a name='prob'></a>
## Problem Statement
The aim of this project is to implement an end-to-end pipeline to swap faces in a
video just like [Snapchat's face swap filter](https://www.snapchat.com/) or [this face swap website](
http://faceswaplive.com/). It's a fairly complicated procedure and variants of these have
been used in many movies. One of the most successful examples has been using these method
is replacing Paul Walker's brother's face by Paul Walkers in the Fast and Furious 7 movie
after the sudden death of Paul Walker in a car crash during shooting. And the ethical issue with this project is people creating fake videos of celibrities called Deep Fakes. Now that I have conviced you that this is not just for fun but is useful too, In the next few sections, let us
see how this can be done.


<a name='data'></a>
## Data Collection
Record two videos. One of yourself with just your face and the other with
your face and your friend's face in all the frames. Convert them to .avi or .mp4 format.
Save them to the Data folder. Feel free to play around with more videos. In the first video,
we'll replace your face with some celebrity's face or your favorite relative's face. In the
second video we'll swap the two faces. If there are more than two faces in the video, swap
the two largest faces.

<a name='ph1'></a>
## Phase 1: Traditional Approach
<!---
    Dlib tutorial for facial landmarks detection
https://www.pyimagesearch.com/2017/04/03/facial-landmarks-dlib-opencv-python/)
-->
<a name='landmarks'></a>
### Facial Landmarks detection
The first step in the traditional approach is to find facial landmarks (important points on the face) so that we have one-to-one correspondence between the facial landmakrs. This is analogous to the detection of corners in the panorama project. One of the major reasons to use facial landmarks instead of using all the points on the face is to reduce computational complexity. Remember that better results can be obtained using all the points (dense flow) or using a meshgrid. For detecting facial landmarks we'll use dlib library built into OpenCV and python. A sample output of Dlib is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p2/dlib.png" width="80%">
  <div class="figcaption">
    Fig 1: Output of dlib for facial landmarks detection. Green landmarks are overlayed on the input image.
  </div>
</div>

<a name='tri'></a>
### Face Warping using Triangulation



## Acknowledgements
This fun project was inspired by a similar project in UPenn's <a href="https://alliance.seas.upenn.edu/~cis581/wiki/index.php?title=CIS_581:_Computer_Vision_%26_Computational_Photography">CIS581</a> (Computer Vision & Computational Photography). 

