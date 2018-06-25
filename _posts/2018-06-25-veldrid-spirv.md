---
layout: post
title:  "Veldrid Support for SPIR-V Shaders"
date:   2018-06-25 12:00:00 -0700
categories: graphics
---

[Veldrid](https://mellinoe.github.io/veldrid-docs/) is a low-level graphics library written in C# that allows you to create GPU-accelerated applications targeting wide variety of platforms, without dealing with platform-specific graphics APIs. Although Veldrid aims to be as portable as possible, one pain point has always been shader code, which differs between platforms. Writing your shaders multiple times is error-prone, limits your portability, and can quickly become a big hassle. Other portable graphics libraries and game engines take different approaches to tackling this problem. Many libraries support a single "official" shading language (often HLSL or a variant, but occasionally a custom shading language) and translate it into a number of shading languages, depending on the graphics APIs being targeted. My [ShaderGen](https://github.com/mellinoe/ShaderGen/) project can be seen as one such custom shading language.

A very promising new option for portable shaders is the [Khronos Group's SPIR-V language](https://www.khronos.org/registry/spir-v/).

> SPIR-V is a binary intermediate language for representing graphical-shader stages and compute kernels.

SPIR-V is a simple bytecode language for graphics and compute that can be targeted from several languages (including GLSL and HLSL), with more in development. There are also a variety of post-processing tools, optimizers, and debugging utilities available for it. Overall, it is a well-supported language with a very healthy and productive ecosystem developing around it.

Today, I'm releasing a Veldrid extension library called Veldrid.SPIRV, which provides support for loading SPIR-V shaders on all of Veldrid's supported backends. Veldrid.SPIRV is built on top of [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross), a library for translating SPIR-V bytecode into several high-level shading languages. With Veldrid.SPIRV, you can write your shaders in any language targeting SPIR-V and use them easily with Veldrid.

Veldrid.SPIRV is available on NuGet.org: [![NuGet](https://img.shields.io/nuget/v/Veldrid.SPIRV.svg)](https://www.nuget.org/packages/Veldrid.SPIRV)

## Veldrid Support

Veldrid.SPIRV exposes several extension methods on ResourceFactory which allow you to create Shaders from SPIR-V bytecode. In order to create a Veldrid Pipeline from SPIR-V, you need to provide the bytecode for all shader stages being used. This is because Veldrid.SPIRV needs to be aware of the full set of shader resources (Buffers, Textures, and Samplers) used by a Pipeline in order to assign the correct "slots" for each resource.

Based on the type of ResourceFactory passed in, Veldrid.SPIRV will figure out which target language is needed, and will automatically generate the appropriate shader code and compile it for you. Most people will just need these two extension methods:

{% highlight C# %}
// Create a set of Shaders usable in a graphics Pipeline.
public static Shader[] CreateFromSpirv(
    this ResourceFactory factory,
    ShaderDescription vertexShaderDescription,
    ShaderDescription fragmentShaderDescription,
    CrossCompileOptions options);

// Create a Shader usable in a compute Pipeline.
public static Shader CreateFromSpirv(
    this ResourceFactory factory,
    ShaderDescription computeShaderDescription,
    CrossCompileOptions options);
{% endhighlight %}

## Specialization Constants

SPIR-V and Vulkan have support for "Specialization Constants", which are an interesting feature providing greater flexibility to shaders. Specialization Constants are constants within a shader program that can be substituted with new values when a Pipeline is created. Likewise, Metal shaders can contain "function constants", which serve roughly the same purpose. I've added support for both of these concepts to Veldrid through a new SpecializationConstant type. When constructing a new Pipeline, you can provide an array of SpecializationConstants which will influence the behavior of your shaders.

Here is an example fragment shader which contains several Specialization Constants. When you create one or more Pipelines with this shader, you can override these values without generating new SPIR-V bytecode or re-compiling your shader at all.

{% highlight GLSL %}
#version 450

layout (set = 0, binding = 0) uniform texture2D Tex;
layout (set = 0, binding = 1) uniform sampler Smp;

layout (constant_id = 0) const bool UseTexture = false;
layout (constant_id = 1) const bool FlipTexture = false;

layout (constant_id = 2) const float RedChannel = 0.1f;
layout (constant_id = 3) const float GreenChannel = 0.1f;
layout (constant_id = 4) const float BlueChannel = 0.1f;

layout (location = 0) in vec2 fsin_TexCoords;
layout (location = 0) out vec4 fsout_Color0;

void main()
{
    if (UseTexture)
    {
        vec2 uv = fsin_TexCoords;
        if (FlipTexture) { uv.y = 1 - uv.y; }
        fsout_Color0 = texture(sampler2D(Tex, Smp), uv);
    }
    else
    {
        fsout_Color0 = vec4(RedChannel, GreenChannel, BlueChannel, 1.0);
    }
}
{% endhighlight %}

[_Click here to see the compiled SPIR-V bytecode for this shader._](http://shader-playground.timjones.io/26c9a6d6acbb7013abf960955eb4e465)

If you want to enable the "UseTexture" and "FlipTexture" flags and substitute different color channels in, you can write code like the following:

{% highlight C# %}
ShaderSetDescription shaderSetDesc = new ShaderSetDescription(
    vertexLayoutDescriptions,
    new Shader[] { vertexShader, fragmentShader },
    new SpecializationConstant[]
    {
        new SpecializationConstant(0, true), // UseTexture = true
        new SpecializationConstant(1, true), // FlipTexture = true
        new SpecializationConstant(2, 0.95f), // RedChannel = 0.95f
        new SpecializationConstant(3, 0.0f), // GreenChannel = 0f
        new SpecializationConstant(4, 0.5f), // BlueChannel = 0.5f
    });
{% endhighlight %}

If this ShaderSetDescription is used to create a Vulkan or Metal Pipeline, then the SpecializationConstant values listed in the array will replace the pre-defined constants in the shader. It is therefore trivial to create another Pipeline which substitutes different constant values by passing in a different array. SPIR-V Specialization Constants always contain default values, so providing SpecializationConstants is optional. You may override a subset (or none) of the Specialization Constants defined in the shader.

Unfortunately, HLSL and OpenGL-style GLSL do not support any kind of specialization constants. All constant values used in the shader must be baked into the shader itself when it is compiled. However, Veldrid.SPIRV allows you to substitute new values in for each Specialization Constant before the shader is translated from SPIR-V into the target language. In practice, this allows you to use SPIR-V shaders with all of Veldrid's backends and still take advantage of the flexibility of Specialization Constants. If you want to produce GLSL or HLSL (bytecode) at build-time for your application (e.g. to improve load time), you will need to manage the "specialization matrix" yourself.

## Extras

Veldrid.SPIRV also supports compiling GLSL code into SPIR-V, by wrapping Google's [shaderc](https://github.com/google/shaderc) compiler library. This gives you even more flexibility at runtime with your shaders. It's possible to defer all shader compilation til runtime and still maintain full portability with Veldrid.

## Limitations

Veldrid.SPIRV relies on a native shared library (libveldrid-spirv), currently packaged for Windows, Linux, and macOS. This is a fairly small component with few dependencies, and is not difficult to compile for additional platforms.