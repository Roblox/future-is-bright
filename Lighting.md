Overview
===

We currently use one lighting system across all of Roblox, which is using voxels and is updated incrementally/lazily on CPU. It has served us well over the last 4 years, but it needs an update to match the future vision. Lighting is complicated - we should be careful when selecting the next generation technology, as there are many factors/tradeoffs to consider. Two future systems have been prototyped to help guide our decision, which will be discussed further in this document; we’ll call them “Voxels” and “Shadow maps”, although there’s more to them. It’s important to understand how both of them work to understand the limitations. There’s a summary table at the end.

Implementation - voxels
===

This is a significant evolution of our existing voxel system. First, the world data is voxelized into a set of voxel grids - each grid is centered around the character and has increasing voxel sizes, e.g. all the way from 1 stud to 16 studs (5 grids in total). Each voxel contains occupancy information (0-100% full). After this, we compute lighting data for each voxel in each grid, based on the occupancy in each voxel, and on the light sources / sun light direction. All of the above happens on the GPU, as CPU is not fast enough to update so many voxels at such a high density.
 
The system stores all data in voxels, specifically for each voxel we have:
* Occupancy (multiple values describing how full each voxel is)
* Skylight (how much of the sky is visible from the voxel)
* Sun shadow (how much of the sun is occluded from the voxel)
* Light object color/cone (the approximation of the color/cone of impact of local light sources on the voxel)
 
This information is later used to compute the color of each pixel at a given resolution; screen resolution and voxel resolution can be controlled independently. Parts of voxel grid(s) can be updated each frame as lights/objects move.

Implementation - shadow maps
===

This method uses rasterization to compute most of the shadow effects and executes in three phases. First, for each shadow casting light we update a shadow map by rendering triangles of objects into a texture from the viewpoint of the light (you can think of this as casting a lot of rays from the light source into the scene and remembering the intersection results). Second, we build a spatial acceleration structure for each visible light object, that is essentially a frustum-shaped voxel grid (froxel grid, http://cheneyshen.com/wp-content/uploads/2016/09/090316_0800_SIGGRAPH15L83.png), within each froxel we record the list of all light objects that intersect it. Finally, when rendering the scene, for each pixel to compute the impact of all lights we look up the froxel our pixel is contained in, go over all lights, and for each light compute the impact of this light using the shadow maps build in step one to determine visibility.
 
The system stores all data in two structures:
* Shadow atlas (all shadow maps from visible lights, packed into one big texture)
* Light grid (froxel grid that can efficiently resolve a point in the camera into a list of lights)
 
The color of each pixel is computed dynamically and is not stored explicitly. Parts of the shadow atlas can be updated each frame as lights/objects move.

Performance - voxels
===

The voxel technique is fundamentally more scalable - as long as we’re willing to degrade the output quality, we can reduce the number of voxel grids, and update fewer voxels each frame (leading to “light lag” - light impact of the objects will update more slowly than the objects themselves). In terms of complexity, voxels decouple all three sources of complexity: geometrical complexity, light complexity, pixel count - geometrical complexity only impacts voxelization cost, so putting more objects doesn’t make anything else slower; light complexity only impacts light computation cost which does not depend on geometrical complexity *or* the number of pixels; finally, computation of the final pixel color is constant-time in the number of voxels/lights/parts, so we can scale the resolution independently without increasing the lighting performance impact.
 
You can think of voxel worst-case performance as O(G)+O(L)+O(P), where G is # of triangles (geometry complexity), L is # of lights, P is # of pixels.
 
Unfortunately, the peak performance of voxels is suboptimal because the number of voxels scales as N^3, and GPUs aren’t ideally suited for the update techniques we have to use to keep performance manageable. Given enough research into GPU computing we should be able to offset the performance loss, but the current baseline cost can be pretty high.

Performance - shadow maps
===

For shadow maps, the technique is more GPU-friendly as it’s designed around rasterization which is what GPUs excel at; the shadow atlas update cost can be partially mitigated by caching/delaying updates (leading to “shadow lag”). For the shadow maps that we do update, optimizing the efficiency of geometry submission (including mesh level of detail) reduces the cost.
 
However, the baseline cost of shadow update in complex scenes is significant since it depends on both the amount of geometric detail and the amount of shadow-casting lights. For a moving light with a large radius inside the building, we have to re-render the entire building each frame to update the shadow information for this light; many moving shadow casting lights in a building result in poor performance (we can choose to update only some lights each frame, resulting in visual artifacts).
 
Additionally, the technique does not provide a way to separate resolution from the light count - for each pixel we have to recompute the influence of all lights that cover this pixel. This step also can not be cached, leading to performance issues on high resolutions in densely lit scenes - 20 overlapping lights in a room at 4K resolution might require 160M light evaluations).
 
You can think of shadow map worst-case performance as O(G*L)+O(L*P), where G is # of triangles (geometry complexity), L is # of lights, P is # of pixels.

Performance - evaluation
===

To put the theoretical results above in a more practical environment, here are results from some of the levels we have tested. These results use *existing* implementations - that have not been necessarily tuned for performance - and also assume *no* caching whatsoever - every light source/part in the world is considered moving.

Paris (sun shadows, very few non-shadow-casting lights)
---

[voxels]
Voxels: 6 ms shadow update, 1.5 ms scene render
Shadow maps: 1 ms shadow update, 2.4 ms scene render
Explanation: Baseline voxel shadow computation cost is larger since it’s not GPU-friendly

Caves (many shadow casting lights)
---

[shadow maps]
Voxels: 7 ms shadow update, 0.9 ms scene render
Shadow maps: 10 ms shadow update, 2.1 ms scene render
Explanation: a lot of geometry and moving lights make shadow map update expensive

Western (many shadow casting lights)
---

[voxels]
Voxels: 8 ms shadow update, 1 ms scene render
Shadow maps: 15 ms shadow update, 2.5 ms scene render
Explanation: with moving light sources and large # of triangles, shadow map update becomes expensive

Lights (1000 non shadow casting lights)
---

[shadow maps]
Voxels: 20 ms light update, 0.5 ms scene render
Shadow maps: 0.5 ms light update, 5 ms scene render
Explanation: cumulative volume of light-voxel overlap in this case makes voxel light update slow

Performance - conclusion
===

In general shadow maps seem to scale nicely to the workload, however two big concerning areas are:
 
Per pixel cost grows as the resolution keeps bigger, making this solution only practical on average (1080p) resolutions; going beyond 1080p requires a *very* good GPU
Shadow cost grows very quickly when many shadow-casting dynamic lights are close to complex geometry. This can be offset by better culling but remains a fundamental problem.
 
In contrast, voxel performance depends on the level contents much less, but has a much higher baseline. This can be offset by better GPU algorithms and by reducing the # of voxels.

Quality - lights
===

Shadow map solution provides the ground truth in terms of simulating the lights - the screenshot with 1000 lights above is made with shadow maps, and you can see that the specular highlights - modeled with a BRDF that is getting us basically light reflections - are perfectly preserved.
 
Voxel solution is fundamentally worse in that it approximates light influence at each voxel as if it just comes from one light, so in general you see the specular quality suffer; for example, here’s a screenshot with two lights (green & red) over a highly reflective surface, taken with shadow maps:
 [shadow maps]
 
Here’s a screenshot from the same camera, using voxel lighting solution:
 [voxels]
 
As you can see, while in voxels away from the center of the camera you see distinct colors, in the area with specular highlights the colors merge in 1:1 ratio, producing yellow specular highlight even though there are no yellow lights in the scene.
 
In some cases the approximation we use produces results that very unconvincing, although we could improve this to some degree:
 [voxels]
 
You can see curved, elongated and miscolored specular highlights, and a few voxels below one of the parts are just missing light information. Same screenshot for shadow maps provides a much better result:
 [shadow maps]
 
Quality - shadows
===

In general a defining quality of shadow maps is fidelity, and a defining quality of voxel shadows is softness. Shadow maps produce pretty crisp shadows, with minimum representable detail sufficiently high to render out a convincing character shadow:
 
 [shadow maps]
 
Our voxel shadow algorithm is very good at producing really soft shadows, but since the smallest voxel size is 1 voxel, the shadows from small parts either don’t register at all, or have vastly incorrect shapes:
 
 [voxels]
 
For this reason we currently use a shadow map variant to render shadows from the characters - this, however, is a duct tape solution in the sense that it only applies to the sun casting shadows from character, other light sources and/or objects aren’t affected).
 
Quality - skylight
===

An important feature that we support in the voxel pipeline is computing the skylight factor - how visible is the sky from the current voxel? This is used to blend between outdoor and indoor lighting conditions and is very effective at enhancing the lighting quality. Consider this screenshot (using voxels):
 
 [voxels]
 
Here you can see that it’s much brighter outside the house than it is inside the house, even in the areas that are in the shadow of the house. In contrast, shadow maps do not provide a solution for this factor, which makes the picture look dull:
 
 [shadow maps]
Quality - geometrical fidelity
===

It’s worth noting the fundamental differences in geometry representation between voxels and shadow maps.
 
Voxels assume that all objects that the lighting engine supports can be “voxelized” - that is, for each voxel in the world there’s a fast way to compute the volume of intersection between that object and the voxel. This is analytically computable for primitive shapes, but complex objects like CSGs and MeshParts present a significant challenge in this area. Currently we rely on a crude convex decomposition and a set of hacks to voxelize these efficiently, that often result in visible artifacts:
 
 [voxels]
 
Shadow maps, on the other hand, use the same polygonal representation we use for rendering, thus they are able to represent the shapes of all objects perfectly:
 
 [shadow maps]
Quality - light leaks
===

While the shape of the shadow is important, what perhaps is even more important is that pixels that should be completely invisible from the light source point of view are treated as such. When various approximations violate this, you get what’s called light leaking - visible strips of light, that are particularly problematic in high-contrast environments, such as being inside a building with bright sun outside. Here’s an example of a light leak:
 
 [voxels]
 
The rough shape of the light cast through the window isn’t too objectionable here, but what *is* objectionable is the thin lit part of the floor right next to the wall.
 
For voxels there are multiple sources of leaking; we try to mitigate some of that by storing anisotropic occupancy - we store 3 values per voxel, signifying “how much matter is in the voxel projection along the axis A” for axes X/Y/Z; unfortunately, while this helps thin parts cast shadows regardless of their thickness, it can’t eliminate all sources of leaking. The only way for a part to guarantee that the light is completely blocked off is to make it twice as thick as a voxel is, so 2 studs. Additionally, leaking increases with the voxel size, which means that on lower quality levels and/or far away leaking is more pronounced.
 
Shadow maps aren’t completely leak proof but leaking is significantly less of a problem - with our shadow map solution a part that’s 0.4 studs thick will have no visible light leaking (a part that’s 0.2 studs thick can still leak a bit of light, which might be something we can improve in the future).
 
Quality - conclusion
===
Overall, shadow maps excel at most quality aspects; the only significant area of improvement is in figuring out a method of computing the skylight factor (it’s possible that this requires a hybrid technique that uses voxelization for skylight - which introduces some problems of the voxel pipeline -  or maybe there are alternative solutions to this problem).
 
Voxels have reasonable quality, but compared to shadow maps they face many issues, particularly with shadow fidelity and specular highlights; we’ll have to address both issues in some way to be able to ship voxel lighting because the current form of voxel lighting can’t deliver good looking player shadows, and using our current shadow solution means we still don’t get player shadows from light sources other than the sun, which seems incompatible with the future vision.
Vision - translucency
TODO: smoke shadows, improve text
 
What voxels lose in shadow quality, they gain in being more general - for example, they effortlessly render shadows from transparent objects:
 
 [voxels]
 
For shadow maps this is not currently supported; this means that if we want to support particles or other transparent objects casting shadows, we may not be able to do this with shadow maps.

Vision - vegetation
===

While shadow maps can’t represent translucency very well, what they can do is represent small features of objects (such as vegetation) regardless of whether they are modeled with geometry or with textures. Voxels aren’t small enough to serve this use case (and also it’s not easy to access the texture information since it requires precisely modeling the surface of the mesh instead of the volume). It seems unlikely that we can ever get good looking shadows from vegetation with voxels, whereas shadow maps can support this use case even with existing content, as shown on this screenshot:
 
 [shadow maps]

Vision - self-illumination
===

Due to how voxels are implemented, it’s relatively simple to inject light from light sources of arbitrary shape and quantity into the grid without affecting other parts of the pipeline performance-wise; while you can add many light sources to the shadow maps, making lights with custom shapes has architectural and performance penalties. Specifically, what is much easier to do with voxels compared to shadow maps is true self-illumination: currently we support Neon material that “emits light”, but it doesn’t actually inject light onto other objects near it (which can be thought of a form of global illumination).
 
TODO: screenshot

Vision - global illumination
===

One big area that we see real-time engines increasingly moving towards solving is GI - global illumination - which means computing secondary light effects, such as light from a lamp bouncing two times off the walls to provide extra illumination for areas that aren’t directly visible to the light.
 
GI in Roblox is unprecedentedly complicated - most solutions to GI sacrifice at least one of (dynamic lights, dynamic geometry, real-time performance, large scenes, good results without tuning), whereas we need all of these. Interestingly, skylight factor and self-illumination mentioned above are all parts of a general GI solution.
 
It’s not clear which GI solutions will be practical within extremely lax content constraints we have; so far voxel-based GI seems more promising than other approaches when we need to consider all of the above.
 
Of course, having voxel-based GI doesn’t really enforce that direct lighting is computed using voxels - most research around voxel-based GI today implies using shadow maps to compute direct light visibility, and enhance the results using voxels.
 
We have done early experiments in this area but it’s too early to conclude whether or when this can be possible in Roblox.

Summary
===

Based on the analysis above, we can summarize the tradeoffs that these two solutions provide. Cells using italic suggest that it might be possible to improve this area with further research. For the purpose of this table, Terrible < Poor < Okay < Good < Excellent.

|  | Voxels | Shadow maps |
|---|---|---|
| Performance: baseline | Poor | Excellent
| Performance: local lights | Okay | Good
| Performance: light shadows | Good | Okay
| Performance: resolution | Excellent | Okay
| Performance: geometry | Excellent |  Okay
| Quality: lights | Okay | Excellent
| Quality: macro shadows | Good | Excellent
| Quality: character shadows | Poor | Excellent
| Quality: skylight | Excellent | Terrible
|  Quality: geometrical fidelity | Okay | Excellent
Quality: light leaks (>2 studs)
Good
Excellent
Quality: light leaks (<2 studs)
Poor
Good
Vision: translucency
Excellent
Poor
Vision: vegetation
Terrible
Good
Vision: self-illumination
Excellent
Poor
Vision: global illumination
Okay?
Terrible
 
