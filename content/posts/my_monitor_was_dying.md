+++
title = "My monitor was dying"
date = 2026-02-01
description = "debugging what looked like panel failure, turned out to be a shadow from polybar crossing monitor boundaries."
[taxonomies]
tags = ["Linux", "x11", "Picom", "Debugging"]
+++

I got a new Dell monitor and a few days in I noticed a dim shadow in the top-left corner. It looked physical. Like backlight bleed or panel uniformity issues. Like I bought a defective unit.

## blaming hardware first

When you see corner darkening you think hardware. Did I twist the panel when mounting it? Bad cable? Backlight issue? I ran the usual checks. Built-in diagnostics were clean. Different cables didn't help. The shadow just sat there looking like it belonged to the panel.

## the wayland test

I was about to start an RMA process when I thought I should at least try a different display stack. Logged into a Wayland session. Same laptop, same cable, same monitor.

Shadow gone.

So the monitor was fine. This had to be something in my X11 setup.

## what changed

I tried to think about what was different. I had updated ZWM (my window manager) a few days ago, right around when this started. Everything else was the same: same picom config, same polybar, same everything.

Quick test: rolled back the ZWM changes. Shadow gone. Restored the changes. Shadow back.

So I broke something.

## finding the actual cause

I started killing things to narrow it down. Disabled shadows in picom. Artifact gone. So it's definitely a shadow from some window.

Re-enabled shadows and started killing windows one by one. When I killed polybar the shadow disappeared instantly.

Polybar was the source. But polybar was on my laptop screen, not on the external monitor. I wasn't even looking at my laptop.

## what I actually changed in ZWM

My WM update reimplemented the stacking order logic. The old code was a mess so I rewrote it to properly follow EWMH, which specifies that window types should stack in a particular order. `_NET_WM_TYPE_DOCK` windows (like polybar) are supposed to sit above most other windows.

Before my fix, polybar wasn't stacked as high as the spec says it should be. After my fix, it was. And since picom draws shadows on windows, polybar's shadow was now rendering on top of other windows instead of under them.

When Firefox or any bright window was near that corner of the screen, the shadow overlay looked exactly like panel dimming.

## why the shadow crossed monitors

X11 treats the entire desktop as one big coordinate space. Monitors are just rectangles positioned within it:

```bash
$ xrandr --listmonitors
Monitors: 2
 0: +*eDP 1920/302x1200/189+0+0  eDP
 1: +DisplayPort-0 2560/597x1440/336+1920+0  DisplayPort-0
```

My laptop screen is at x=0. The external monitor starts at x=1920.

Polybar sits at the top of the laptop screen. Its shadow extends past its bounds. If that extension reaches x=1920, it bleeds into the top-left corner of the external monitor.

Shadows and blur effects don't know about monitor boundaries. They just work in global pixel coordinates. That's where my "hardware defect" was.

## fix

Added polybar to picom's shadow exclusion list. Shadow gone. Problem never came back.

## takeaway

The monitor was fine the whole time. I almost returned working hardware because of a software shadow.

Isolating layers is faster than spiraling through hardware returns. The Wayland test took 30 seconds and told me everything I needed to know. Also learned that X11 multi-monitor is one big canvas and compositor effects don't stop at monitor boundaries unless you explicitly make them.
