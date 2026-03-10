---
layout: post
title: "Plane Color Pipeline, CSC, 3D LUT, and KWin"
date: 2026-03-10
---

A wild blog appears...

## The Plane Color Pipeline API and KWin

A couple months ago the [DRM/KMS Plane Color Pipeline API](https://patchwork.freedesktop.org/series/157608/) was merged after more than 2 years of work and deep discussions. Many people worked on it and it's nice to see it upstream. KWin and other compositors implemented support for it. I'll mainly focus on kwin here because that's what I use regularly and what I am most familiar with. I will also focus on AMD HW because that's what I'm working on.

On AMD HW with a kernel that includes the new Color Pipeline API KWin enables HW composition for surfaces that update more than 20 times per second on a single enabled display. It needs a few other things to match as well. In particular, this means that running `mpv` with the default backend (`--vo=gpu`) will use HW composition for `mpv`'s video surface with the rest of the desktop. The easiest way to observe this is with [UMR](https://gitlab.freedesktop.org/tomstdenis/umr) by running it in `--gui` mode and looking at the `KMS` tab.

![UMR's KMS pane showing 4 planes](/assets/images/Screenshot_20260303_163142.png)

In my examples UMR also gets HW composed, so this shows 4 planes:
- 1920x1200 desktop plane - XR24
- 720p video plane - AB48
- 720p UMR plane  - XR24
- 256x256 cursor plane - AR24

The mpv framebuffer shows up as an AB48 buffer, not NV12. This is because the `--vo=gpu` backend in `mpv` performs any required color-space conversation, scaling, and tone-mapping, and then offers up a 16-bpc buffer to the Wayland compositor, which `kwin` passes to the DRM/KMS driver.

## NV12/P010 Scanout

We can tell `mpv` to pass the raw YUV buffer (NV12 or P010) to kwin by passing using the `--vo=dmabuf-wayland` backend. This tells `mpv` to simply decode the video stream but leave the buffer alone. It then passes the buffer information to `kwin` via the Wayland [`color-management`](https://wayland.app/protocols/color-management-v1) and [`color-representation`](https://wayland.app/protocols/color-representation-v1) protocol extensions.

When we do this we don't see a HW-composed plane in `umr`. `KWin` color-space converts, scales, tone-maps, and composes the plane via OpenGL. It can't offload it to display HW because the DRM color pipeline API doesn't yet support color-space conversion (CSC). The `drm_plane` does have [`COLOR_RANGE`](https://drmdb.emersion.fr/properties/4008636142/COLOR_RANGE) and [`COLOR_ENCODING`](https://drmdb.emersion.fr/properties/4008636142/COLOR_ENCODING) properties to specify CSC, but they are deprecated with the color pipeline API.

So I went and implemented a CSC `drm_colorop`, added IGT tests and added support for it in `kwin`.

[Kernel Patches](https://gitlab.freedesktop.org/hwentland/linux/-/commits/csc-colorop)

[IGT Patches](https://gitlab.freedesktop.org/hwentland/igt-gpu-tools/-/commits/csc-colorop)

[KWin Patches](https://invent.kde.org/hwentlan/kwin/-/tree/csc)

With this new CSC colorop we now see an NV12 buffer for our SDR video (I'm using a 1080p60 Big Buck Bunny clip).

![UMR's KMS pane showing 4 planes, one is NV12](/assets/images/Screenshot_20260303_170025.png)

## Banding and 3DLUTs

Unfortunately we see some banding during the HW composed Big Buck Bunny playback:

![Big Buck Bunny HW composed with banding](/assets/images/bbb_hw_composed.jpg)

This is the SW composed version, showing no problems:

![Big Buck Bunny SW composed](/assets/images/bbb_sw_composed.jpg)

I haven't yet debugged the banding. It seems to happen with one of the 1D LUTs.

But the AMD HW also has a 3D LUT and `kwin` lets us sample its entire internal color pipeline, so we can simply sample it at our 3DLUT coordinates and program it to HW. This allows us to represent any complex color pipeline with a single 3D LUT operation. The result is this.

![Big Buck Bunny HW composed with 3DLUT](/assets/images/bbb_3dlut_hw.jpg)

[kwin branch](https://invent.kde.org/hwentlan/kwin/-/tree/csc-3dlut)

## HDR Video

In order to compose HDR content `kwin` creates a tone-mapper. By packing the entire color pipeline into a 3D LUT we don't need to worry about it and get support for HW composition of HDR content for free.

![UMR's KMS pane showing 4 planes, one is P010](/assets/images/Screenshot_20260304_105706.png)

**Note:** AMD's 3D LUT uses 17 entries per dimension and interpolates tetrahedrally. This will give good results when applied in non-linear luminance space. KWin blends in non-linear space, and the input buffer is non-linear, so it works well here. While this gives good results you might still observe minor differences, especially in brighter areas of the image. This can be observed when toggling between HW and SW composition in certain scenes.

## Scaling

Because AMD's DCN HW uses a multi-tap scaler filter but kwin's SW composition uses `GL_LINEAR` there are differences in scaling. It's apparent when doing 4-to-1 downscaling, such as when scaling this 4k (HDR) video down to 720p.

SW composed:

![SW composed 4-to-1 downscaled](/assets/images/hdr_sw_composed.jpg)

HW composed:

![HW composed 4-to-1 downscaled](/assets/images/hdr_hw_composed.jpg)

The former has stronger aliasing. The latter looks softer and more natural, in my opinion. The difference becomes much less pronounced when the image is downscaled less, e.g., from 4k to 1080p.

At this point there is no good API to help align the two. GL seems to only have [`GL_NEAREST` and `GL_LINEAR`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glTexParameter.xhtml) when not dealing with mipmaps. DRM/KMS provides "Default" and "Nearest Neighbor" via the [`SCALING_FILTER`](https://drmdb.emersion.fr/properties/4008636142/SCALING_FILTER) property. While this allows us to use a nearest neighbor filter in both cases that's undesirable since it'll make the image worse in both cases.

## Seeing is believing

When I started this work I asked myself: How do I see whether a surface is a candidate for offloading? How do I see which actual surface is being offloaded?

To solve this I worked with Claude to create a plugin that marks surfaces and their offload status:

![showoffload effect in kwin debug console](/assets/images/image_0007.jpg)

It's quite useful to see immediately which surfaces are candidates, and why they might fail to offload.

I've also added a new tab to dynamically toggle HW composition and and off. The screenshot shows toggles for 3D LUT and tone-mapping and while we can add those as well they don't always take effect as expected, so I left them out of the branch that's linked below. But the ability to toggle HW composition is quite powerful when debugging HW composition issues.

![Offload Settings tab in kwin debug console](/assets/images/image_0008.jpg)

[kwin branch](https://invent.kde.org/hwentlan/kwin/-/tree/showoffload)

[kwin branch on top of csc-3dlut branch](https://invent.kde.org/hwentlan/kwin/-/tree/csc-3dlut-showoffload)

## UMR DCN tab

DCN HW programming can be logged via the `amdgpu_dm_dtn_log` debug log debugfs. But that log is quite extensive. It can be useful to show this in UMR with auto-update functionality to see programmed settings immediately.

![UMR DCN Tab](/assets/images/Screenshot_20260304_121331.png)

The code still needs a fair bit of work as this was the first time I used Claude for something extensive like this. I plan to post it eventually.

## A Brief Note on LLMs

I used Claude Sonnet extensively for this work (it basically wrote all the code), so I thought it prudent to leave a couple thoughts on it.

LLMs are large language models, not actual artificial intelligence. They're language models, and are designed to work well with language. They're large and can hold much more context than humans. Use them in ways that uses their strength. I've found value in understanding complex code-bases, and creating code that fits within those code-bases.

Don't stop owning your code. Even if it's produced by an LLM, take pride and ownership. This means, review what you get from an LLM. Be active in steering it. Don't throw trash at maintainers. Your name and reputation are on the line.

## Next Steps

I'll be working with relevant communities and maintainers to attempt to upstream these things.

The CSC colorop is probably in a good shape.

The KWin code requires feedback from maintainers. I expect it will need more work.

This also needs more testing. At times I see the 3D LUT fail to apply. I'm not sure whether this is a problem with my kwin code or amdgpu.

I don't see offload candidate surfaces from many applications where I'd expect to see them. This needs further analysis. For one, I'm unsure what happens with games. The other thing is Youtube in Firefox, which fails to present the video as an offload surface. Some other videos work fine in Firefox, in particular local video playback.

*sneaky edit*: and power measurements, of course, since that's the entire reason for this.