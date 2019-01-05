---
layout: page
mathjax: true
title: Learning the basics of Computer Vision
permalink: /pano-prereq/
---


**This article is written by <a href="http://chahatdeep.github.io/">Chahat Deep Singh.</a>** (chahat[at]terpmail.umd.edu)

Please contact **Chahat** if there are any errors.

Table of Contents:

- [Introduction](#intro)
- [Features and Convolution](#feat-conv)
	- [Features](#feat)
	- [Convolution](#conv)
	- [Deconvolution](#deconv)
	- [Different Operators](#diff-operators)
	- [Corner Detection](#corner-detection)
<!-- - [Optical Flow](#flow) (Coming Soon) -->


<a name='intro'></a>
## 1. Introduction

We have seen fascinating things that our camera applications like instagram, snachat or your default phone app do. It can be creating a full structure of your face for facial recognition or simply creating a panorama from multiple images. In this course, we will learn how to recreate such applications but before that we require to learn about the basics like filtering, features, camera models and transformations. This article is all about laying down the groundwork for the aforementioned applications. Let's start with understanding the most basic aspects of an image: features.

<a name='feat-conv'></a>
## 2. Features and Convolution
<a name='feature'></a>
### 2.1 What are features?
What are <i>features</i> in Machine vision? <i> Is it similar as Human visual perception?</i> This question can have different answers but one thing is certain that feature detection is an imperative building block for a variety of computer vision applications. We all have seen <i> Panorama Stitching </i> in our smartphones or other softwares like Adobe Photoshop or AutoPano Pro. The fundamental idea in such softwares is to align two or more images before seamlessly stitching into a panorama image. Now back to the question, <i> what kind of features should be detected before the alignment? </i> Can you think of a few types of features?

Certain locations in the images like corners of a building or mountain peaks can be considered as features. These kinds of localized features are known as <i>corners</i> or keypoints or interest points and are widely used in different applications. These are characterized by the appearance of neigborhood pixels surrounding the point (or local patches). Fig. 1 demonstrates strong and weak corner features.
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/strong-corners.png" width="49%">
  <div class="figcaption">Fig. 1: The section in <b><font color="red">red</font></b> illustrates good or strong corner features while the <b>black</b> depicts weak or bad features.</div>
</div>

The other kind of feature is based on the orientation and local appearance and is generally a good indicator of object boundaries and occlusion events. <i> (Occlusion means that there is something in the field of view of the camera but due to some sensor/optical property or some other scenario, you can't.)</i> There are multiple ways to detect certain features. One of the way is <b>convolution</b>. 

<a name='conv'></a>
### 2.2 Convolution
<i>Ever heard of Convolutional Neural Networks (CNN)? What is convolution? Is it really that 'convoluted'? </i> Let's try to answer such questions but before that let's understand what convolution really means! Think of it as an operation of changing the pixel values to a new set of values based on the values of the nearby pixels. <i> Didn't get the gist of it? Don't worry! </i>

Convolution is an operation between two functions, resulting in another function that depicts how the shape of first function is modified by the second function. The convolution of two functions, say $$f$$ and $$g$$ is written is $$f\star g$$ or $$f*g$$ and is defined as:

$$
(f*g)(t) = \int_{-\infty}^{\infty} f(\tau)g(t-\tau)d\tau = \int_{-\infty}^{\infty} f(t-\tau)g(\tau)d\tau
$$

Let's try to visualize convolution in one dimension. The following <i>figure</i> depcits the convolution <i>(in black)</i> of the two functions <i>(blue and red).</i> One can think convolution as the common area under the functions  $$f$$ and $$g$$.
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/conv.gif" width="49%">
  <div class="figcaption">Convolution of \(f\) and \(g\) is given by the area under the black curve.</div>
</div>


Since we would be dealing with discrete functions in this course (as images are of the size $$M\times N$$), let us look at a simple discrete 1D example:
$$f = [10, 50, 60, 10, 20, 40, 30]$$ and 
$$g = [1/3, 1/3, 1/3]$$.
Let the output be denoted by $$h$$. What would be the value of $$h(3)$$? In order to compute this, we slide $$g$$ so that it is centered around $$f(3)$$ _i.e._
$$\begin{bmatrix}10 & 50 & 60 & 10 & 20 & 40 & 30\\0 & 1/3 & 1/3 & 1/3 & 0 & 0 & 0\end{bmatrix}$$.
We multiply the corresponding values of $$f$$ and $$g$$ and then add up the products _i.e._
$$h(3)=\dfrac{1}{3}50+\dfrac{1}{3}60\dfrac{1}{3}10=40$$
It can be inferred that the function $$g$$ (also known as kernel or the filter) is computing a windowed average of the image.

Similarly, one can compute 2D convolutions (and hence any $$N$$ dimensions convolution) as shown in image below:
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/Conv2D.png" width="49%">
  <div class="figcaption">Convolution of a Matrix with a kernal.</div>
</div>
These convolutions are the most commonly used operations for smoothing and sharpening tasks. Look at the example down below:
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/ConvImg.png" width="49%">
  <div class="figcaption">Smoothing/Blurring using Convolution.</div>
</div>
Different kernels (or convolution masks) can be used to perform different level of sharpness:

<div class="fig figcenter fighighlight">
  <img src="/assets/pano/DifferentKernel.png" width="49%">
  <div class="figcaption">(a). Convolution with an identity masks results the same image as the input image. (b). Sharpening the image. (c). Normalization (or box blur). (d). 3X3 Gaussian blur. (e). 5X5 Gaussian blur. (f). 5X5 unsharp mask, it is based on the gaussian blur. NOTE: The denominator outside all the matrices are used to normalize the operation.</div>
</div>

Apart from the smoothness operations, convolutions can be used to detect features such as edges as well. The figure given below shows 
how different kernel can be used to find the edges in an image using convolution.
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/ConvImgEdge.png" width="49%">
  <div class="figcaption">Detecting edges in an image with different kernels.</div>
</div>


A good explanation of convolution can also be found <a href="http://colah.github.io/posts/2014-07-Understanding-Convolutions/">here</a>.


<a name='deconv'></a>
### Deconvolution:

Clearly as the name suggests, deconvolution is simply a process that reverses the effects of convolution on the given information. Deconvolution is implemented (generally) by computing the _Fourier Transform_ of the signal $$h$$ and the transfer function $$g$$ (where $$h=f * g$$). In frequency domain, (assuming no noise) we can say that: $$F=H/G$$. Fourier Transformation: [Optional Read](https://betterexplained.com/articles/an-interactive-guide-to-the-fourier-transform/), [Video](https://www.youtube.com/watch?v=spUNpyF58BY/)


One can perform deblurring and restoration tasks using deconvolution as shown in figure:
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/deconv.jpg" width="49%">
  <div class="figcaption">Left half of the image represents the input image. Right half represents the image after deconvolution.</div>
</div>


Now, since we have learned the fundamentals about convolution and deconvolution, let's dig deep into _Kernels_ or _point operators_). One can apply small convolution filters of size $$2\times2$$ or $$3\times3$$ or so on. These can be _Sobel, Roberts, Prewitt, Laplacian_ operators etc. We'll learn about them in a while. Operators like these are a good approximation of the derivates in an image. While for a better texture analysis in an image, larger masks like _Gabor_ filters are used.
But what does it mean to take the derivative of an image? The derivative or the gradient of an image is defined as: 
$$\nabla f=\left[\dfrac{\delta f}{\delta x}, \dfrac{\delta f}{\delta y}\right]$$
It is important to note that the gradient of an image points towards the direction in which the intensity changes at the highest rate and thus the direction is given by:
$$\theta=tan^{-1}\left(\dfrac{\delta f}{\delta y}\Bigg{/}\dfrac{\delta f}{\delta x}\right)$$
Moreover, the gradient direction is always perpendicular to the edge and the edge strength can by given by:
$$||\nabla f|| = \sqrt{\left(\dfrac{\delta f}{\delta x}\right)^2 + \left(\dfrac{\delta f}{\delta y}\right)^2}$$. In practice, the partial derivatives can be written (in discrete form) as the different of values between consecutive pixels _i.e._ $$f(x+1,y) -  f(x,y)$$ as $$\delta x$$  and  $$f(x,y+1) -  f(x,y)$$ as $$\delta y$$.

Figure below shows commonly used gradient operators. 
<div class="fig figcenter fighighlight">
  <img src="/assets/pano/gradoperators.png" width="49%">
  <div class="figcaption">Commonly used gradient operators.</div>
</div>

You can implement the following in MATLAB using the function `edge` with various methods for edge detection. 
Before reading any further, try the following in MATLAB:
-  `C = conv2(A, B)` <b>[Refer](https://www.mathworks.com/help/matlab/ref/conv2.html) </b>
-  `BW = edge(I, method, threshold, sigma)` <b>[Refer](https://www.mathworks.com/help/images/ref/edge.html)</b>
-  `B = imfilter(A, h, 'conv')` <b>[Refer](https://www.mathworks.com/help/images/ref/imfilter.html)</b>

<a name='diff-operators'></a>
### Different Operators:

#### Sobel Operator:
This operator has two $$3\times 3$$ kernels that are convolved with the original image `I` in order to compute the approximations of the derivatives.
The horizontal and vertical are defined as follows:

$$G_x = \begin{bmatrix} +1 & 0 & -1 \\ +2 & 0 & -2 \\ +1 & 0 & -1 \end{bmatrix}$$

$$G_y = \begin{bmatrix} +1 & +2 & +1 \\ 0 & 0 & 0 \\ -1 & -2 & -1 \end{bmatrix}$$

The gradient magnitude can be given as: $$G=\sqrt{G^2_x + G^2_y}$$ and the gradient direction can be written as: $$\theta=a tan \Bigg(\cfrac{G_y}{G_x}\Bigg)$$. Figure below shows the input image and its output after convolving with Sobel operator.

<div class="fig figcenter fighighlight">
  <img src="/assets/pano/filt1.png" width="70%">
  <div class="figcaption">(a). Input Image. (b). Sobel Output.</div>
</div>

#### Prewitt:

$$G_x = \begin{bmatrix} +1 & 0 & -1 \\ +1 & 0 & -1 \\ +1 & 0 & -1 \end{bmatrix}$$

$$G_y = \begin{bmatrix} +1 & +1 & +1 \\ 0 & 0 & 0 \\ -1 & -1 & -1 \end{bmatrix}$$

#### Roberts:

$$G_x = \begin{bmatrix} +1 & 0 \\ 0 & -1 \end{bmatrix}$$

$$G_x = \begin{bmatrix} 0 & +1 \\ -1 & 0 \end{bmatrix}$$


#### Canny:
Unlike any other filters, we have studied above, canny goes a bit further. In Canny edge detection, before finding the intensities of the image, a gaussian filter is applied to smooth the image in order to remove the noise. Now, one the gradient intensity is computed on the imge, it uses _non-maximum suppression_ to suppress only the weaker edges in the image. Refer: [A computational approach to edge detection](https://ieeexplore.ieee.org/document/4767851/). `MATLAB` has an `approxcanny` function that finds the edges using an approximate version which provides faster execution time at the expense of less precise detection. Figure below illustrates different edge detectors.


<div class="fig figcenter fighighlight">
  <img src="/assets/pano/filt2.png" width="80%">
  <div class="figcaption">(a). Prewitt Output. (b). Roberts Output. (c). Canny Output (d) Laplacian of Gaussian (LoG). </div>
</div>


<a name='corner-detection'></a>
### Corner Detection:
Now that we have learned about different edge features, let's understand what corner features are! Corner detection is used to extract certain kinds of features and common in panorama stitching (Project 1), video tracking (project 3), object recognition, motion detection etc. 

_What is a corner?_ To put it simply, a corner is the intersection of two edges. One can also define corner as: if there exist a point in an image such that there are two defintive (and different) edge directions in a local neighborhood of that point, then it is considered as a corner feature. In computer vision, corners are commonly written as 'interest points' in literature. 

The paramount property of a corner detector is the ability to detect the same corner in multiple images under different translation, rotation, lighting etc. The simplest approach for corner detection in images is using correlation. (Correlation is similar in nature to convolution of two functions). Optional Read: [Correlation](http://www.ee.ic.ac.uk/hp/staff/dmb/courses/e1fourier/00800_correlation.pdf)

In MATLAB, try the following:
- `corner(I, 'detectHarrisFeatures')`: Harris Corners
- `corner(I, 'detectMinEigenFeatures')`: Shi-Tomasi Corners 
- `detectFASTFeatures(I)`: FAST: Features from Accelerated Segment Test

<div class="fig figcenter fighighlight">
  <img src="/assets/pano/harris.jpg" width="49%">
  <div class="figcaption">Harris Corner Detection.</div>
</div>

<!--
<a name='blob-detection'></a>
### Blob detection:
Another important element in computer vision is blob. _What is a blob?_ A blob is a group of connected pixels in an image that shares some common properties like intensity values. In other words, all the pixels in a blob can be considered to be similiar to each other. Convolution is the most common method for blob detection. 

- LoG (Laplacian of Gaussian):
One of the most common blob detectors uses LoG. Laplacian filters are derivative filters used to find areas of 
Let $$f(x,y)$$ be the input image. Let $$g(x,y,t)$$ be the Gaussian kernel:

$$g(x,y,t)=\cfrac{1}{2\pi t} e^{-\frac{x^2+y^2}{2t}}$$

where 

<div class="fig figcenter fighighlight">
  <img src="/assets/pano/blob1.jpg" width="49%">
  <img src="/assets/pano/blob.png" width="49%">
  <div class="figcaption">Harris Corner Detection.</div>
</div>



- DoG (Difference of Gaussian)


<a name='feat-desc'></a>
### Feature Descriptor:

- Feature Descriptor: SIFT (Scale selectivity (CIS 580) Multi-scale concepts), SURF and HOG

-->
