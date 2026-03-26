---
layout: page
mathjax: true
title: Scenes in Splats - A 3D Gaussian Splatting Approach
permalink: /2024/proj/p4/
---

This article is written by Naitri Rajyaguru.
If you have any questions/corrections regarding the article, please email at nrajyagu[at]umd[dot]edu.

Adapted from _16-825: Learning for 3D Vision_, Shubham Tulsiani, CMU ([learning3d.github.io](https://learning3d.github.io/)).

**To be submitted individually.**

Table of Contents:
- [1. Deadline](#due)
- [2. Introduction](#intro)
- [3. Background and Concepts](#background)
	- [3.1. The Problem: Novel View Synthesis](#nvs)
	- [3.2. Why Not Just Use a Point Cloud?](#pointcloud)
	- [3.3. The Core Idea: Scene as a Set of Gaussian Primitives](#coreidea)
	- [3.4. Anatomy of a Single 3D Gaussian](#anatomy)
	- [3.5. Covariance Parameterization: Rotation × Scale](#covariance)
	- [3.6. View-Dependent Color via Spherical Harmonics](#sh)
	- [3.7. Volume Rendering and Alpha Compositing](#volrender)
	- [3.8. The Full 3DGS Pipeline at a Glance](#pipeline)
- [4. Environment Setup](#setup)
- [5. 3D Gaussian Splatting](#gaussiansplatting)
	- [5.1. 3D Gaussian Rasterization (35 points)](#rasterization)
		- [5.1.1. Project 3D Gaussians to Obtain 2D Gaussians](#project)
		- [5.1.2. Evaluate 2D Gaussians](#evaluate)
		- [5.1.3. Filter and Sort Gaussians](#sort)
		- [5.1.4. Compute Alphas and Transmittance](#alpha)
		- [5.1.5. Perform Splatting](#splat)
	- [5.2. Training 3D Gaussian Representations (15 points)](#training)
		- [5.2.1. Setting Up Parameters and Optimizer](#optimizer)
		- [5.2.2. Perform Forward Pass and Compute Loss](#forwardpass)
- [6. Extra Credit](#extra)
	- [6.1. Compare Against Official 3DGS (+10 points)](#ec1)
	- [6.2. Run on a More Challenging Dataset (+10 points)](#ec2)
- [7. Notes about the Dataset](#dataset)
- [8. Submission Guidelines](#sub)
	- [8.1. File Tree and Naming](#files)
	- [8.2. Report](#report)
- [9. Collaboration Policy](#coll)

<a name='due'></a>
## 1. Deadline
**11:59PM, May 12, 2024.**

Starter code and data: [Download Project Codebase (Google Drive)](https://drive.google.com/file/d/1saZDsOn_7h37rer3N_KjsgsQYFHGS2Cr/view?usp=sharing)

<a name='intro'></a>
## 2. Introduction

In the previous project, you reconstructed a 3D scene and simultaneously obtained the camera poses of a monocular camera w.r.t. the given scene using Structure from Motion (SfM). The output of SfM is a _sparse point cloud_ along with calibrated camera poses — a skeletal outline of the scene. But this sparse representation cannot answer the question: _"what would this scene look like from a new, unphotographed viewpoint?"_

That question is the problem of **Novel View Synthesis (NVS)**, and it is one of the central open problems in computer vision and graphics. Now let's learn how to solve NVS using **3D Gaussian Splatting (3DGS)**, introduced by Kerbl et al. at SIGGRAPH 2023. The key idea: represent the scene as a collection of millions of colored, semi-transparent 3D ellipsoids called _Gaussian primitives_, then _rasterize_ them onto the image plane — the same way a game engine rasterizes triangles, but applied to fuzzy, transparent ellipsoids. Because the rasterizer is differentiable, the Gaussian parameters can be optimized end-to-end using only the posed photographs you already have from SfM. The result is photorealistic novel-view synthesis that renders at **≥ 30 fps at 1080p** (typically well over 100 fps in practice).

Here is an example of 3DGS rendering applied to a real-world scene:

<div class="fig fighighlight">
  <video width="100%" controls autoplay loop muted>
    <source src="https://drive.google.com/uc?export=download&id=1sES9hflNWKaCfSkhphWpWRik8CEZxe3J" type="video/mp4">
  </video>
  <div class="figcaption">
    Video 1: 3DGS rendering of the <em>bicycle</em> scene (Mip-NeRF 360 dataset). Each frame is rendered in real-time by alpha-compositing millions of 3D Gaussian primitives.
  </div>
  <div style="clear:both;"></div>
</div>

The bridge between SfM and 3DGS is direct: the sparse point cloud produced by SfM is used to _initialize_ the 3D Gaussian positions, and the calibrated camera poses provide the training supervision. You have already built the front-end of this pipeline. Now you will build the back-end.

There are a few steps that collectively form 3DGS:

- Represent the scene as a set of **3D Gaussian primitives** (mean, covariance, opacity, color)
- **Project** 3D Gaussians to 2D image-plane Gaussians using camera geometry
- **Sort by depth** and filter those outside the view frustum
- Compute **alpha and transmittance** per pixel
- Composite the final image via differentiable **alpha-blending** (splatting)
- **Backpropagate** gradients to optimize Gaussian parameters
- **Adaptively densify or prune** Gaussians during training

<a name='background'></a>
## 3. Background and Concepts

<a name='nvs'></a>
### 3.1. The Problem: Novel View Synthesis

Given a finite set of photographs of a scene taken from known viewpoints, _novel view synthesis_ asks: **synthesize a photorealistic image of the same scene from an arbitrary new viewpoint that was never photographed.**

<div class="fig fighighlight">
  <img src="/assets/2024/p4/nvs.png" width="100%">
  <div class="figcaption">
    Figure 1: Novel view synthesis. Given a set of posed input images (left), render any new viewpoint (right).
  </div>
  <div style="clear:both;"></div>
</div>

This is hard because:

- **Depth ambiguity.** A 2D photograph discards depth. You cannot trivially un-project a pixel into 3D without additional information.
- **View-dependent effects.** Surfaces change appearance with viewing angle: specular highlights shift, reflections change. A correct renderer must model this.
- **Real-time constraint.** For VR, gaming, and telepresence, rendering must be real-time (30–120 fps).

<a name='pointcloud'></a>
### 3.2. Why Not Just Use a Point Cloud?

The SfM output is a point cloud — why not just render those points? Three reasons:

1. **Sparsity.** SfM only places points where features matched. Large textureless regions have no points at all.
2. **No surface area.** A point is infinitely small. Rendering N points produces N isolated pixels, not a solid surface.
3. **No appearance model.** Each point stores an RGB value from a specific view. There is no model for how color should change with viewpoint.

What we need is a representation that: (a) fills in continuous regions of the scene, (b) has controllable size and orientation to model surfaces, and (c) has a differentiable appearance model. 3D Gaussians satisfy all three.

<a name='coreidea'></a>
### 3.3. The Core Idea: Scene as a Set of Gaussian Primitives

3DGS represents a scene as a set of **N 3D Gaussian primitives**, where each Gaussian is a fuzzy, colored, semi-transparent ellipsoid floating in 3D space. Think of each primitive as a soft blob of paint: a position, a shape (how elongated and how it is oriented), a color, and a transparency. Together, millions of these blobs tile the entire scene.

<div class="fig fighighlight">
  <img src="https://drive.google.com/uc?export=view&id=1b-WLPPY2e03pylWaeG0dm-gXh7sbco5E" width="100%">
  <div class="figcaption">
    Figure 2: The 3DGS pipeline. Starting from a sparse SfM point cloud, each point seeds one Gaussian ellipsoid. After optimization, the ellipsoids grow, flatten, and orient to model surface patches. The final rendered image is produced by alpha-compositing the 2D projections of all Gaussians from the current camera viewpoint.
  </div>
  <div style="clear:both;"></div>
</div>

To render a new view, we _project_ all N Gaussians onto the image plane (turning each 3D ellipsoid into a 2D "splat"), sort them by depth, and _alpha-composite_ them front-to-back. This process is called **splatting**.

<a name='anatomy'></a>
### 3.4. Anatomy of a Single 3D Gaussian

Each Gaussian primitive $$i$$ is described by four sets of learnable parameters:

**① Mean (Position) — $$\mu \in \mathbb{R}^3$$**

The center of the Gaussian in world space. Initialized from the SfM point cloud positions.

**② Covariance (Shape) — $$\Sigma \in \mathbb{R}^{3\times3}$$, positive semi-definite**

Controls the size and orientation of the ellipsoid. The 3D Gaussian function at world-space point $$\mathbf{x}$$ is:

$$G(\mathbf{x}) = \exp\!\left(-\tfrac{1}{2}(\mathbf{x}-\mu)^T \Sigma^{-1} (\mathbf{x}-\mu)\right)$$

**③ Opacity — $$o \in \mathbb{R}$$ (passed through sigmoid to get $$\alpha \in (0,1)$$)**

A scalar controlling how opaque the Gaussian is. Stored as a raw logit; squeezed to $$(0,1)$$ via sigmoid during rendering.

**④ Color / Spherical Harmonic Coefficients — $$f \in \mathbb{R}^k$$**

The color, potentially view-dependent (see §3.6). In this assignment: one RGB triplet per Gaussian.

| Parameter | Symbol | Dim. | Initialized from | Meaning |
|---|---|---|---|---|
| Mean | $$\mu$$ | $$\mathbb{R}^3$$ | SfM point position | Location of the ellipsoid |
| Covariance | $$\Sigma$$ | 3×3 PSD | Small isotropic sphere | Size and orientation |
| Opacity | $$o$$ | $$\mathbb{R}$$ → sigmoid | Near-zero (transparent) | How much light is blocked |
| SH coefficients | $$f$$ | $$\mathbb{R}^{3(d+1)^2}$$ | SfM point RGB color | View-dependent appearance |

<a name='covariance'></a>
### 3.5. Covariance Parameterization: Rotation × Scale

Why not optimize the 9 entries of $$\Sigma$$ directly? Because **a valid covariance matrix must be positive semi-definite (PSD)**. Unconstrained gradient descent violates this in two ways:

- **Loss of symmetry.** Gradient updates on raw 9 entries do not preserve $$\Sigma = \Sigma^T$$.
- **Indefiniteness.** Even with symmetry enforced, updates can drive eigenvalues negative. By the Spectral Theorem, a real symmetric matrix always has _real_ eigenvalues (imaginary eigenvalues cannot occur), so the actual danger is _negative real eigenvalues_ — making the matrix indefinite and its inverse ill-conditioned or undefined.

3DGS avoids this by decomposing:

$$\Sigma = R \, S \, S^T R^T$$

where **$$S = \text{diag}(s_1, s_2, s_3)$$** with $$s_i > 0$$ controls the squared radii along each principal axis (always ≥ 0), and **$$R \in SO(3)$$** (stored as a unit quaternion $$q$$) controls the orientation. The product $$RSS^TR^T$$ is always PSD for any valid $$R$$ and positive $$S$$.

_It is important to note that this is exactly the **eigendecomposition** of $$\Sigma$$: the columns of $$R$$ are the eigenvectors (principal axes of the ellipsoid) and $$s_i^2$$ are the eigenvalues (squared radii)._

> **Connection to Project 3:** In NonlinearPnP you used a unit quaternion to represent and optimize camera rotation — for exactly this reason: to enforce SO(3) without hard constraints. Here the same quaternion trick represents the orientation of each Gaussian ellipsoid.

<a name='sh'></a>
### 3.6. View-Dependent Color via Spherical Harmonics

Real surfaces look different depending on where you stand — specular highlights shift, reflections change. 3DGS encodes this view-dependence using **Spherical Harmonics (SH)** — orthonormal basis functions defined on the unit sphere, analogous to the Fourier series on a circle. Degree 0 gives a constant view-independent color. Higher degrees add directional variation.

The color of Gaussian $$i$$ viewed from unit direction $$\mathbf{d}$$ is:

$$c(\mathbf{d}) = \sum_{l=0}^{l_{\max}} \sum_{m=-l}^{l} f_{lm} \cdot Y_l^m(\mathbf{d})$$

In this assignment you will use only the **degree-0 (view-independent) component**: each Gaussian has a single RGB color that does not depend on the viewing direction. The original paper uses degree 3.

| SH Degree | # Basis Functions | Color Params | Models |
|---|---|---|---|
| 0 _(this assignment)_ | 1 | 3 | Constant color (no view-dependence) |
| 1 | 4 | 12 | Low-frequency directional variation |
| 2 | 9 | 27 | Diffuse + soft glossy effects |
| 3 _(original paper)_ | 16 | 48 | Specular highlights, mirror-like surfaces |

<a name='volrender'></a>
### 3.7. Volume Rendering and Alpha Compositing

Once we have projected all Gaussians to the image plane, we need to combine them to produce a final pixel color using **alpha compositing** — the same front-to-back blending used in 2D compositing software, but applied to projected 3D volumes.

**Intuition — Stacked Transparencies:** Imagine stacking N semi-transparent sheets of colored glass in front of a camera, ordered front-to-back. Each sheet adds its own color proportional to how opaque it is, and its contribution is attenuated by how much light has already been blocked by closer sheets.

Consider N Gaussians at pixel $$\mathbf{x}$$, sorted by depth $$d_1 < d_2 < \ldots < d_N$$. Define:

**Alpha $$\alpha(\mathbf{x}, i)$$** — opacity contribution of Gaussian $$i$$ at pixel $$\mathbf{x}$$:

$$\alpha(\mathbf{x}, i) = \sigma(o_i) \cdot \exp\!\bigl(P(\mathbf{x}, i)\bigr)$$

where $$\sigma(o_i) \in (0,1)$$ is the sigmoid of the learned opacity logit and $$P(\mathbf{x},i) \leq 0$$ is the Gaussian power (defined in §5.1.2), making $$\exp(P) \in (0,1]$$.

**Transmittance $$T(\mathbf{x}, i)$$** — fraction of light surviving past all Gaussians in front of $$i$$:

$$T(\mathbf{x}, i) = \prod_{j=1}^{i-1} \bigl(1 - \alpha(\mathbf{x}, j)\bigr)$$

When $$i=1$$ (frontmost), the product is empty so $$T(\mathbf{x},1) = 1$$. If any Gaussian $$j < i$$ is nearly opaque, $$T(\mathbf{x},i) \approx 0$$.

**Final pixel color:**

$$C(\mathbf{x}) = \sum_{i=1}^{N} c_i \cdot \alpha(\mathbf{x}, i) \cdot T(\mathbf{x}, i)$$

> **Note:** Depth sorting is mandatory. $$T(\mathbf{x},i)$$ is the product over all _closer_ Gaussians. Processing in the wrong order assigns incorrect transmittances and produces physically wrong occlusion.

> **Note — 3DGS uses zero neural networks.** Not at training, not at inference, not at any point. There is no MLP, no encoder, no decoder. The optimization is purely mathematical gradient descent on explicit float parameters (means, scales, quaternions, opacity logits, SH coefficients). This total absence of neural computation is precisely why rendering is real-time. It is sometimes said 3DGS "doesn't use a neural network at inference" — but this understates it. There is no neural network at any stage whatsoever.

<a name='pipeline'></a>
### 3.8. The Full 3DGS Pipeline at a Glance

<div class="fig fighighlight">
  <img src="https://drive.google.com/uc?export=view&id=1b-WLPPY2e03pylWaeG0dm-gXh7sbco5E" width="100%">
  <div class="figcaption">
    Figure 3: The complete 3DGS pipeline. The training loop renders the current Gaussians to produce a predicted image, computes a photometric loss against the ground-truth photograph, and backpropagates gradients through the differentiable rasterizer to update all Gaussian parameters. Every ~100 iterations, Adaptive Density Control clones, splits, or prunes Gaussians to improve scene coverage.
  </div>
  <div style="clear:both;"></div>
</div>

<a name='setup'></a>
## 4. Environment Setup

Before you begin, you must set up a conda environment with a CUDA-enabled PyTorch installation that is compatible with your GPU driver. Follow these steps **in order**:

**Step 1 — Verify your GPU driver and CUDA version**

```bash
nvidia-smi
```

Note the **CUDA Version** shown in the top-right corner of the output (e.g., `12.1`). This is the maximum CUDA version supported by your driver.

**Step 2 — Verify the CUDA compiler toolkit version**

```bash
nvcc -V
```

Note the CUDA compiler version (e.g., `release 11.8`). This must be ≤ the driver CUDA version from Step 1. If `nvcc` is not found, install the CUDA Toolkit matching your driver from [developer.nvidia.com/cuda-toolkit-archive](https://developer.nvidia.com/cuda-toolkit-archive).

**Step 3 — Create a conda environment**

```bash
conda create -n 3dgs python=3.9 -y
conda activate 3dgs
```

**Step 4 — Install PyTorch with CUDA support**

Install PyTorch using the CUDA version identified in Step 2. For example, for CUDA 11.8:

```bash
conda install pytorch==2.0.1 torchvision==0.15.2 torchaudio==2.0.2 pytorch-cuda=11.8 -c pytorch -c nvidia
```

For CUDA 12.1:

```bash
conda install pytorch==2.1.0 torchvision==0.16.0 torchaudio==2.1.0 pytorch-cuda=12.1 -c pytorch -c nvidia
```

Always install PyTorch _before_ pip packages. You can find the correct install command for your CUDA version at [pytorch.org/get-started/locally](https://pytorch.org/get-started/locally/).

**Step 5 — Verify your PyTorch CUDA installation**

```python
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

The output must show `True` for `cuda.is_available()`. **Do not proceed to Step 6 if CUDA is not available** — this means your PyTorch and CUDA versions are mismatched.

**Step 6 — Install assignment-specific packages**

Only after confirming that CUDA is available in PyTorch, install the remaining requirements:

```bash
pip install -r requirements.txt
```

**Notes:**

- Please run all code from the `Q1` folder for this assignment.
- Search for `### YOUR CODE HERE ###` for areas where code should be written.
- Please remember to follow the deliverables specified in the Submission section in each question.

<a name='gaussiansplatting'></a>
## 5. 3D Gaussian Splatting

In this part of the assignment, we will explore 3D Gaussian Splatting by building a simplified version of the 3D Gaussian rasterization pipeline introduced by the original paper. Once we create the rasterizer, we will first use it to render pre-trained 3D Gaussians. Then, we will create training code which leverages the renderer to optimize 3D Gaussians to represent custom scenes.

<a name='rasterization'></a>
### 5.1. 3D Gaussian Rasterization (35 points)

In this section, we will implement a 3D Gaussian rasterization pipeline in PyTorch. The official implementation uses custom CUDA code and several optimizations to make the rendering very fast. For simplicity, our implementation avoids many of the tricks and optimizations used by the official implementation and hence would be much slower. Additionally, instead of using all the spherical harmonic coefficients to model view dependent effects, we will only use the view independent components.

Inspite of these limitations, once you complete this section successfully, you will find that our simplified 3D Gaussian rasterizer can still produce renderings of pre-trained 3D Gaussians that were trained using the original codebase reasonably well!

For this section, you will have to complete the code in the files `model.py` and `render.py`. The file `model.py` contains code that manipulates and renders Gaussians. The file `render.py` uses functionality from `model.py` to render a pre-trained 3D Gaussian representation of an object.

In sections 5.1.1 to 5.1.4, you will have to complete the code in the classes `Gaussians` and `Scene` in the file `model.py`. In section 5.1.5, you will have to complete the code in the file `render.py`. It might be helpful to first skim both the files before starting the assignment to get a rough idea of the workflow.

**Note:** All the functions in `model.py` perform operations on a batch of N Gaussians instead of 1 Gaussian at a time. As such, it is recommended to write loopless vectorized code to maximize performance. However, a solution with loops will also be accepted as long as it works and produces the desired output.

<a name='project'></a>
#### 5.1.1. Project 3D Gaussians to Obtain 2D Gaussians

We will begin our implementation of a 3D Gaussian rasterizer by first creating functionality to project 3D Gaussians in the world space to 2D Gaussians that lie on the image plane of a camera.

A 3D Gaussian is parameterized by its mean (a 3 dimensional vector) and covariance (a 3×3 matrix). Following equations (5) and (6) of the original paper, we can obtain a 2D Gaussian (parameterized by a 2D mean vector and 2×2 covariance matrix) that represents an approximation of the projection of a 3D Gaussian to the image plane of a camera.

**⚠️ Camera Convention Warning — Read Before Coding**

Two opposite conventions exist, and confusing them is the most common implementation bug:

- **World-to-Camera (w2c):** transforms world → camera. Used by COLMAP, OpenCV, SfM. Needed for Gaussian projection: $$p_\text{cam} = R \cdot p_\text{world} + t$$.
- **Camera-to-World (c2w / pose):** transforms camera → world. Used by the original NeRF paper and **this assignment's dataset JSON files** (NeRF-style format). The camera center in world space is the translation column of the c2w matrix.

**The dataset provided stores camera-to-world (c2w) matrices.** To project Gaussians you need world-to-camera: `w2c = inv(c2w)`. The provided data loader handles this — but any custom camera code you write must be explicit about which convention it uses.

**Step 1 — Transform Mean to Camera Space**

Apply the world-to-camera transform to get the Gaussian mean in camera space:

$$p = R_\text{w2c} \cdot \mu + t \qquad p = [p_x, p_y, p_z]$$

**Step 2 — Compute the Projection Jacobian**

The perspective projection $$\pi(p) = \bigl(f_x \cdot p_x/p_z + c_x,\; f_y \cdot p_y/p_z + c_y\bigr)$$ has 2×3 Jacobian (from EWA Volume Splatting):

$$J = \begin{bmatrix} f_x/p_z & 0 & -f_x p_x/p_z^2 \\ 0 & f_y/p_z & -f_y p_y/p_z^2 \end{bmatrix}$$

_Implementation note:_ $$J$$ above is 2×3. For the product $$JW\Sigma W^TJ^T$$ to be conformant with the 3×3 matrices, pad $$J$$ to **3×3** by appending a zero third row. The result is a 3×3 matrix; only the top-left 2×2 block is used. This follows the EWA Volume Splatting convention used in the original 3DGS CUDA code.

**Step 3 — Project the 3D Covariance**

Using $$J$$ (3×3 padded) and $$W = R_\text{w2c}$$ (the upper-left 3×3 block of the w2c matrix):

$$\Sigma_\text{cam} = J\, W\, \Sigma\, W^T J^T$$

The 2D covariance is the **top-left 2×2 block**, discarding the depth dimension:

$$\Sigma' = \Sigma_\text{cam}[0\text{:}2,\; 0\text{:}2]$$

For this section, you will need to complete the code in the functions `compute_cov_3D`, `compute_cov_2D` and `compute_means_2D` of the class `Gaussians`.

<a name='evaluate'></a>
#### 5.1.2. Evaluate 2D Gaussians

In the previous section, we had implemented code to project 3D Gaussians to obtain 2D Gaussians. Now, we will write code to evaluate the 2D Gaussian at a particular 2D pixel location.

A 2D Gaussian is represented by the following expression:

$$f(\mathbf{x};\, \mu_i, \Sigma_i) = \frac{1}{2\pi|\Sigma_i|}\exp\!\left(-\tfrac{1}{2}(\mathbf{x}-\mu_i)^T\Sigma_i^{-1}(\mathbf{x}-\mu_i)\right) = \frac{1}{2\pi|\Sigma_i|}\exp\!\left(P(\mathbf{x},i)\right)$$

Here, $$\mathbf{x}$$ is a 2D vector that represents the pixel location, $$\mu_i$$ is a 2D vector representing the mean of the $$i$$-th 2D Gaussian, and $$\Sigma_i$$ represents the covariance of the 2D Gaussian. The exponent part $$P(\mathbf{x},i)$$ is referred to as _power_ in the code:

$$P(\mathbf{x},i) = -\tfrac{1}{2}(\mathbf{x}-\mu_i)^T\Sigma_i^{-1}(\mathbf{x}-\mu_i)$$

**Efficient 2×2 Inverse:** Use the 2×2 closed form:

$$\Sigma' = \begin{bmatrix}a & b \\ c & d\end{bmatrix} \implies (\Sigma')^{-1} = \frac{1}{ad-bc} \begin{bmatrix}d & -b \\ -c & a\end{bmatrix}$$

_Numerical stability:_ Add a small $$\epsilon \approx 10^{-6}$$ to the diagonal of $$\Sigma'$$ before inverting to prevent division by near-zero determinants.

The function `evaluate_gaussian_2D` of the class `Gaussians` is used to compute the power. In this section, you will have to complete this function.

**Unit Test:** To check if your implementation is correct so far, we have provided a unit test. Run `python unit_test_gaussians.py` to see if you pass all 4 test cases.

<a name='sort'></a>
#### 5.1.3. Filter and Sort Gaussians

Now that we have implemented functionality to project 3D Gaussians, we can start implementing the rasterizer!

Before starting the rasterization procedure, we should first sort the 3D Gaussians in increasing order by their depth value. We should also discard 3D Gaussians whose depth value is less than 0 (we only want to project 3D Gaussians that lie in front of the image plane).

<div class="fig fighighlight">
  <img src="/assets/2024/p4/depth_sort.png" width="80%">
  <div class="figcaption">
    Figure 4: Effect of depth sorting on alpha compositing correctness. Without sorting (left), occlusion is physically wrong. After sorting front-to-back (right), the result is correct.
  </div>
  <div style="clear:both;"></div>
</div>

Complete the functions `compute_depth_values` and `get_idxs_to_filter_and_sort` of the class `Scene` in `model.py`. You can refer to the function `render` in class `Scene` to see how these functions will be used.

<a name='alpha'></a>
#### 5.1.4. Compute Alphas and Transmittance

Using these N ordered and filtered 2D Gaussians, we can compute their alpha and transmittance values at each pixel location in an image.

The alpha value of a 2D Gaussian $$i$$ at a single pixel location $$\mathbf{x}$$ can be calculated using:

$$\alpha(\mathbf{x}, i) = o_i \exp\!\bigl(P(\mathbf{x}, i)\bigr)$$

Here, $$o_i$$ is the opacity of each Gaussian, which is a learnable parameter.

Given N ordered 2D Gaussians, the transmittance value of a 2D Gaussian $$i$$ at a single pixel location $$\mathbf{x}$$ can be calculated using:

$$T(\mathbf{x}, i) = \prod_{j < i}\bigl(1 - \alpha(\mathbf{x}, j)\bigr)$$

In this section, you will need to complete the functions `compute_alphas` and `compute_transmittance` of the class `Scene` in `model.py` so that alpha and transmittance values can be computed.

**Note:** In practice, when N is large and when the image dimensions are large, we may not be able to compute all alphas and transmittance in one shot since the intermediate values may not fit within GPU memory limits. In such a scenario, it might be beneficial to compute the alphas and transmittance in mini-batches. In our codebase, we provide the user the option to perform splatting `num_mini_batches` times, where we splat K Gaussians at a time (except at the last iteration, where we could possibly splat less than K Gaussians). Please refer to the functions `splat` and `render` of class `Scene` in `model.py` to see how splatting and mini-batching is performed.

<a name='splat'></a>
#### 5.1.5. Perform Splatting

Finally, using the computed alpha and transmittance values, we can blend the colour value of each 2D Gaussian to compute the colour at each pixel. The equation for computing the colour of a single pixel is (which is the same as equation (3) from the original paper).

More formally, given N ordered 2D Gaussians, we can compute the colour value at a single pixel location $$\mathbf{x}$$ by:

$$C_\mathbf{x} = \sum_{i=1}^{N} c_i\, \alpha(\mathbf{x}, i)\, T(\mathbf{x}, i)$$

Here, $$c_i$$ is the colour contribution of each Gaussian, which is a learnable parameter. Instead of using the colour contribution of each Gaussian, we can also use other attributes to compute the depth and silhouette mask at each pixel as well!

In this section, you will need to complete the function `splat` of the class `Scene` in `model.py` and return the colour, depth and silhouette (mask) maps. While the equation for colour is given in this section, you will have to think and implement similar equations for computing the depth and silhouette (mask) maps as well. You can refer to the function `render` in the same class to see how the function `splat` will be used.

Once you have finished implementing the functions, you can open the file `render.py` and complete the rendering code in the function `create_renders` (this task is very simple, you just have to call the `render` function of the object of the class `Scene`).

After completing `render.py`, you can test the rendering code by running `python render.py`. This script will take a few minutes to render views of a scene represented by pre-trained 3D Gaussians!

<div class="fig fighighlight">
  <img src="https://drive.google.com/uc?export=view&id=1hjmHlN4dS-LLKdeAGxujmuRZj8Had09H" width="100%">
  <div class="figcaption">
    Figure 5: Sample training view from the truck dataset alongside expected rasterizer outputs — RGB color (left), depth map (center), and silhouette mask (right). All three come from the same splatting computation with different per-Gaussian attributes substituted into the compositing equation.
  </div>
  <div style="clear:both;"></div>
</div>

For reference, here is one frame of the GIF that you can expect to see after running `render.py`:

<div class="fig fighighlight">
  <img src="/assets/2024/p4/Q1_Render.png" width="60%">
  <div class="figcaption">
    Figure 6: One frame of the expected rendering GIF. Do note that while the reference we have provided is a still frame, we expect you to submit the GIF that is output by the rendering code.
  </div>
  <div style="clear:both;"></div>
</div>

**GPU Memory Usage:** This task (with default parameter settings) may use approximately 6GB GPU memory. You can decrease/increase GPU memory utilization for performance by using the `--gaussians_per_splat` argument.

**Submission:** In your report, attach the GIF that you obtained by running `render.py`.

<a name='training'></a>
### 5.2. Training 3D Gaussian Representations (15 points)

Now, we will use our 3D Gaussian rasterizer to train a 3D representation of a scene given posed multi-view data.

More specifically, we will train a 3D representation of a **toy truck** given multi-view data and a point cloud. The folder `./data/truck` contains images, poses and a point cloud of the truck scene. The point cloud is used to initialize the means of the 3D Gaussians.

In this section, for ease of implementation and because the scene is simple, we will perform training using isotropic Gaussians. Do recall that you had already implemented all the necessary functionality for this in the previous section! In the training code, we just simply set `isotropic` to `True` while initializing `Gaussians` so that we deal with isotropic Gaussians.

For all questions in this section, you will have to complete the code in `train.py`.

<a name='optimizer'></a>
#### 5.2.1. Setting Up Parameters and Optimizer

First, we must make our 3D Gaussian parameters trainable. You can do this by setting `requires_grad` to `True` on all necessary parameters in the function `make_trainable` in `train.py` (you will have to implement this function).

Next, you will have to setup the optimizer. It is recommended to provide different learning rates for each type of parameter (for example, it might be preferable to use a much smaller learning rate for the means as compared to opacities or colours). You can refer to the pytorch documentation on how to set different learning rates for different sets of parameters.

Your task is to complete the function `setup_optimizer` in `train.py` by passing all trainable parameters and setting appropriate learning rates. Feel free to experiment with different settings of learning rates. Suggested starting values:

| Parameter | Suggested LR | Rationale |
|---|---|---|
| Means (positions) | $$1.6 \times 10^{-4}$$ | Small — preserve SfM geometry |
| Scaling | $$5 \times 10^{-3}$$ | Moderate — shapes adapt quickly |
| Rotation quaternions | $$1 \times 10^{-3}$$ | Moderate |
| Opacity logits | $$5 \times 10^{-2}$$ | Large — start near-zero, need to grow |
| Color / SH coefficients | $$2.5 \times 10^{-3}$$ | Moderate |

<a name='forwardpass'></a>
#### 5.2.2. Perform Forward Pass and Compute Loss

We are almost ready to start training. All that is left is to complete the function `run_training`. Here, you are required to call the relevant function to render the 3D Gaussians to predict an image rendering viewed from a given camera. Also, you are required to implement a loss function that compares the predicted image rendering to the ground truth image. Standard L1 loss should work fine for this question, but you are free to experiment with other loss functions as well.

The original paper uses a combined loss:

$$\mathcal{L} = (1-\lambda)\mathcal{L}_1(\hat{I}, I) + \lambda \mathcal{L}_\text{D-SSIM}(\hat{I}, I), \quad \lambda = 0.2$$

Standard **L1 loss** is sufficient for full credit. You are free to experiment with the combined loss for potentially better results.

Finally, we can now start training. You can do so by running `python train.py`. This script would save two GIFs (`q1_training_progress.gif` and `q1_training_final_renders.gif`).

For reference, here is one frame from the training progress GIF from our reference implementation. The top row displays renderings obtained from Gaussians that are being trained and the bottom row displays the ground truth. The top row looks good in this reference because this frame is from near the end of the optimization procedure. You can expect the top row to look bad during the start of the optimization procedure.

<div class="fig fighighlight">
  <img src="/assets/2024/p4/Q1_Training_1.png" width="100%">
  <div class="figcaption">
    Figure 7: One frame from the training progress GIF.
  </div>
  <div style="clear:both;"></div>
</div>

Also, for reference, here is one frame from the final rendering GIF created after training is complete:

<div class="fig fighighlight">
  <img src="/assets/2024/p4/Q1_Training_2.png" width="100%">
  <div class="figcaption">
    Figure 8: One frame from the final renders GIF.
  </div>
  <div style="clear:both;"></div>
</div>

Do note that while the reference we have provided are still frames, we expect you to submit the GIFs output by the rendering code.

Feel free to experiment with different learning rate values and number of iterations. After training is completed, the script will save the trained Gaussians and compute the PSNR and SSIM on some held out views.

**GPU Memory Usage:** This task (with default parameter settings) may use approximately 15.5GB GPU memory. You can decrease/increase GPU memory utilization for performance by using the `--gaussians_per_splat` argument.

**Submission:** In your report, include the following details:

- Learning rates that you used for each parameter. If you had experimented with multiple sets of learning rates, just mention the set that obtains the best performance.
- Number of iterations that you trained the model for.
- The PSNR and SSIM.
- Both the GIFs output by `train.py`.

<a name='extra'></a>
## 6. Extra Credit

<a name='ec1'></a>
### 6.1. Compare Against Official 3DGS &nbsp;&nbsp; _(+10 points)_

Install and run the **official 3DGS implementation** ([graphdeco-inria/gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting)) on the same truck dataset used in the base assignment. Compare your simplified PyTorch renderer against the official CUDA-accelerated implementation.

Your report must include:
- Side-by-side rendered frames from at least 3 held-out test views.
- A quantitative comparison table: PSNR and SSIM for both implementations on the same test split.
- Analysis of the performance gap: isotropic vs. anisotropic Gaussians? Degree-0 vs. degree-3 SH? PyTorch vs. CUDA rasterizer? Number of Gaussians after training? Be specific — attribute the gap to each factor if possible.
- A timing comparison: wall-clock training time and rendering fps for both implementations.

<a name='ec2'></a>
### 6.2. Run on a More Challenging Dataset &nbsp;&nbsp; _(+10 points)_

Run your 3DGS implementation (and optionally the official one) on a **more challenging scene** beyond the truck dataset. The following datasets are available:

**Additional data provided:** [Download additional scenes (Google Drive)](https://drive.google.com/drive/folders/1cK3UDIJqKAAm7zyrxRYVFJ0BRMgrwhh4)

Other publicly available datasets:
- **NeRF Synthetic / Blender** — 8 synthetic objects (lego, drums, ship, hotdog, etc.) placed on white backgrounds with complex geometry and challenging view-dependent effects. [Download (Google Drive)](https://drive.google.com/drive/folders/128yBriW1IG_3NJ5Rp7APSTZsJqdJdfc1)
- **Mip-NeRF 360** — 4 indoor + 5 outdoor unbounded object-centric scenes (bicycle, garden, stump, kitchen, etc.). [Download (360\_v2.zip)](http://storage.googleapis.com/gresearch/refraw360/360_v2.zip)
- **Tanks and Temples + Deep Blending** — large-scale outdoor/indoor scenes; the standard benchmark used in the original 3DGS paper. [Download (tandt\_db.zip, 650 MB)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/datasets/input/tandt_db.zip)
- **Your own captured dataset** — capture ≥ 50 images of a scene yourself (do **not** download from the internet), run COLMAP, calibrate and undistort, then run 3DGS. Analyze the success and the failure of your algorithm and showcase that in your report. Note: you will need to capture images, calibrate them, and undistort them.

Your report must include:
- Description of the chosen dataset: scene type, number of images, resolution, and why it is more challenging than the truck dataset.
- PSNR and SSIM on the test split.
- Novel-view rendered GIF or frames.
- Analysis of **successes and failure cases**: where does 3DGS do well? Where does it struggle? (e.g., reflective surfaces, thin structures, unbounded background.) Explain why each failure occurs based on your understanding of the algorithm.

<a name='dataset'></a>
## 7. Notes about the Dataset

Run your 3DGS algorithm on the **truck scene** provided in the [starter code download](https://drive.google.com/file/d/1saZDsOn_7h37rer3N_KjsgsQYFHGS2Cr/view?usp=sharing). The data given to you are a set of images of a toy truck captured from multiple viewpoints, along with a point cloud produced by COLMAP and camera pose files. The point cloud is used to initialize the Gaussian means.

The data folder `./data/truck` contains:
- **Images:** posed RGB images of the truck from multiple viewpoints. The last $$N_\text{test}$$ views are held out for evaluation.
- **Camera poses:** `transforms_train.json` / `transforms_test.json` in NeRF-style **camera-to-world (c2w)** format (see §5.1.1 warning on convention).
- **Point cloud:** A sparse `.ply` file produced by COLMAP, with per-point RGB color, used to initialize Gaussian means.
- **Calibration:** Camera intrinsic parameters $$(f_x, f_y, c_x, c_y)$$ are embedded in the JSON files.

A data loader is provided — you do not need to write one. Please **DO NOT** include the dataset in your submission.

**For the extra credit:** Also run your 3DGS algorithm on the additional scenes provided at [this link](https://drive.google.com/drive/folders/1cK3UDIJqKAAm7zyrxRYVFJ0BRMgrwhh4) or on your own captured images. Analyze the success and the failure of your algorithm and showcase that in your report.

<a name='sub'></a>
## 8. Submission Guidelines

<b>If your submission does not comply with the following guidelines, you'll be given ZERO credit.</b>

<a name='files'></a>
### 8.1. File Tree and Naming

Your submission on ELMS/Canvas must be a `zip` file, following the naming convention `YourDirectoryID_p4.zip`. If your email ID is `abc@umd.edu` or `abc@terpmail.umd.edu`, then your DirectoryID is `abc`. For our example, the submission file should be named `abc_p4.zip`. The file **must have the following directory structure** because we'll be autograding assignments. The file to run for your project should be called `Wrapper.py`. You can have any helper functions in sub-folders as you wish, be sure to index them using relative paths and if you have command line arguments for your Wrapper codes, make sure to have default values too. Please provide detailed instructions on how to run your code in `README.md` file. Please **DO NOT** include data in your submission.

```
YourDirectoryID_p4.zip
│   README.md
|   Your Code files
|   ├── model.py
|   ├── render.py
|   ├── train.py
|   ├── unit_test_gaussians.py
|   ├── Wrapper.py
|   ├── Any subfolders you want along with files
|   Wrapper.py
|   Outputs
|   ├── q1_render.gif
|   ├── q1_training_progress.gif
|   ├── q1_training_final_renders.gif
|   ├── RenderFrames/
|   ├── TrainingFrames/
|   ├── ExtraCredit/
└── Report.pdf
```

<a name='report'></a>
### 8.2. Report

There will be no Test Set for this project. For each section of the project, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented. You must include the following details in your writeup:

- Please make your report extremely detailed with PSNR and SSIM after each step (rasterizer outputs, training convergence, before and after ADC ablation, and so on). Describe all the steps (anything that is not obvious) and any other observations in your report.
- Your report **MUST** be typeset in LaTeX in the IEEE Tran format provided to you in the `Draft` folder and should be of a conference quality paper.
- Present the following in your report:
	- Section 5.1: the output GIF from `render.py`, rendered color + depth + silhouette images from the pre-trained scene.
	- Section 5.2: learning rates used and justification; number of iterations; training progress GIF and final renders GIF; PSNR and SSIM on test views.
	- ADC ablation: results with full ADC / without ADC / pruning only.
	- Extra Credit 6.1 (if attempted): side-by-side frames, PSNR/SSIM comparison table, gap analysis, timing comparison.
	- Extra Credit 6.2 (if attempted): dataset description, PSNR/SSIM, novel-view renders, success/failure case analysis.
- Present failure cases and explanation, if any.
- Do not use any function that directly implements a part of the pipeline. If you have any doubts, please contact us via Piazza.

<a name='coll'></a>
## 9. Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup. For the full honor code refer to the CMSC733 Spring 2024 website.

---

## References

[1] Kerbl, B., Kopanas, G., Leimkühler, T., and Drettakis, G. "3D Gaussian Splatting for Real-Time Radiance Field Rendering." _ACM Transactions on Graphics (SIGGRAPH 2023)_, 42(4). [[Project]](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) [[arXiv:2308.04079]](https://arxiv.org/abs/2308.04079) [[Code]](https://github.com/graphdeco-inria/gaussian-splatting)

[2] Yurkova, K. "A Comprehensive Overview of Gaussian Splatting." _Towards Data Science_, Dec 2023. [[Link]](https://towardsdatascience.com/a-comprehensive-overview-of-gaussian-splatting-e7d570081362/)

[3] "Introduction to 3D Gaussian Splatting." _Hugging Face Blog_, 2023. [[Link]](https://huggingface.co/blog/gaussian-splatting)

[4] kwea123. "Gaussian Splatting Notes: A detailed formulae explanation." GitHub, 2023. [[Link]](https://github.com/kwea123/gaussian_splatting_notes)

[5] joeyan. "Gaussian Splatting MATH.md." GitHub, 2023. [[Link]](https://github.com/joeyan/gaussian_splatting/blob/main/MATH.md)

[6] Bouamer, T. "A Comprehensive Study for Gaussian Splatting." 2025. [[Link]](https://tarekbouamer.github.io/posts/gaussian-splatting/)

[7] "Introduction to 3D Gaussian Splatting." _Summer Geometry Institute 2024_. [[Link]](https://summergeometry.org/sgi2024/introduction-to-3d-gaussian-splatting/)

[8] "3D Gaussian Splatting — Paper Explained." _LearnOpenCV_, 2024. [[Link]](https://learnopencv.com/3d-gaussian-splatting/)

[9] Mildenhall, B., Srinivasan, P.P., Tancik, M., Barron, J.T., Ramamoorthi, R., and Ng, R. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." _ECCV 2020_. [[Project]](https://www.matthewtancik.com/nerf)

[10] Zwicker, M., Pfister, H., van Baar, J., and Gross, M. "EWA Volume Splatting." _IEEE Visualization 2001_.

[11] nerfstudio team. "Data Conventions." _nerfstudio documentation_, 2023. [[Link]](https://docs.nerf.studio/quickstart/data_conventions.html)

[12] Tulsiani, S. _16-825: Learning for 3D Vision_. Carnegie Mellon University. [[learning3d.github.io]](https://learning3d.github.io/)

---

_This article is written by Naitri Rajyaguru. If you have any questions/corrections, please email nrajyagu[at]umd[dot]edu. Adapted from [16-825: Learning for 3D Vision](https://learning3d.github.io/), Shubham Tulsiani, CMU._
