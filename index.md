---
layout: default
title: Future Is Bright
---

# Hack Week

We are always trying to improve Roblox by enhancing the fidelity and the scale of our simulation. In particular, our lighting engine has served us well over the years but limits the creative power of our developers, which is why over the last few years we have built several lighting prototypes during hack week:

| Hack Week 2015 (Next Generation Voxels) | Hack Week 2016 (Future Is Bright) |
|:-:|:-:|
| [![](https://img.youtube.com/vi/z5TmqDtpwSM/0.jpg)](https://www.youtube.com/watch?v=z5TmqDtpwSM) | [![](https://img.youtube.com/vi/lrvOGqC9ZjQ/0.jpg)](https://www.youtube.com/watch?v=lrvOGqC9ZjQ) |

The focus of these hack week projects was to push the boundaries of quality of content on Roblox; since it became clear that both represent significant improvements in the lighting technology that our community wants, we decided to invest more time in this.

# Exploration

Since we had two competing approaches, we could not just pick any one of them - we had to pick the right one. There are many factors to consider when building a new lighting engine:

* What kind of features can it have now?
* What kind of features can become possible as future extensions?
* How well does the engine perform on existing content?
* How well does the engine perform in extreme cases?
* What is the range of hardware the system can run on?

We tried to answer these questions and more by taking the code built during the hack weeks and developing it further to incorporate existing Roblox features as well as implementing some possible future extensions. Both approaches were integrated into one build that made it easy to compare the quality and performance on the same levels on the same hardware.

Based on this exploration, we have created a comparative analysis document that we want to share:

[Comparison between two new lighting engines with screenshots](compare)

After building this out, it became obvious that this is a tradeoff - each system has some nice properties that the other system doesn't, and we are still debating internally which system we should go with. This is a hard decision because we have to balance many variables and pick the best system that can serve us well for years to come.

# Prototype

In the spirit of transparency, we decided that instead of just making this decision ourselves we should also gather feedback from our community. We could just share the document and let you pick, but there are a lot of subtleties that you can only really notice when trying to build content for new systems, which is why instead we shared the prototype builds with you (check out the Releases page for v1-v11 builds).

The builds had three lighting engines built in, that you could switch between with Lighting.LightingMode property:

* VoxelCPU - the current voxel engine (4^3 voxels);
* VoxelGPUCascaded - the new voxel engine from "Next Generation Voxels" video (1^3 voxels and other improvements);
* ShadowMap -  the new shadow map-based engine from "Future Is Bright" video.

Based on this exploration and the community feedback, we have settled on the approach we want to take, detailed below.

# Hybrid

Both VoxelGPUCascaded and ShadowMap engines had their own strenghts and weaknesses in terms of quality and performance. We also had to balance this with the performance and quality on low-end systems. After a lot of experimentation we have settled on a plan going forward:

- On low-end systems, we'll continue to use CPU-based voxel engine.
- We will improve some key aspects of the CPU-based voxel engine while maintaining the high performance and versatility
- On high-end systems, we'll use ShadowMaps for most of the lighting work, and the same CPU-based voxel engine for skylight
- Depending on the quality level, different portions of the screen will be using different engines for different components; we'll try our best to minimize visual discrepancies.

Starting from v12, we're now focusing on this hybrid technology, and are working on quality on both low-end and high-end, and performance. You can download a preview build here:

- [Download Windows build (.zip, ~120 MB)](https://github.com/Roblox/future-is-bright/releases/download/v12/future-is-bright-v12.zip); updated 3/20/2018 (v12)
- [Download macOS build (.zip, ~130 MB)](https://github.com/Roblox/future-is-bright/releases/download/v12/future-is-bright-v12-mac.zip); updated 3/20/2018 (v12)

This is a custom build of Roblox Studio. Make sure to copy the folder from this .zip to your computer before running the build inside - don't run directly from .zip. The Windows build requires Windows 7 (or higher) and a mid-tier DirectX 10 compatible GPU - this does *not* mean that either lighting engine can only work on these systems, but limiting the supported hardware for the prototype allows us to iterate much faster and release the prototype to you much sooner. The Mac build requires Metal and a recent macOS release (there may be issues on early OS versions such as OSX 10.11).

<img src="images/mode_switch.png">

To activate new lighting engine, make sure to enable FutureIsBright property in Studio settings, and restart Studio after that.

There are some other properties you might want to experiment with, such as Lighting.ShadowSoftness/Light.ShadowSoftness and Light.ShadowCutoff (disables shadows for parts really close to the light). The build also uses an experimental High-Dynamic Range implementation, which means that for both new lighting engines you can use values for light brightness that exceed 1; if you have existing content with high Brightness values you might have to adjust these for it to look better.

Finally, you can adjust hybrid blending parameter via Lighting.HybridBlendDist and Lighting.HybridBlendSoftness; these will eventually be driven through the quality level.

We are excited to see what you build and hope that you will help us make the right choice by sharing the content you build and the problems you encounter (you can [report issues on GitHub](https://github.com/Roblox/future-is-bright/issues)).

# Results

We released the prototype build on July 22nd and the response has been nothing short of astounding. [Here are some of the screenshots that Roblox developers shared with us in 60 hours since we released the build](results).
