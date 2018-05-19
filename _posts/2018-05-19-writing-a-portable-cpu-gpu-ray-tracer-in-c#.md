---
layout: post
title:  "Writing a Portable CPU/GPU Ray Tracer in C#"
date:   2018-05-19 10:07:00 -0700
categories: graphics
---

![Book Scene - 64 Samples Per Pixel](/images/BookSceneOutput.png){:class="img-responsive"}

Ray tracing is getting a lot of hype lately. Lots of advanced rendering techniques are emerging that involve scene tracing of some kind, there are several frameworks for high-performance ray tracing, D3D12 is getting [integrated support for ray tracing](https://blogs.msdn.microsoft.com/directx/2018/03/19/announcing-microsoft-directx-raytracing/), and there's a [proposed Vulkan extension](http://on-demand.gputechconf.com/gtc/2018/presentation/s8521-advanced-graphics-extensions-for-vulkan.pdf) for the same. I've written (bad) ray tracers in the past, but it's been a while and I wanted to get back up to speed with how things are done. Lots of people are following along with [Ray Tracing in One Weekend](http://in1weekend.blogspot.com/2016/01/ray-tracing-in-one-weekend.html) by Peter Shirley, which is a nice, short book going over the fundamentals of ray tracing. The structure of my ray tracer is adapted roughly from the C++ code in that book.

However, I wanted to try something more interesting than just a simple translation to C#. Although .NET is quite fast these days, it still can't compare to how fast a GPU will chew through computations for a ray tracer. Enter [ShaderGen](https://github.com/mellinoe/shadergen), a project which lets you author portable shader code in C#. You give it regular C# structures and methods, and it automatically converts them to HLSL, GLSL SPIR-V, and Metal shaders. I wanted to see how far I could get by pushing my C# code through ShaderGen and using the resulting compute shaders with [Veldrid](https://mellinoe.github.io/veldrid-docs/). In theory, I can use the same C# code to run my ray tracer on the CPU *and* the GPU, across a bunch of graphics API's -- Vulkan, Direct3D, Metal, and OpenGL.

## Goals

Ahead of time, I knew of a few snags that I was going to hit when doing this, but these were my primary goals:

* Write all of the ray tracing logic in C#, including the compute shaders running on the GPU.
* _As much as possible_, use the same C# code on the CPU and GPU.
* Use the same code to run on Vulkan, Direct3D, Metal, and OpenGL.

## GPU-friendly Tracing

The structure in Peter Shirley's book is a good starting point, but it needs to be changed a bit to run in a compute shader. The original code's "Sphere" class contains a pointer to an abstract "Material" object which has a virtual function controlling how instances behave. This pattern won't work in a compute shader. In my version, a Material is another simple, flat structure, and there's an array of them sitting next to the array of Spheres -- one Material per Sphere. Instead of a virtual function, the Material struct just contains an enumerated value identifying its type, and the tracing logic switches off of that. All of these patterns work perfectly fine in C#, but you might not reach for them if you were writing a regular CPU ray tracer.

Another obvious limitation is that GPU code cannot recurse. A simple ray tracer will recurse for each reflection and refraction ray, up to a max depth, because it's elegant and clean. In a GPU tracer, you'll instead need to use an explicit loop, bounded by the max depth, which accumulates the color of reflected and refracted rays.

![ToyPathTracer Scene - 64 Samples Per Pixel](/images/ToyPathTracerOutput.png){:class="img-responsive"}

## Results

All of the code is available in the [Veldrid Ray Tracer repo](https://github.com/mellinoe/veldrid-raytracer) on GitHub. Check out the README file there for instructions on how to run the program.

Ultimately, I was able to share most of the code between the two versions. There were some limitations (see below), some of which I was aware of already, on how much code could be shared. Obviously, there is some baseline "entry point" code that differs between a compute shader and a C# application. A compute shader runs inside a "thread" set up by the graphics API and drivers. Thread scheduling and dispatch is handled automatically -- all that's needed is the code that grabs the predefined dispatch ID, converts it to a screen coordinate, and traces a ray from it. In the C# app, though, job scheduling is handled manually. For simplicity, I've just used the built-in Parallel.For method to loop over each row of pixels in the output texture. Each job then loops over its row of pixels and traces rays through the scene in the same way as the compute shader. There's not much of a difference here, and this is only a small part of the ray tracer.

I set up two test scenes: one from the final chapter of Peter Shirley's book, and the other from Aras Pranckevičius's [ToyPathTracer project](https://github.com/aras-p/ToyPathTracer) (image above). I've run both scenes on a couple of my machines, on the CPU and on several graphics API's. I wasn't really trying to squeeze the best performance out of this project, but I think it's interesting to see how the code runs in these different contexts. It's a "brute-force" ray tracer, and there's some obvious optimizations that aren't in place that would make it a lot faster. Also, ShaderGen doesn't yet support some code patterns that could have made the CPU version quite a bit faster, like readonly structs and "in" parameters.

### Book scene: 100 spheres, 1280x720, 4 samples per pixel

Windows 10, Nvidia GTX 770, Core i7-4770K

* Vulkan: **96** million rays / sec
* Direct3D 11: **98** million rays / sec
* OpenGL: **85** million rays / sec
* CPU: **2** million rays / sec

macOS 10.13, Intel Iris Plus 640, Intel Core i5 2.3 GHz

* Metal: **58** million rays / sec
* CPU: **1.15** million rays / sec

### ToyPathTracer scene: 46 spheres, 1280x720, 4 samples per pixel

Windows 10, Nvidia GTX 770, Core i7-4770K

* Vulkan: **196** million rays / sec
* Direct3D 11: **182** million rays / sec
* OpenGL: **165** million rays / sec
* CPU: **3.5** million rays / sec

macOS 10.13, Intel Iris Plus 640, Intel Core i5 2.3 GHz

* Metal: **118** million rays / sec
* CPU: **2.4** million rays / sec

Each graphics API on Windows seems to be in roughly the same ballpark, with OpenGL underperforming a little bit, as expected. All versions are using 100% of my GPU or CPU. I was actually surprised at how well the Metal version runs, since my MacBook has a pretty weak GPU.

## Limitations

One of my goals was "use as much of the same code as possible on the CPU and GPU". In any ray tracer, you need to loop over multiple collections of structures: shapes, materials, lights, etc. In my code, these are stored in Veldrid "StructuredBuffer" objects. These correspond to StructuredBuffers in HLSL, storage buffer blocks in GLSL, and buffer-backed "constant pointers" in Metal. All of these shader concepts are roughly equivalent and work well for fine-grained methods operating on individual elements. However, there's no way to pass one of these buffer blocks around as a method parameter in GLSL, unless it is "fixed-size". This doesn't work so well for my ray tracer, unless I lock the size in at compile time. I'd really like to be able to resize these buffers and pass them around freely to different chunks of tracing logic in individual functions, but GLSL doesn't support it.

What this means is that I wasn't able to share _all_ of the tracing logic. Any methods that access the scene data need to be duplicated between my CPU and GPU code, at least until I figure out a clever solution to the above. Luckily, in my current version, the only method that this affects is the top-level "Color" method. That's the one that actually loops over all of the spheres and determines which one intersects with the current ray. Once the sphere and its "Material" are identified, the remaining logic doesn't need to worry about any of the other objects in the scene. I have two versions of this method which are roughly identical. However, things would get ugly if I wanted to add a collection of lights or light-emitting spheres, for example. Even after I know which object is hit, I would then still need to loop over the collection of lights (or emissive objects) to determine how each affects the sphere. I'd have to pull out even more code into GPU- and CPU-specific chunks.

The "OpenGL ES flavor" of GLSL has another quirky limitation -- you can't read and write to a Texture unless it has a single-channel pixel format. Perhaps this limitation could be worked around, but for the time being my shaders don't work on OpenGL ES.

## Practicality

So how useful would this be in an actual game? Well, I'm not sure, but the possibilities are interesting. For most problems utilizing a compute shader, a CPU fallback won't be fast enough. Algorithms designed for a GPU are also likely to be different from the optimal algorithm for a CPU, at least for more complex problems. For lighter-weight applications, like a very simple particle system, maybe it would work well enough on both sides. For systems that don't support compute shaders (OpenGL on macOS, for example), you could fall back to updating your particle buffers on the CPU, with the same code powering both versions. If your particles aren't doing anything particularly taxing, then the performance could be acceptable, and you would no longer need to maintain two versions of the same particle system code.

From another angle, it's a lot easier to debug regular C# than it is to debug anything running on your GPU. Having the option to debug problematic areas from the regular VS debugger could be valuable in itself.

## Related

* Most of the code in my project is based on [Ray Tracing in One Weekend](http://in1weekend.blogspot.com/2016/01/ray-tracing-in-one-weekend.html), which is a great little intro book on ray tracing.

* Aras Pranckevičius has a fun series on ray tracing [here](https://github.com/aras-p/ToyPathTracer). In it, he has built a C# tracer which is much more optimized than mine, as well as various versions in other languages.
