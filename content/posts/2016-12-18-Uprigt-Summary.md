---
title: Upright Orientation Detection Methods
date: 2016-12-15
toc: true
comments: true
categories:
- Computer Graphics
tags:
- Computer Graphics
- Research Log
---

## Introduction

Detecting the upright orientation of a 3d shape is a basic yet difficult problem in CG literature. It often serves as a preprocessing step for many geometry processing and shape analysis algorithms such as shape recognition and retrieval.

![Upright orientation of a chair](http://7xllm5.com1.z0.glb.clouddn.com/upright.png)

(*Upright orientation of a chair*)

<!-- more -->

For many shapes, there exist natural bases where they naturally stand on so the upright direction should be the normal to that base. However, other shapes (consider a swimmer) may not exhibit a clear standing base so telling this problem could be more subtle.

The following sections summarize existing methods (to the best of my knowledge, they are all of them) and address their issues and limitations.

## Known Methods

1. Fu08
  - Pipeline:
    - computing convex hull
    - generating hand-crafted features
    - training on random forest + support vector machine
  - Data Set: Princeton Shape Benchmark
  - Precision: 87.5%
  - Timing: 10.62s (2.13GHz)
2. Lin12
  - Pipeline:
    - computing convex hull
    - generating hand-crafted features
    - combining scores linearly
  - Data Set: Princeton Shape Benchmark (120 selected)
  - Precision: 79%
  - Timing: 154s (2.33GHz *including time to compute best view*)
3. Jin12 & Wang14 (*the second method is an improvement on the first one, statistics below accounts for the newer method only*)
  - Pipeline:
    - optimizing against the projection/tensor matrix's rank
    - low rank matrix corresponds to a model posed in axis-aligned manner
    - finding the base from 6 candidate orientations
  - Data Set: Princeton Shape Benchmark
  - Precision: about 70%
  - Timing: 1-2mins (3.30GHz)
4. Han15
  - Pipeline:
    - using view selection methods to compute view scores
    - clustering views whose scores are low as model base candidates
    - determining upright orientation w.r.t. several hand-crafted features
  - Data Set: Princeton Shape Benchmark
  - Precision: 79%
  - Timing: 7.8s (2.0GHz)
5. Liu16
  - Pipeline:
    - voxelizing shapes
    - training a classification convnet to dispatch input shapes
    - training a regression convnet for each shape category to find upright orientation
  - Data Set: ModelNet10
  - Precision: 90.1% (*with convnets clustring*)
  - Timing: 0.15s (3.20GHz)

## Comments
1. Most man-made models are supposed to stand on its own so computing the convex hull and assigning a hull face as supporting base is appropriate. On the other hand, natural models like human may pose large deformation so there is no clear supporting base. Previous methods failed to solve this conflict.
2. Method 1 and 5 are still short on training data and the generation power on unseen models is limited.
3. Other methods based on hand-crafted features and criterions usually give inferior results.

## References

Fu H, Cohen-Or D, Dror G, et al. Upright orientation of man-made objects[C]. ACM transactions on graphics (TOG). ACM, 2008, 27(3): 42.

Lin C K, Tai W K. Automatic upright orientation and good view recognition for 3D man-made models[J]. Pattern Recognition, 2012, 45(4): 1524-1530.

Jin Y, Wu Q, Liu L. Unsupervised upright orientation of man-made models[J]. Graphical Models, 2012, 74(4): 99-108.

Wang W, Liu X, Liu L. Upright orientation of 3D shapes via tensor rank minimization[J]. Journal of Mechanical Science and Technology, 2014, 28(7): 2469-2477.

Han HL,Wang WC,Hua M.Getting upright orientation of 3D objects via viewpoint scoring.Ruan Jian Xue Bao/Journal of Software,2015,26(10):2720?2732(in Chinese).

Liu Z, Zhang J, Liu L. Upright orientation of 3D shapes with Convolutional Networks[J]. Graphical Models, 2016, 85: 22-29.
