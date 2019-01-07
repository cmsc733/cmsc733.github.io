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
Like we discussed before, we have now obtained facial landmarks, but what do we do with them? We need to ideally warp the faces in 3D, however we don't have 3D information. Hence can we make some assumption about the 2D image to approximate 3D information of the face. One simple way is to triangulate using the facial landmarks as corners and then make the assumption that in each triangle the content is planar (forms a plane in 3D) and hence the warping between the the triangles in two images is affine. Triangulating or forming a triangular mesh over the 2D image is simple but we want to trinagulate such that it's fast and has an "efficient" triangulation. One such method is obtained by drawing the dual of the Voronoi diagram, i.e., connecting each two neighboring sites in the Voronoi diagram. This is called the **Delaunay Triangulation** and can be constructed in \\(\mathcal{O}(n\log{}n)\\) time. We want the triangulation to be consistent with the image boundary such that texture regions won't fae into the background while warping. Delaunay Triangulation tries the maximize the smallest angle in each triangle.
 
<div class="fig fighighlight">
  <img src="/assets/2019/p2/DT.PNG" width="100%">
  <div class="figcaption">
    Fig 2: Triangulation on two faces we want to swap (a cat and a baby). 
  </div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p2/DTAsDualOfVoronoi.PNG" width="70%">
  <div class="figcaption">
    Fig 3: Delaunay Triangulation is the dual of the Voronoi diagram. Black lines show the Voinoi diagram and colored lines show the Delaunay triangulation.
  </div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p2/GoodAndBadTriangulation.PNG" width="100%">
  <div class="figcaption">
    Fig 4: Comparison of good and bad triangulation depending on choice of landmarks. 
  </div>
</div>


Since, Delaunay Triangulation tries the maximize the smallest angle in each triangle, we will obtain the same triangulation in both the images, i.e., cat and baby's face. Hence, if we have correspondences between the facial landmarks we also have correspondences between the triangles (this is awesome! and makes life simple). Because we are using dlib to obtain the facial landmarks (or click points manually if you want to warp a cat to a kid), we have correspondences between facial landmarks and hence correspondences between the triangles, i.e., we have the same mesh in both images. Use the ``getTriangleList()`` function in ``cv2.Subdiv2D`` class of OpenCV to implement Delaunay Triangulation. Refer to [this tutorial](https://www.learnopencv.com/delaunay-triangulation-and-voronoi-diagram-using-opencv-c-python/) for an easy start. Now, we need to warp the destination face to the source face (we are using inverse warping so that we don't have any holes in the image, read up why inverse warping is better than forward warping) or to a mean face (obtained by averaging the triangulations of two faces). Implement the following steps to warp one face (\\(\mathcal{A}\\) or source) to another (\\(\mathcal{B}\\) or destination). 

1. For each triangle in the target/destination face \\(\mathcal{B}\\), compute the Barycentric coordinate. 

$$ \begin{bmatrix}
 \mathcal{B}_{a,x} & \mathcal{B}_{b,x} & \mathcal{B}_{c,x}\\
 \mathcal{B}_{a,y} & \mathcal{B}_{b,y} & \mathcal{B}_{c,y}\\
 1 & 1 & 1\\
 \end{bmatrix} \begin{bmatrix} \alpha \\ \beta \\ \gamma \\ \end{bmatrix} = \begin{bmatrix} x \\ y \\ 1\\ \end{bmatrix} $$

Here, the Barycentric coordinate is given by \( \begin{bmatrix} \alpha & \beta & \gamma \end{bmatrix}^T \). Note that, the matrix on the left hand size and it's inverse need to be computed only once per triangle. In this matrix, \( a, b, c \) represent the corners of the triangle and \(x,y\) represent the \(x\) and \(y\) coordinates of the particular triangle corner respectively. 

Now, given the values of the matrix on the left hand size we will call \( \mathcal{B}_{\Delta} \) and the value of \( \begin{bmatrix} x & y & 1 \end{bmatrix}^T \) we can compute the value of \( \begin{bmatrix} \alpha & \beta & \gamma \end{bmatrix}^T \) as follows:

$$
 \begin{bmatrix} \alpha \\ \beta \\ \gamma \\ \end{bmatrix} = \mathcal{B}_{\Delta}^{-1} \begin{bmatrix} x \\ y \\ 1\\ \end{bmatrix}
$$
Now, given the values of \( \alpha, \beta, \gamma\) we can say that a point \(x\) lies inside the triangle if \( \alpha \in [0, 1] \), \( \beta \in [0, 1] \) and \(\alpha + \beta + \gamma \in [0,1]\). **DO NOT USE any built-in function for this part**.

2. Compute the corresponding pixel position in the source image \(\mathcal{A}\) using the barycentric equation shown in the last step but with a different triangle coordinates. This is computed as follows:

$$
 \begin{bmatrix} x_{\mathcal{A}} \\ y_{\mathcal{A}} \\ z_{\mathcal{A}} \\ \end{bmatrix} = \mathcal{A}_{\Delta} \begin{bmatrix} \alpha \\ \beta \\ \gamma\\ \end{bmatrix}
$$

Here, \( \mathcal{A}_{\Delta} \) is given as follows:

$$
\mathcal{A}_{\Delta} = \begin{bmatrix}
 \mathcal{A}_{a,x} & \mathcal{A}_{b,x} & \mathcal{A}_{c,x}\\
 \mathcal{A}_{a,y} & \mathcal{A}_{b,y} & \mathcal{A}_{c,y}\\
 1 & 1 & 1\\
 \end{bmatrix}
$$

Note that, after we obtain  \(\begin{bmatrix} x_{\mathcal{A}} & y_{\mathcal{A}} &z_{\mathcal{A}} \end{bmatrix}^T\), we need to convert the values to homogeneous coordinates as follows:

$$
x_{\mathcal{A}} = \frac{x_{\mathcal{A}}}{z_{\mathcal{A}}} \text{ and } y_{\mathcal{A}} = \frac{y_{\mathcal{A}}}{z_{\mathcal{A}}}
$$

3. Now, copy back the value of the pixel at \( (x_{\mathcal{A}}, y_{\mathcal{A}} ) \) to the target location. Use ``scipy.interpolate.interp2d`` to perform this operation.

The warped images are shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p2/WarpOutput.png" width="70%">
  <div class="figcaption">
    Fig 5: Top row: Original images, Bottom row (left to right): Cat warped to baby and baby warped to cat. 
  </div>
</div>

<a name='tps'></a>
### Face Warping using Thin Plate Spline
As we discussed before, triangulation assumes that we are doing affine transformation on each triangle. This might not be the best way to do warping since the human face has a very complex and smooth shape. A better way to do the transformation is by using Thin Plate Splines (TPS) which can model arbitrarily complex shapes. Now, we want to compute a TPS that maps from the feature points in \( \mathcal{B}\) to the corresponding feature
points in \( \mathcal{A}\) . Recall we need two splines, one for the \(x\) coordinate and one for the \(y\). A thin
plate spline has the following form:

$$
f(x,y) = a_1 + (a_x)x + (a_y)y + \sum_{i=1}^p{w_i U\left( \vert \vert (x_i,y_i) - (x,y)\vert \vert\right)}
$$

Here, \( U(r) = r^2\log (r^2 )\)

## Acknowledgements
This fun project was inspired by a similar project in UPenn's <a href="https://alliance.seas.upenn.edu/~cis581/wiki/index.php?title=CIS_581:_Computer_Vision_%26_Computational_Photography">CIS581</a> (Computer Vision & Computational Photography). 

