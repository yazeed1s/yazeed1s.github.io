+++
title = "Debugged a Broken Monitor (It Was a Shadow)"
date = 2026-02-01
description = "How an X11 composite shadow crossed monitor boundaries and looked like hardware failure."
[taxonomies]
tags = ["linux", "x11", "polybar", "picom", "debugging", "window manager"]
+++

I bought a new dell monitor. A few days in, I noticed a dim shadow in the top-left corner. Sometimes it stretched briefly on power-on.

It looked **_physical_**. Like panel uniformity issues. Like backlight bleed. Like I bought a defective monitor.

---

## hardware spiral
when you see a corner darkening, you blame everything physical:

- monitor arm (did i twist the panel?)
- backlight / diffuser
- cable / port
- the monitor itself

I ran through the usual checks. built-in diagnostics were clean. different cables didn't help. the shadow just sat there, stable, looking like it belonged to the panel.

---

## wrong rabbit hole
Same day this started, I was messing with `tlp` and `auto-cpufreq`. 

So naturally: "did I power-save my way into a broken display?". So I turned it all off. and the shadow stayed!!

At least now I knew it wasn't some weird power profile.

---

## software suspects
My stack started looking suspicious:

- custom window manager
- x11
- picom (shadows, damage tracking, the usual "maybe it's stale buffers")

Nothing in that layer had changed recently though. No updates, no config changes. Not proof, but enough to stop me from rewriting compositor configs at 2am.

---

## isolation
I logged into wayland. Shadow gone. same laptop. same cable. same monitor.

That's when I stopped worrying about RMA. wayland wasn't "better" here, it was just a control group.

---

## bisect
Once I knew it was x11 and not the monitor, I thought about what actually changed.

I had recently updated my window manager.
Laziest possible bisect:

- switch to x11 -> artifact returns
- roll back WM -> artifact gone

This was a regression I introduced.

---

## the reveal
Turns out I didn't mess up stacking order, I actually implemented it correctly. EWMH says window types stack in a specific order, and `polybar` (type `_NET_WM_TYPE_DOCK`) now sat above basically everything.

I run picom with shadows. polybar now had a shadow that landed on top of other windows.

On firefox or any bright window near that corner, it looked exactly like panel dimming.


The funniest thing is that the bar wasn't even on the external monitor. Only its shadow was.

---

## why the shadow landed on the other monitor
My laptop screen has polybar. External monitor doesn't.

But x11 treats the desktop as one big coordinate space. Monitors are just rectangles inside it:

```bash
$ xrandr --listmonitors
Monitors: 2
 0: +*eDP 1920/302x1200/189+0+0  eDP
 1: +DisplayPort-0 2560/597x1440/336+1920+0  DisplayPort-0
```

external starts at `x = 1920`.

shadows don't care about monitor boundaries. just pixels in global space. if you blur something near the right edge of the laptop screen, that blur can spill into the left edge of the external monitor.

which is exactly where i was seeing the "hardware defect."

---

## confirmation
Killed polybar. dimming gone instantly.
---

## the fix
Make sure polybar doesn't cast shadows. Fake hardware defect never came back.

---

## notes
- this doesn't generalize to "it's always software." panels do fail. but isolating layers beats spiraling.
- multi-monitor on x11 is one big canvas. effects don't stop at the seam unless you make them.
