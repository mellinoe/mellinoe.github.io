---
layout: post
title:  "Overhauling ImGui.NET"
date:   2018-10-11 18:00:00 -0700
categories: graphics
---

I'm happy to finally release the new, overhauled version of [ImGui.NET](https://github.com/mellinoe/imgui.net). The library has been re-built from the ground up, utilizing the new auto-generated [cimgui](https://github.com/cimgui/cimgui) library and its associated tools. Previously, updating ImGui.NET was a very time-consuming and elaborate process, done by hand, and it often lagged behind official Dear ImGui releases. Instead of continuing that, I've implemented a code generator which processes cimgui's pre-parsed data files and spits out a bunch of C# code automatically. There are a lot of benefits to this new approach, and it will allow me to keep the library up to date much more easily and painlessly. I've also taken this opportunity to improve the usability of the library a great deal, and to line up the C# interface and versioning more closely with C++.

## Layers and Design

As before, there are two layers to ImGui.NET: the raw, unsafe native layer (`ImGuiNative`), and the safe, C#-friendly layer (`ImGui`). Previously, a very common problem was some functionality being available to the low-level layer, but not to the friendly high-level layer. Since both layers are automatically generated now, there are no gaps between the two. Additionally, new features added to Dear ImGui will automatically surface in both places in ImGui.NET. In most cases, users of ImGui.NET should never need to touch unsafe code, and the high-level `ImGui` class will suffice. For advanced scenarios, low-level access may still be more convenient, and the option to drop down to `ImGuiNative` remains.

It is also painless to utilize the auto-generation machinery to create a different version of ImGui.NET for an experimental branch of Dear ImGui -- for example, [the docking branch](https://github.com/ocornut/imgui/issues/2109). Pointing ImGui.NET's code generator at the processed output from that branch will give you a fully-usable library exposing all of the new functions and types, including safe wrappers.

## Safety

Some care has been taken to automatically generated safe wrapper code for the library. In many places, Dear ImGui expects that you will interact with it through various pointers to structures. In order to simplify and protect these patterns in C#, I've introduced a number of "Ptr"-suffixed structures, each of which represent a specific typed pointer. `ImGui.GetIO`, for example, now returns an `ImGuiIOPtr`, which is a safe struct wrapper over the native `ImGuiIO*` that the function returns. It provides safe access to all of that type's members and functions, and requires no unsafe code to use. Unlike the previous version, it allocates no garbage-collected memory at all, and does not make any copies when fields are accessed, utilizing managed references instead. These "Ptr" structures should be viewed as a thin wrapper over a pointer, and are implicitly convertible back and forth with those native types.

Previously-opaque structures like `ImVector<T>` are also much more friendly to C# than they were before, and give you safe, copy-free access to individual elements inside the vector.

## Breaking Changes

ImGui.NET 1.65.0 introduces some breaking changes if you are upgrading from 0.4.7. Many structure, enum, method, and parameters have been renamed so that they are identical to their C++ counterparts. These should be viewed as one-time breaks; future versions of ImGui.NET will continue to match the C++ naming as closely as possible.

As a result of the bump to Dear ImGui 1.65, there are also several functions that have been removed or deprecated. This category of breaking change is documented well in Dear ImGui's release notes.

## Versioning

Going forward, ImGui.NET will use a versioning scheme that more closely lines up with native Dear ImGui. To start, this initial NuGet package will be versioned 1.65.0, corresponding to [v1.65 of Dear ImGui](https://github.com/ocornut/imgui/releases/tag/v1.65).

Note that previous releases of ImGui.NET were versioned from 0.1.0+. Version 1.65.0 contains breaking changes from 0.4.7 (the last release of the previous series), and will not necessarily maintain compatibility between updates. Going forward, I intend to inherit the deprecations and removals from Dear ImGui itself, rather than maintain strict binary compatibility between versions of ImGui.NET.
