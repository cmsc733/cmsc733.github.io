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
    - [Motion Filtering](#motfilt)
- [Phase 2: Deep Learning Approach](#ph2)
- [Notes about Test Set](#testset)
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
The overview of the system for face replacement is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p2/Overview.png" width="100%">
  <div class="figcaption">
    Fig 1: Overview of the face replacement pipeline.
  </div>
</div>


<a name='landmarks'></a>
### Facial Landmarks detection
The first step in the traditional approach is to find facial landmarks (important points on the face) so that we have one-to-one correspondence between the facial landmakrs. This is analogous to the detection of corners in the panorama project. One of the major reasons to use facial landmarks instead of using all the points on the face is to reduce computational complexity. Remember that better results can be obtained using all the points (dense flow) or using a meshgrid. For detecting facial landmarks we'll use dlib library built into OpenCV and python. A sample output of Dlib is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p2/dlib.jpg" width="80%">
  <div class="figcaption">
    Fig 2: Output of dlib for facial landmarks detection. Green landmarks are overlayed on the input image.
  </div>
</div>

<a name='tri'></a>
### Face Warping using Triangulation
Like we discussed before, we have now obtained facial landmarks, but what do we do with them? We need to ideally warp the faces in 3D, however we don't have 3D information. Hence can we make some assumption about the 2D image to approximate 3D information of the face. One simple way is to triangulate using the facial landmarks as corners and then make the assumption that in each triangle the content is planar (forms a plane in 3D) and hence the warping between the the triangles in two images is affine. Triangulating or forming a triangular mesh over the 2D image is simple but we want to trinagulate such that it's fast and has an "efficient" triangulation. One such method is obtained by drawing the dual of the Voronoi diagram, i.e., connecting each two neighboring sites in the Voronoi diagram. This is called the **Delaunay Triangulation** and can be constructed in \\(\mathcal{O}(n\log{}n)\\) time. We want the triangulation to be consistent with the image boundary such that texture regions won't fae into the background while warping. Delaunay Triangulation tries the maximize the smallest angle in each triangle.
 
<div class="fig fighighlight">
  <img src="/assets/2019/p2/DT.PNG" width="100%">
  <div class="figcaption">
    Fig 3: Triangulation on two faces we want to swap (a cat and a baby). 
  </div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p2/DTAsDualOfVoronoi.PNG" width="70%">
  <div class="figcaption">
    Fig 4: Delaunay Triangulation is the dual of the Voronoi diagram. Black lines show the Voinoi diagram and colored lines show the Delaunay triangulation.
  </div>
</div>

<div class="fig fighighlight">
  <img src="/assets/2019/p2/GoodAndBadTriangulation.PNG" width="100%">
  <div class="figcaption">
    Fig 5: Comparison of good and bad triangulation depending on choice of landmarks. 
  </div>
</div>


Since, Delaunay Triangulation tries the maximize the smallest angle in each triangle, we will obtain the same triangulation in both the images, i.e., cat and baby's face. Hence, if we have correspondences between the facial landmarks we also have correspondences between the triangles (this is awesome! and makes life simple). Because we are using dlib to obtain the facial landmarks (or click points manually if you want to warp a cat to a kid), we have correspondences between facial landmarks and hence correspondences between the triangles, i.e., we have the same mesh in both images. Use the ``getTriangleList()`` function in ``cv2.Subdiv2D`` class of OpenCV to implement Delaunay Triangulation. Refer to [this tutorial](https://www.learnopencv.com/delaunay-triangulation-and-voronoi-diagram-using-opencv-c-python/) for an easy start. Now, we need to warp the destination face to the source face (we are using inverse warping so that we don't have any holes in the image, read up why inverse warping is better than forward warping) or to a mean face (obtained by averaging the triangulations of two faces). Implement the following steps to warp one face ($$\mathcal{A}$$ or source) to another ($$\mathcal{B}$$ or destination). 

1. For each triangle in the target/destination face $$\mathcal{B}$$, compute the Barycentric coordinate. 

$$ \begin{bmatrix}
 \mathcal{B}_{a,x} & \mathcal{B}_{b,x} & \mathcal{B}_{c,x}\\
 \mathcal{B}_{a,y} & \mathcal{B}_{b,y} & \mathcal{B}_{c,y}\\
 1 & 1 & 1\\
 \end{bmatrix} \begin{bmatrix} \alpha \\ \beta \\ \gamma \\ \end{bmatrix} = \begin{bmatrix} x \\ y \\ 1\\ \end{bmatrix} $$

Here, the Barycentric coordinate is given by $$ \begin{bmatrix} \alpha & \beta & \gamma \end{bmatrix}^T $$. Note that, the matrix on the left hand size and it's inverse need to be computed only once per triangle. In this matrix, $$ a, b, c $$ represent the corners of the triangle and $$x,y$$ represent the $$x$$ and $$y$$ coordinates of the particular triangle corner respectively. 

Now, given the values of the matrix on the left hand size we will call $$ \mathcal{B}_{\Delta} $$ and the value of $$ \begin{bmatrix} x & y & 1 \end{bmatrix}^T $$ we can compute the value of $$\begin{bmatrix} \alpha & \beta & \gamma \end{bmatrix}^T $$ as follows:

$$
 \begin{bmatrix} \alpha \\ \beta \\ \gamma \\ \end{bmatrix} = \mathcal{B}_{\Delta}^{-1} \begin{bmatrix} x \\ y \\ 1\\ \end{bmatrix}
$$
Now, given the values of $$ \alpha, \beta, \gamma$$ we can say that a point $$x$$ lies inside the triangle if $$ \alpha \in [0, 1] $$, $$ \beta \in [0, 1] $$ and $$\alpha + \beta + \gamma \in [0,1]$$. **DO NOT USE any built-in function for this part**.

2. Compute the corresponding pixel position in the source image $$\mathcal{A}$$ using the barycentric equation shown in the last step but with a different triangle coordinates. This is computed as follows:

$$
 \begin{bmatrix} x_{\mathcal{A}} \\ y_{\mathcal{A}} \\ z_{\mathcal{A}} \\ \end{bmatrix} = \mathcal{A}_{\Delta} \begin{bmatrix} \alpha \\ \beta \\ \gamma\\ \end{bmatrix}
$$

Here, $$ \mathcal{A}_{\Delta} $$ is given as follows:

$$
\mathcal{A}_{\Delta} = \begin{bmatrix}
 \mathcal{A}_{a,x} & \mathcal{A}_{b,x} & \mathcal{A}_{c,x}\\
 \mathcal{A}_{a,y} & \mathcal{A}_{b,y} & \mathcal{A}_{c,y}\\
 1 & 1 & 1\\
 \end{bmatrix}
$$

Note that, after we obtain  $$\begin{bmatrix} x_{\mathcal{A}} & y_{\mathcal{A}} & z_{\mathcal{A}} \end{bmatrix}^T$$, we need to convert the values to homogeneous coordinates as follows:

$$
x_{\mathcal{A}} = \frac{x_{\mathcal{A}}}{z_{\mathcal{A}}} \text{ and } y_{\mathcal{A}} = \frac{y_{\mathcal{A}}}{z_{\mathcal{A}}}
$$

3. Now, copy back the value of the pixel at $$ (x_{\mathcal{A}}, y_{\mathcal{A}} ) $$ to the target location. Use ``scipy.interpolate.interp2d`` to perform this operation.

The warped images are shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p2/WarpOutput.png" width="70%">
  <div class="figcaption">
    Fig 6: Top row: Original images, Bottom row (left to right): Cat warped to baby and baby warped to cat. 
  </div>
</div>

<a name='tps'></a>
### Face Warping using Thin Plate Spline
As we discussed before, triangulation assumes that we are doing affine transformation on each triangle. This might not be the best way to do warping since the human face has a very complex and smooth shape. A better way to do the transformation is by using Thin Plate Splines (TPS) which can model arbitrarily complex shapes. Now, we want to compute a TPS that maps from the feature points in $$\mathcal{B}$$ to the corresponding feature
points in $$\mathcal{A}$$ . Note that we need two splines, one for the $$x$$ coordinate and one for the $$y$$. A thin
plate spline has the following form:

$$
f(x,y) = a_1 + (a_x)x + (a_y)y + \sum_{i=1}^p{w_i U\left( \vert \vert (x_i,y_i) - (x,y)\vert \vert\right)}
$$

Here, $$ U(r) = r^2\log (r^2 )$$.

Note that, again in this case we are performing inverse warping, i.e., finding parameters of a Thin Plate Spline which will map from $$\mathcal{B}$$ to $$ \mathcal{A}$$. Warping using a TPS is performed in two steps. Let's look at the steps below.

1. In the first step, we will estimate the parameters of the TPS. The solution of the TPS model requires solving the following equation:

$$
 \begin{bmatrix} K & P\\ P^T & 0\\ \end{bmatrix} 
  \begin{bmatrix} w_1 \\ w_2 \\ \vdots \\ w_p \\ a_x \\ a_y \\ a_1  \end{bmatrix}  =
  \begin{bmatrix} v_1 \\ v_2 \\ \vdots \\ v_p \\ 0 \\ 0 \\ 0 \end{bmatrix}  
$$

where \\( K_{ij} = U\left( \vert \vert (x_i,y_i)-(x_j,y_j) \vert \vert \right)\\). $$v_i = f(x_i,y_i)$$ and the i<sup>th</sup> row of $$P$$ is $$(x_i, y_i, 1)$$. $$K$$ is a matrix of size size $$p \times p$$, and $$P$$ is a matrix of size $$p \times 3$$. In order to have a stable solution you need to compute the solution by:

$$ 
 \begin{bmatrix} w_1 \\ w_2 \\ \vdots \\ w_p \\ a_x \\ a_y \\ a_1  \end{bmatrix}  = 
  \left(\begin{bmatrix} K & P\\ P^T & 0\\ \end{bmatrix}  + \lambda I(p+3, p+3)\right)^{-1}
 \begin{bmatrix} v_1 \\ v_2 \\ \vdots \\ v_p \\ 0 \\ 0 \\ 0 \end{bmatrix} 
 $$

where $$I(p+3,p+3)$$ is a $$p+3 \times p+3$$ identity matrix. $$\lambda \ge 0$$ but is generally very close to zero. Think about why we need this. Note that you need to do this step twice, once for $$x$$ co-ordinates and once for $$y$$ co-ordinates. 

2. In the second step, use the estimated parameters of the TPS models (both $$x$$ and $$y$$ directions), transform all pixels in image $$\mathcal{B}$$ by the TPS model. Now, read back the pixel value from image $$\mathcal{A}$$ directly. The position of the pixels in image $$\mathcal{A}$$ is generated by the TPS equation (first equation in this section).

**Note that, both warping methods solve the same problem but with different formulations, you are required to implement both and compare the results.**


<a name='replace'></a>
### Replace Face
This part is very simple, you have to take all the pixels from face $$\mathcal{A}$$, warp them to fit face $$\mathcal{B}$$ and replace the pixels. Note that simply replacing pixels will not look natural as the lighing and edges look different. A sample output of face replacement is shown below.


<div class="fig fighighlight">
  <img src="/assets/2019/p2/FaceReplacement.png" width="70%">
  <div class="figcaption">
    Fig 7: Output of sample face replacement. Notice the difference in color and edges. The output is not a seamless blend. 
  </div>
</div>

<a name='blending'></a>
### Blending
We will follow a method called Poisson Blending to blend the warped face onto the target face. More details about this method can be found in [this paper](http://www.irisa.fr/vista/Papers/2003_siggraph_perez.pdf). Note that, you **DO NOT** have to implement this part from scratch, feel free to use any open-source implementation and cite your source in your report and your code. Your task in this part is to blend the face as seamlessly as possible. Feel free to reuse concepts you learnt from panorama stitching project's last part here. A good blending output is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/p2/FaceSwap.png" width="70%">
  <div class="figcaption">
    Fig 8: Output of sample face replacement after blending.
  </div>
</div>

<a name='motfilt'></a>
### Motion Filtering
After you have detected, warped and blended the face your algorithm works really well for individual frames. But when you want to do this for a video, you'll see flickering. Come up with your own solution to reduce the amount of flickering. You can use a low-pass fillter or a fancy Kalman Filter to do this. Feel free to use any third party or built-in code to do this. If you use third party code, please do not forget to cite them. Look at this holy grail video of face replacement where Jimmy Fallon interviews his cousin.

<iframe src="https://player.vimeo.com/video/257360045" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/257360045">Jimmy Fallon interview his twin!</a> from <a href="https://vimeo.com/user16478660">ZeroCool22</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

<a name='ph2'></a>
## Phase 2: Deep Learning Approach
For this phase, we'll run an off-the-shelf model to obtain face fiducials using deep learning. We think that implementing this part is fairly trivial and is left as a fun excericse if you want some programming practice (you are not graded on this implementation). We'll use the code from [this paper](https://arxiv.org/abs/1803.07835), which implements a supervised encoder-decoder model to obtain the full 3D mesh of the face. We recommend you to read the paper for more details. The code from the paper can be found [here](https://github.com/YadiraF/PRNet). Your task is to setup the code and run to obtain face fiducials/full 3D mesh. Use this output to perform face replacement as before. Feel free to use as much code as you want from last part/phase. Present a detailed comparison of both the traditional methods (triangulation and TPS) along with the deep learning method.   

<a name='testset'></a>
## Notes about Test Set
One day (24 hours) before the deadline, a test set will be released with details of what faces to replace. We'll grade on the completion of the project and visually appealing results.

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



## Acknowledgements
This fun project was inspired by a similar project in UPenn's <a href="https://alliance.seas.upenn.edu/~cis581/wiki/index.php?title=CIS_581:_Computer_Vision_%26_Computational_Photography">CIS581</a> (Computer Vision & Computational Photography). 

