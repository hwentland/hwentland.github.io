---
title: 2025 Display Next Hackfest

---

# Day 1

## ACS - MPO, HDR, and Freesync

* ACS (amdgpu composition stack) is a fork of weston -- staging area for developing full-stack features on AMD
* Supported and open source: VRR, MPO underlays, multiseats...

VRR/Freesync: only support fullscreen
e transfer function -> Dolby PQ
MPO (Multi plane overlay): already existing in mainline Weston, but limited/restrictive. AMD created MPO using underlays. As Primary is always on the back and overlay on top of it, to use the primary plane "on top" (sometimes required for hardware acceleration), you have to create "holes" in the overlay planes. 
=> allows to not use the GPU for composition (hardware decoding direclty output to primary plane, and compositor offload composition to display pipeline using this "hole")

- pq: VRR and underlay with rendered hole support should be present in Weston upstream main branch.

color managment/wide gamut: Issue is that each application can use a differnt color space, format, luminance, depth... what should the compositor chose?

- pq: The compositor should convert and map according to the display capabilities rather than any single source in a multi-source situation.

https://gitlab.com/acs-wayland
https://gitlab.com/acs-wayland/weston/-/wikis/home

## Commit Failure Feedback

Currently: no information about a commit failure. Only current solution is to try many different configurations, it can take multiple seconds.

The idea is to get some kind of basic feedback: is it a bandwidth issue, a overlay mis-usage...

Anything can be good.

proposition: drm_mode_atomic, there is reserved u64 that can be used as a feedback_ptr

=> we could attach a string, but we should pass the string size (ioctl, CAP)
=> we could attach a second drm_mode_atomic_ext that contains multiple informations

Current drm_mode_atomic_ioctl is okay => it already checks if this reserved field is empty

we could also use CLIENT_CAP to pass an other pointer that the kernel can fill

Another option is to use something similar to OUT_FENCE_PTR attached per-crtc

dmesg is not enough for compositor, they need information to fix the issue: is it a plane config issue, bandwidth...
There are different "scope" of feedback (local to a plane, local to a MST connector, global to the whole pipeline...)

AMD example:
- memory bandwidth is global
- connector bandwith is "local" per connector

Adding a new property on each drm object to tell issue per specific object (like drm_fence_ptr)

We need two different kind of informations: "generic" (like plane 1 failed) and "hardware" (like this specific gpu does not support this plane position because it is not an odd number)

we cant rely on logs because:
- not cleary synchronized with actual atomics
- a lot of atomic commits are expected to fail, this can destroy performance

Multiple problems? Anyway most of the driver does early return, so you alays get the first one. This means that you will be able to fix only one issue at a time.

We may need to attach the enum value to a drm object ID (connector, crtc...)

Expose bandwith values (per plane, per color format...)? Bad idea: too much information, depends a lot on hardware, can not even be a simple addition...

Need a way to help compositor to do the "good" thing:
=> maybe we could report a suggested strategy: REDUCE_PLANES vs SCANOUT_BANDWITH

At least a list of possible strategies can be useful per driver, the important thing for compositor is to know if their attempt may solve the issue or not. Some documentation can be enough.

Use flags for report multiple issues? it can be used to says "planes + colorop is not possible", but not to say "plane 4 + colorop yuv->rgb is not possible"

Want a simple extendable error reporting framework to start that covers:
1. A global enum-like feedback which is easily actionable by the compositor
2. Per DRM-object property for fine-grained feedback
3. A string feedback which can be used for logging/bug reports


### Conclusion: 

Start with simple stuff (but keep it expandable for the future)
- an enum to show what is the issue
- a string/log to display "verbose" information about the failure (should not be used as UAPI, only for debugging purpose)
- an array of KMS object IDs participating in the failure (optional to start)

enum values:
- undefined
- scanout bandwidth
- connector bandwidth
- memory domain
- scanin bandwidth
- spec violation (out of range, immutable, -EINVAL)
 
**AP**: we need someone to submit at least an RFC/draft so it can be used a bit/each driver can add their own failures (looking for volunteers)

## Async Pageflip Failures

Issue: some async pageflip can fail "temporarly", for example memory domain modification, that may require a sync pageflip, but the following pageflips can be async.

**AP**:
- once we have better commit failure feedback: add more information in the feedback to tell if this a transient error or not
- Add more IGT tests to properly test the userspace expectation (transient error should disapear) => Melissa will try to work on this

## Backlight Improvements

AMD improvements:
- fix default brightness, add default for AC/DC levels, reset behavior
- Custom backlight curves: allows firmware to have some curve to control more reliably the backlight depending on the specific hardware
- Export bigger range for backlight
- OLED 0 bright is not power off

Power saving policy
- waiting for compositor support (kwin is working on it)
=> For PSR the compositor may need more control on PSR (panel self refresh) because it knows better when it should be enabled/disabled. It is not possible to decrease the timout in the kernel, because the compositor can start to schedule slowly, but the kernel does not know it and if PSR is enabled, it is long to disable it (can be 3 frames)
=> ABM level does not only affect colors, but also brightness, the compositor need to know that it does both. Example usage: outdoor usage, you need to have maximum brightness, but if you enable ABM it reduce the max brightness

2022 recap - see slides

Windows has an API for nits reporting, linux should have it, compositors agrees!

Initial work that should be done for nit information:
- not doing DDC
- having a way to expose nit level, with some extra metadata (max, scale, linear...)

0 means minimum brightness, not off.
We need to advertise the minimum nits value, and compositor needs to know if 0 is power off or not.

Avoid database in the kernel: they can't be updated easily.

Create a new uAPI, the idea that we should start with new devices, and change the old devices after. Needed to avoid regression before more testings
=> in any case the old api will be available and not change the behavior, so it should not be a problem to add the new uAPI and remove the old after a lot of testing.

We don't care about hotkey handling, everything is managed from compositor.

Not to be in the first revision: it could be nice if backlight can be controlled by atomic uapi. HDR mode on KDE could use it to make some ramping of backlight (you need to increase the backlight to make more HDR headroom on demand, but you don't want to make the user blind, so you need to "Synchronize" the backlight modification with the pixel changes in commits).
It would be amazing if the firmware does not do anything "magic".

Mux API did not move, Nvidia took over but no advancement.

**AP**: Is anyone working this? Hans is already very busy.
Mario will try to do a proposal.

**Sean Joined and further discussion**
Sean is supportive of connector based properties.

First go may not support atomic handling.

Sean: Could potentially do sysfs files instead.
Mario: Better to go for KMS properties to avoid SUID.


## Desktop Testing Framework

Windows have a certification test they have to measure the duration of a battery using a very specific bench.
They are doing a lot of automation "click on this button", "open this app"...
They do various uses cases and measure the battery usage.

Can we create a similar testing framework?

The only thing that went somewhere was MIR compositor test suit developed by Canonical: [WLCS](https://github.com/canonical/wlcs); wierd licensing issue, unappealing c++ codebase, requires a specific compositor support plugin.
Kinda outdated.

[Way-assay](https://gitlab.freedesktop.org/bwidawsk/way-assay) is a similar thing in Rust, they had gone to the point of having some tests, but abandoned now.
=> this was a good aproach

It could be nice to have tests that also allow to check if the compositor use the wayland api properly.

Each compositor need to have a common API to expose abstraction. Kwin and COSMIC said that a protocol or library interface that every compositor must implement would be problematic to define and maintain. A much better idea would be to have the test suite offer a library, which compositors can use to run common tests by implementing the necessary callbacks from the library.
=> This could help to test wayland protocol or compositor stuff, but not "system behavior" (session managment)

Harry would like to have a test for power managment, not compositor features, the goal is to emulate "user is using his laptop to look at video" and measure power consuption.
This needs an API to run the application, but the API is very similar that the "compositor test suite", so this could be nice to merge efforts.
It can also help to support more compositors in the same test suite.
The accessibility API could be used to do some automation.
The accessibility infrastructure will not help for positioning windows, pointer motion and clicks, it will only help for "content" (text, widget interactions).


Input portals?
OpenQA?

## Enable VRR

Sometimes enable VRR creates a black screen (screen reset) on the panel with cosmic. We don't expect this to happen with DP, but possibly with HDMI. Both AMD and Intel GPUs are believed to handle the seamless VRR enable.


## 30 Hz video on a VRR sink with 40 Hz - 60 Hz refresh range

Some frames are long, others short = judder.
Can the compositor compensate and double the refresh rate (30 -> 60) in this case?

We still need a mechanism to disable LFC (low-framerate compensation) in amdgpu. Ideally the compositor would also be able to provide a min/max refresh rate.


## VKMS

Links to pending VKMS work:
configfs series: https://lore.kernel.org/all/20250507135431.53907-1-jose.exposito89@gmail.com/
more formats series: https://lore.kernel.org/all/20250703-b4-new-color-formats-v7-0-15fd8fd2e15c@bootlin.com/
vkms-configfs-everything branch: https://github.com/Fomys/linux/tree/b4/vkms-dynamic-connectors

Slides: https://bootlin.com/pub/conferences/2025/display-next-hackfest/vkms.pdf

Mutter has a branch which uses the new API for existing VKMS tests.

AP: Use it for more tests in mutter

## DP Aux Improvements

https://github.com/fwupd/fwupd/pull/7007

fwupd probing can interfere with link training when it probes devices at boot. Doing a firmware update can also clash with kernel's AUX operation. Can we serialize the respective AUX sequences? This might be problematic with the DP specs requirements of transmitters to read DPCD within a short period of time after receiving a short pulse interrupt.

A way to deal with this is to let the kernel handle the sequences directly so it can deconflict internally. We would need one sysfs node that can provide device information for fwupd, so fwupd wouldn't need to issue AUX probles directly. We'd also need a mechanism for fwupd to pass a payload to the kernel driver that it can then issue to the MST dock/monitor.

We would want to report "not supported" on nodes that fwupd shouldn't need to probe.

* Kernel should have a database of known offsets to read opportunistically the input.

* Userspace could pass an offset to get a value safely as well.

# Day 2


## Color & HDR - Status

Wayland protocol is merged! compositors and apps started to use it, it is actually very nice and better than windows.
[Display profiling interfaces](https://gitlab.freedesktop.org/pq/color-and-hdr/-/issues/27) are still missing, and might be done with portals.

Color representention procol was merged for video formats, it was a lot of work on compositor side.

For kwin HDR is working for quite a while, switching from absolute nit to luminance, but it works better for compositors and apps
extended dynanic range implemented for laptops, so you can "emulate" some kind of HDR on SDR display

Mutter also have HDR support
It is exposed by default
They have the reference white form the begining, not a lot of issue, some applications already use it.
Still some work to do on color managment, mutter have to improve on the transient functions, primary, exposinging color managed mode which are not HDR
Some patches are ready for ICC profile support for the protocol side
Currently the backend is not doing a lot of color managment, only HDR
Mutter worked on backlight handling to properly manage the reference white level, to expose it transparently in a single slide in the UI

About the wayland protocol color managment: Pekka still has one MR open. https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/415
We likely need to adjust the default luminances for a few transfer characteristics as well.

Cosmic: No color management support yet, but planned for a future release.
A lot of things were worked on in background, but nothing user-visible until after 1.0.

Weston has supported ICC-to-ICC color transformations for a long time, by assuming all app content is sRGB. Color-management protocol implementation is ready, meaning apps can set ICC profiles. Support for handling parametric profiles is WIP. Colorimetric rendering intents have been implemented for parametric-to-parametric color transformations in a MR. Color transformations between ICC and parametric profiles for colorimetric rendering intents are also in a MR. Off-loading pre- and post-blending color operations to KMS are WIP in MRs. Pre-blend off-loading targets the new color pipeline KMS UAPI.

wlroots side: some basic HDR stuff clipping. 

Applications: gamescope, blender, firefox WIP, ...

Gamescope wants to migrate the driver specific properties to generic KMS color API, but no time enough for it right now. Melissa is trying to push this a bit from Simon Ser draft, but needs to figure out how to hack libliftoff.
- David shared an already WIP on the libliftoff side: https://gitlab.freedesktop.org/emersion/libliftoff/-/issues/85

Simon asked about creating a MR to add KMS color API support to `drm_info`: https://gitlab.freedesktop.org/mwen/drm_info/-/commits/kms_color?ref_type=heads

IGT tests: 
Alex H. confirmed there is a series ready for it.
Harry mentioned some limitations on 3D LUT testing that depends on ICC profile (in Weston?).

Intel: they have extended current proposal to fit their hw but there are still some open question to discuss with userspace developers about what is acceptable.

Chromium: Sasha is actually trying to implement color pipeline but there are still some conflicts with Intel implementation... from their use cases the proposal fits well and is in a good shape.

GTK side: New video player with (planned?) gstreamer color pipeline support

Gstreamer wayland side: we didn't have fully HDR metadata yet in weston...?

Xaver mentioned tone mapping usages but not on display because the behavior is still quite weird.

For some screens, if you report back the maximum brightness reported by the display it will do something dumb, so some quirks are necessary.

Libplacebo supports Dolby Vision in some form, using https://github.com/quietvoid/dovi_tool/tree/main/dolby_vision to get the metadata that's used in a shader.

**AP:**
- Melissa to create MR for COLOR_PIPELINE support to `drm_info`

## Color & HDR - Q&A

Answering Questions:
Charles Poynton: https://www.poynton.ca/
Keith Lee

We don't actually know what's in the definition on HDMI source based tone mapping. We are struggling on support because not all information from HDMI (Forum) is open.


**Q:** Why was HDR developed? What desires needed fulfilling?
Make pictures sparkle! Remove the limitation of classic HD, the goal is to have brighter color than classic white paper. For example reflection on glass/water or direct light source, that can be brighter than the "paper white". It can refer both to "classic" video but also for artists who wants to use HDR capabilities.

Pictures and movies tell a story, they should not capture the viewer's attention by shiny stuff. Huge number of screw-ups in the last 10y! Manufacturers want to tell the viewer: "yeah we're using HDR now!!!1"

**Q:** Why did reference white specification happen so late in the HDR history?

Video formats and stills had somewhat implicitly defined reference white levels. 
Some images/stills don't have reference white on the picture (back alley with no direct light, bad weather...)

BBC with HLG has OOTF (mastering display vs consumer display).
Artificially increase the chroma to compensate for the absolute luminance loss.

Display-referred reference white level?

Tweets, paraphrasing:

- Scene-referred: when there is a mathematical description of the mapping from radiometric light in front of camera to the image data + relative luminance.
- Display-referred: when there is a mathematical description of the mapping from image data to the radiometric emissions on a display.

Keith mentioned PQ has its own problems, for example, it allocates many values that cannot be reproduced by any display.
Also mentioned that Apple laptops have sensors that capture environment light levels to adjust how to display HDR.

**Q:** Counter-movement against HDR, dynamic range doesn't give you anything over HDR, what are the differences between SDR/HDR?

Steve Yedlin's Debunking HDR talk: https://www.yedlin.net/DebunkingHDR/
Previous one, display prep: https://www.yedlin.net/DisplayPrepDemo/

**Q:** How all this process applies on image recognition/machine vision? 

In most machine vision applications you are only interested in the diffuse reflection of the object in front of the camera. Not aware of any use of specular reflection, and you rarely want to look at the source of light. Most of the stuff just want to crop signals at reference white.
Special case: automotive, you want to see the trafic light (so they are light source, you don't want to crop the range here)

On camera, they apply a kind of correction, which is not linear, but tries to reproduce the human vision. Take care about "linear" term, this is rarely used to say that the color transformation from "real life" to "image" is "linear", it most often refer to a linear transformation on an already transformed image from the camera.
If a vendor tell you that something is linear, they often just mean "Gamma 2.2"

**Q:** Advantages / disavantages of PQ and HLG? When to use it, not use it?

For google HLG and PQ is the same, they render HLG for a 1000 nit display and use their "normal" PQ transformation to generate the final image.

Looking at OOTF formula: ties together peak luminance and HDR reference luminance - not the case with PQ. W/ PQ: arbitrary ratio between mid-tones and highlights. Could we mix these two to get sth better?

Barten -> medical gray-scale transfer function -> Dolby PQ

Dolby PQ: for cinema (completely fine for TV as well)

- Completely display-referred. No cameras, no system gamma, etc. No OETF.
- They don't trust metadata (good on them!) → absolute luminance.
- Tailored for the human eye (hardcoded Barten formula with fixed parameters).

BBC HLG: for TV

- Screws up by focusing on OOTF
- As a relative luminance format, loses the picture's connection with absolute luminance levels which affect color perpception.

**Q:**: how to adapt content meant for a reference display to the actual viewing env?

What Apple does:

- Brightness sensor adaptation


**AP:** Update some of the color-and-hdr docs

## Color & HDR - DRM/KMS

Color API -- Intention is that kernel does exactly the mathematical operations that userspace prescribes.
- Kernel does not decide the pipeline to go from A to B.
- Userspace will construct the correct pipeline for its content.

IGT `kms_colorop` tests that driver applies the right color ops by comparing a cpu-generated "expected" output with the hardware generated "actual" output, obtained via connector writeback (allowing for some variance). Open for reviews!

Intel precision hints and segmented LUTs: https://patchwork.freedesktop.org/patch/662044/?series=129811&rev=5

KMS color API: ensure that COLOR_ENCODING and COLOR_RANGE are not exposed or implicitly applied in this first version at all.

AMD remap CRTC CTM to plane CTM for some hw versions, it was fixed on DCN3: https://lore.kernel.org/amd-gfx/20230721132431.692158-1-mwen@igalia.com/ but it's good to double check if the issue is present in newer versions, for example.

**APs**:

- Alex H. to ignore COLOR_ENCODING/COLOR_RANGE when color pipeline is enabled, and look into hiding these props.
- Alex H. to double check if CRTC CTM isn't remapped to pre-blending gamut_remap in any HW version that exposes plane COLOR PIPELINE.
- Intel to look at IGT patches.

## Color & HDR - Compositors & Applications

Long discussion about viewing environment adjustments.
-  Depending on the absolute luminance, some gamma adjustment is needed. What gamma is best for a given luminance is not completely figured out in industry or science.
- SMPTE ST 2094-50 is another adaptive gain curve
- The image data for sRGB shown at 320 cd/m² with gamma 2.2 in a bright environment is exactly the same data as BT.1886 shown at 100 cd/m² (gamma 2.4) in dim environment.
- 200 cd/m² level would probably decode that data with gamma 2.3

HDR test materials:
- No good HLG materials known.
- PQ can be good, just missing the reference luminance level which may vary.

Vulkan HDR API:
- Problem: Windows has "copy nits to screen" behavior on desktop monitors. On Wayland you get reference luminance matching instead.
- either have the driver tell the application about the reference luminance that the app has to adhere to > easier for games and drivers
- and/or add an extension for a fixed reference luminance (possibly per format) < easier for some other applications

## CI, DRM CI, IGT

DRM-CI presentation by Helen Koike

"CTS" for KMS drivers?
- https://www.mripard.dev/2025/01/17/kms-testing-improvements.html
- More generic IGT KMS tests
- List of expected successful/failing tests and flakes
- More unit tests
- Requirements and testing designed by userspace developers
- DRM-CI

Idea: use IGT to collect "low level tests" that came from specific issue (for exemple, just "plane rotation", or "color conversion", a kind of "DRM/KMS api validation"/"unit tests") and use compositor CI/tests for "complete" tests (multi plane usage, different "real life" scenario, a kind of "integration tests")

# Day 3

## Real-time Scheduling

### Page-flip Timeouts

When userspace schedules an atomic commit with the pageflip callback it can happen that the kernel never returns and times out and the pageflip kind of gets lost. In this case the compositor can't schedule another pageflip again because there's a pending pageflip and the screen will be completely frozen.

Kwin has seen a bunch of problems like this and kwin added some detection around this to point users at the bugtracker but fixing this generally is not going to work, currently.

When this happens the user has to reset the system (kill the power) to get their system back.

In the past when this happens the compositor would block and simply crash if the pageflip never returned. In this case drivers often recovered.

**Approach 1**:
- can the driver send a pageflip event even though it didn't happen so the compositor can continue
- maybe a new "page flip failed" event

**Approach 2**:
- driver detects the pageflip timeout happened
- could the driver send a wedge event so the compositor can unload and reload the driver?
- currently only cosmic handles reset of the primary GPU


`drm_atomic_helper_wait_for_flip_done` detects the flip_done timeout

At this point we should do something to handle this generally.

We could report a new page_flip_failed event to userspace to let userspace recover.

At this point DRM state and HW state could be out of sync so we would need a full modeset to recover.

Userspace could attempt recovery by issuing a full modeset.

Maybe the driver could also figure out why the timeout might have happened and recover appropriately, possibly by resetting display FW and IPs, possibly by resetting the entire GPU, possibly by doing a clean modeset.

The wedge event is an event to provide feedback to userspace about GPU resets. It includes severity, etc. One could add a flag for display reset there.

DRM wedged event:
- [Introduce DRM device wedged event](https://lore.kernel.org/dri-devel/20241128153707.1294347-1-raag.jadav@intel.com/)
- [drm: Create a task info option for wedge events](https://lore.kernel.org/dri-devel/20250617124949.2151549-1-andrealmeid@igalia.com/)

Harry said that display reset would be potentially an option; and then fallback to full GPU reset.

Alex suggested that there needs to be a helper to automatically complete any pending display events in the event of a display reset.

If restoring the state after a display reset no need to do a full modeset from compositor. The GPU driver can just reapply the atomic state.  Not the "best" experience for the user, but way better than current situation.

This needs to be documented so that compositors will know how to handle.

It would be very helpful to have a path to contrive failures (fake timeouts) so that compositors can test their recovery paths are working properly.

Recovery will be different for different vendors so there should be a new drm callback into the driver as `handle_timeout()`.
Drivers can then use implementation specific responses to that timeout.


### Scheduling Atomic Commits

*tricky topic*, compositors want lowest latency as possible without missing deadline for the next frame. Compositors estimate how long the various steps will take.  IE rendering, CPU scheduler delays for when waking up to do atomic commit and how long the commit itself takes.

How long it takes is problematic because they can measure how long the ioctl takes, but the hardware programming might not be done when it returns.
It might take more time to hit the next frame.  Compositor not knowing it missed the deadline it's useless.

Ideally want to know when actually complete to take this into account when scehduling events.  Right now kwin just assumes 1-2ms (somewhere in that range).  Everyone seems to have a different number. Ideally want to reduce latency or control how often frames are dropped.

Discussed last year, but it's a problem of *someone needs to do it*.

Michel did an experiment last week where he made a new event with a timestamp to show when it finished programming to the hardware.  It was useful as long as it managed to finish before the deadline.  Then the compositor actually gets the information for how much time it needed for the programming. However if it missesd the deadline it looked like the driver couldn't program the commit immediately; timestamps would be after the scanout of the next frame (probably overestimation of how much time the driver really needed).

Michel would like to see information about the actual deadline or at least how far away from the deadline they are. Would want to have the "delta" between the two.

There has to be one point in the driver where you check "can I program this now" or "do I have to wait". This is basically the deadline. At this point you can take the current time and calculate the latest possible time for programming the commit. If you already missed the deadline, you can create a delta of how far it was **with the opposite sign**.

Michel: Ideally you would have the **deadline** and **the time it takes to program**.

What does that mean?  Does it take color pipeline LUTs into account?  Yes; it should have it all and it may be variable.

Just like GPU rendering the compositor needs to look at timestamps and do the best they can.

We're not a real time system.  There is pre-emption, etc. We can report the deadline in hardware, but the programming time may be variable.

*Right now; compositor only knows you were off; but not by how much.*

Harry: Simplifying it; it's a **hardware done event**.  This is the time until the hardware will start scanning out.

Victoria: will this work in VRR?
Harry: it can still work for AMD.
For ARM it might not work 

Michal: Maybe it doesn't need to be an event but it could be a property?  Driver puts it in.

David: Why have userspace do the math?  Just calculate in the driver and put slack time, ie. the amount of time between programming the CRTC finishing and the vblank deadline.  The compositor can tune whether it commits earlier or later based on how much slack there was.  If the deadline is missed then the reported number is large (about 1 frame interval) because the commit goes in the next frame.  The compositor can easily identify this and knows it needs to commit earlier.  So there's just a single number and no need for multiple events or pinning down detailed commit/programming/vblank semantics.  For VRR this number is probably always reported as zero (assuming you're in the refresh-rate window and LFC is off).

**Conclusion:**
- report hw_done event up to userspace
- report time until HW latches with the event


## Per-CRTC Page-flip Event Requests

Problem: A separate page-flip event is sent for each CRTC included in an atomic commit.  Compositors need to anticipate which page-flip events they will receive.  But working out exactly which CRTCs are included in a commit is complicated, they can get pulled in unexpectedly.

Bonus problem: It's not allowed (an error is raised) if you request page-flip events while including a CRTC which is disabled and stays disabled.  Because that CRTC will never page-flip so you'd never receive the page-flip events.  This is the kernel trying to help avoid buggy userspace.

Overall everyone seems to agree \o/

- Add a CRTC property: PAGE_FLIP_EVENT, set to 1 if you want a page-flip event for that CRTC.  Needs to be explicitly set to 1 in every commit you want a page-flip event for.
- Make it now allowed to request page-flip events on a CRTC which is disabled and stays disabled.  The kernel should just immediately send a dummy page-flip event in this case.  This is easy for the kernel and avoids user-space having to carefully track this state.
- Be careful about rmfb() implicitly disabling some CRTCs (on amdgpu, because they can't allow an active CRTC with no primary plane) (but you probably aren't using rmfb at the same time as atomics / page-flip events)

## VRR for Desktop Use-Cases

- Enabling VRR on desktop can save power
- Fixed refresh rate (e.g. min multiple of video rate)

DRM Level:
- client cap that compositor can manage the VRR rate (min/max)
- upon seeing the cap driver should disable any LFC, ramping, etc
- new properties for min/max
- driver programs HW to keep VRR within the range

For video players and games it would be nice if they could specify a preferred refresh rate. This would need a wayland protocol.

Would it make sense for applications to also communicate a desired min/max rate for VRR? -> maximum yes, minimum doesn't have a use case?

## Open Session

### Slow Atomic Check

- "just" needs someone to profile and optimize whatever is possible on the kernel driver side
- compositors can try to work around the issue by only using overlay planes in mostly static situations

### Blitting Engine

- Some devices have powerful 2d hardware compositors, would be nice to use this for rendering instead of the GPU (lower power, more efficient, avoids conversion to/from tiled formats)
- These could be exposed via KMS as a writeback connector.  It is *possible* to build a compositor using this for rendering (David did this on rpi 5 as a proof-of-concept) but it's pretty awful, and confusing to use the same DRM device for rendering and scanout.
- Probably makes a lot more sense to expose them as a separate Vulkan queue and use that for rendering.
- When do you want to use a 2d compositor (or writeback connector) vs doing scanout composition with DRM?  Scanout-time composition uses less memory bandwidth for constantly-changing content (eg. video) but writeback-style composition is better for rarely changing content (eg. your desktop background and panel).

### Logical displays / Superframes

The compositor seems to be the right place for this

Splitting at a low level might be too inflexible:
https://github.com/swaywm/sway/issues/8645

### SW decoded video with HW composition

https://gitlab.freedesktop.org/drm/misc/kernel/-/commit/2271e0a20ef795838527815e057f5af206b69c87
https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/34303

Raspberry Pi 5 can do this with YUV420.  Uses the udmabuf kernel driver to import userspace buffers as dmabufs.  Relies on Pi5's DRM hardware having an IOMMU to scatter/gather non-contiguous buffers, not all hardware can do this.

For HDR video there's a gap between HW and SW decoders. HW decoders do MSB alignment as 16 bit format and we use the top ten bits while SW decoder do it the other way around (LSB).  SW decoder devs are insistent that this isn't just an arbitrary choice they could change.

SW decoder people think HW decoder people are crazy and vice versa.

Robert M asks if DRM drivers could support these formats so we can scanout sw-decoded video directly via DRM.




