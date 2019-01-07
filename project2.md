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
  <img src="/assets/2019/p2/dlib.jpg" width="80%">
  <div class="figcaption">
    Fig 1: Output of dlib for facial landmarks detection. Green landmarks are overlayed on the input image.
  </div>
</div>

<a name='tri'></a>
### Face Warping using Triangulation
Like we discussed before, we have now obtained facial landmarks, but what do we do with them? We need to ideally warp the faces in 3D, however we don't have 3D information. Hence can we make some assumption about the 2D image to approximate 3D information of the face. One simple way is to triangulate using the facial landmarks as corners and then make the assumption that in each triangle the content is planar (forms a plane in 3D). Triangulating or forming a triangular mesh over the 2D image is simple but we want to trinagulate such that it's fast and has an "efficient" triangulation. One such method is obtained by drawing the dual of the Voronoi diagram, i.e., connecting each two neighboring sites in the Voronoi diagram. This is called the **Delaunay Triangulation** and can be constructed in \\(\mathcal{O}(n\log{}n)\\) time. We want the triangulation to be consistent with the image boundary such that texture regions won't fae into the background while warping. Delaunay Triangulation tries the maximize the smallest angle in each triangle.
 
<div class="fig fighighlight">
  <img src="/assets/2019/p2/DT.png" width="100%">
  <div class="figcaption">
    Fig 2: Triangulation on two faces we want to swap (a cat and a baby). 
  </div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p2/DTAsDualOfVoronoi.png" width="70%">
  <div class="figcaption">
    Fig 3: Delaunay Triangulation is the dual of the Voronoi diagram. Black lines show the Voinoi diagram and colored lines show the Delaunay triangulation.
  </div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p2/DTAsDualOfVoronoi.png" width="70%">
  <div class="figcaption">
    Fig 4: Comparison of good and bad triangulation depending on choice of landmarks. 
  </div>
</div>


Since, Delaunay Triangulation tries the maximize the smallest angle in each triangle, we will obtain the same triangulation in both the images, i.e., cat and baby's face. Hence, if we have correspondences between the facial landmarks we also have correspondences between the triangles (this is awesome! and makes life simple). Because we are using dlib to obtain the facial landmarks (or click points manually if you want to warp a cat to a kid), we have correspondences between facial landmarks and hence correspondences between the triangles, i.e., we have the same mesh in both images. Use the ``getTriangleList()`` function in ``cv2.Subdiv2D`` class of OpenCV to implement Delaunay Triangulation. Refer to [this tutorial](https://www.learnopencv.com/delaunay-triangulation-and-voronoi-diagram-using-opencv-c-python/) for an easy start. Now, we need to warp the destination face to the source face (we are using inverse warping so that we don't have any holes in the image, read up why inverse warping is better than forward warping) or to a mean face. Implement the following steps to warp one face (\\(\mathcal{A}\\) or source) to another (\\(\mathcal{B}\\) or destination). 


## Acknowledgements
This fun project was inspired by a similar project in UPenn's <a href="https://alliance.seas.upenn.edu/~cis581/wiki/index.php?title=CIS_581:_Computer_Vision_%26_Computational_Photography">CIS581</a> (Computer Vision & Computational Photography). 

