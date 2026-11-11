---
title: "Reducing degradation in long Viggle Remake takes"
date: 2026-11-11T12:00:00-03:00
description: "A 1:23 ComfyUI test explores how fixed keystone and capstone anchors can help preserve the background across Viggle Remake passes."
summary: "How context protection combines a 17-frame overlap with fixed visual references to reduce accumulated changes in long takes."
tags: [AI, ComfyUI, Viggle Remake, video]
draft: false
---

## Reducing degradation in long Viggle Remake takes

When multiple Viggle Remake passes are chained together, each new window uses the previous window’s final 17 frames as context. This keeps the action moving forward, but small visual changes can accumulate. In a 1:23 video, I noticed this in the white fabric backdrop: it became less defined as the passes progressed.

The **Viggle Remake Long Video (Context Protection)** variant keeps the 17-frame overlap and adds stable visual references to help limit that drift.

## How context protection works

The node saves two latents from the first pass and reuses them in later passes:

- The **keystone** anchors the beginning of each new window to a fixed visual reference.
- The **capstone** provides a second reference near the end of the first window. It is introduced only during the final denoising steps, following the *Late Lock* technique, to reduce the chance of pulling generation toward an earlier pose.

The capstone stays fixed; it is not updated after every pass. The extra latents are removed before the output is assembled, so they do not add frames to the finished video. The original 17-frame overlap remains in place.

## What the test showed

I compared the source video, the output from the original node, and the Context Protection output at the first and last frames of a 1:23 test. In visual review, the background appeared to retain its look better in the protected version. In a shorter test, the join that had shown a one-frame glitch also appeared clean with the fixed capstone and *Late Lock*.

These are visual observations from a small number of tests, not an objective measurement. The technique is still experimental, so it is worth reviewing the joins and background details in each video.

**First frame**

![First-frame comparison: source on the left, original node in the center, and Context Protection on the right](https://raw.githubusercontent.com/hojenaia/viggle-remake-comfyui-long-take/main/images/comparison-first-frame.png)

**Last frame**

![Last-frame comparison: source on the left, original node in the center, and Context Protection on the right](https://raw.githubusercontent.com/hojenaia/viggle-remake-comfyui-long-take/main/images/comparison-last-frame.png)

## Code and workflow

The implementation and example workflow are available in the [viggle-remake-comfyui-long-take](https://github.com/hojenaia/viggle-remake-comfyui-long-take) repository. The Context Protection variant is experimental, and the original node remains available as an alternative.
