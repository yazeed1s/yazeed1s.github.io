+++
title = "My monitor was dying"
date = 2026-02-01
description = "debugging what looked like panel failure, turned out to be a shadow from polybar crossing monitor boundaries."
[taxonomies]
tags = ["Linux", "x11", "Picom", "Debugging"]
+++

So I bought a new Dell monitor and a few days in I noticed this dim shadow in the top-left corner. Sometimes it would stretch briefly when the monitor powered on, and it just looked physical. Like backlight bleed, like panel uniformity issues, like I somehow bought a defective unit.

## blaming hardware

When you see corner darkening the first thing you think is hardware. Monitor arm, maybe I twisted the panel when mounting it. Backlight diffuser issue. Bad cable or port. The monitor itself. So I ran through the usual checks, built-in diagnostics came back clean, and I also tried different cables and it changed nothing, the shadow just sat there looking like it belonged to the panel literally.

## wrong rabbit hole

Same day this started I was messing with `tlp` and `auto-cpufreq` so naturally I thought maybe I power-saved my way into a broken display, which was silly of me, but I couldn't shake the thought. So I turned all of that off and the shadow stayed. At least now I knew it wasn't some weird power profile thing causing this.

## software suspects

At this point I was thinking okay if it's not hardware and it's not power management, what else could it be. I started looking at my stack: custom window manager that I wrote, x11, picom with shadows and damage tracking. But then I thought wait, nothing in that layer had changed recently, no updates, no config changes. Not proof of anything really but it was enough to stop me from rewriting compositor configs.

## trying wayland

So I was stuck and I thought maybe I should just try a completely different display stack to rule things out. I logged into wayland and the shadow was gone. Same laptop, same cable, same monitor. That's when it clicked, wayland wasn't "better" here, it was just a control group. If the shadow disappears when I switch display servers then the monitor is clearly fine, and this has to be something in my x11 setup.

## bisect

Once I knew it was x11 and not the hardware I started thinking about what actually changed recently. I had updated my window manager a few days ago, right around when this started. So I thought let me try the laziest possible bisect: switch to x11 and check if the artifact returns, then roll back the WM changes and see if the artifact goes away. And that's exactly what happened. So this was a regression I introduced.

## what actually happened

So I started killing things to see what makes the shadow disappear. I disabled all window shadows in picom and the artifact was gone. Okay so it's definitely a shadow from some window. I re-enabled shadows and started killing windows one by one, and when I killed polybar (which was on my laptop screen) the dimming disappeared instantly. So the shadow was coming from polybar!! wtf.

But why was polybar casting a shadow that landed on my external monitor now? It wasn't doing this before my WM update. Then I thought about what I actually changed in that update. I had reimplemented the stacking order logic from scratch because the old code was a mess. And in the new implementation I correctly followed EWMH which says window types should stack in a specific order, and `_NET_WM_TYPE_DOCK` windows like polybar are supposed to sit above basically everything else. In my old WM code polybar wasn't stacked as high as it should have been, but now it was. And since I run picom with shadows enabled, polybar's shadow was now landing on top of other windows. On firefox or any bright window near that corner it looked exactly like panel dimming.

The funny thing is polybar wasn't even on the external monitor, it was on my laptop screen, and I wasn't even looking at my laptop this whole time.

## why the shadow crossed monitors

So I was thinking how is polybar's shadow even reaching the external monitor when polybar itself is only on my laptop screen. Then I remembered that x11 treats the whole desktop as one big coordinate space, monitors are just rectangles inside it:

```bash
$ xrandr --listmonitors
Monitors: 2
 0: +*eDP 1920/302x1200/189+0+0  eDP
 1: +DisplayPort-0 2560/597x1440/336+1920+0  DisplayPort-0
```

The external monitor starts at `x = 1920`. Shadows don't care about monitor boundaries, they're just pixels in global space. If you blur something near the right edge of the laptop screen that blur can spill into the left edge of the external monitor, which is exactly where I was seeing the "hardware defect."

## fix

Killed polybar and the dimming was gone instantly. Made sure polybar doesn't cast shadows and the fake hardware defect never came back.

## what I took from this

Panels do fail and this doesn't generalize to "it's always software" but isolating layers beats spiraling through hardware returns. Also multi-monitor on x11 is one big stupid canvas and effects don't stop at monitor seams unless you explicitly make them stop by padding between monitors or whatever.
