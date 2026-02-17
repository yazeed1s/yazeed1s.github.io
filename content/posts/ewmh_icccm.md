+++
title = "EWMH and ICCCM: What a WM Actually Needs to Implement"
date = 2024-06-02
description = "The two specs that define how X11 window managers communicate with clients."
[taxonomies]
tags = ["Linux", "x11", "Window Manager"]
+++

EWMH and ICCCM are specs that define how window managers and clients communicate. Properties on windows, atoms, message formats. Without them, every app would need to know specifically how to talk to your WM. With them, apps can assume standard behavior.

Every WM implements some subset. Here's what actually matters.

## ICCCM: the old one

ICCCM is the Inter-Client Communication Conventions Manual. It's from 1988. Defines the basics:

- `WM_NAME` — window title
- `WM_CLASS` — instance name and class name (used for matching windows to rules)
- `WM_HINTS` — input focus model, icon, initial state
- `WM_NORMAL_HINTS` — size hints (min/max size, aspect ratio, resize increments)
- `WM_PROTOCOLS` — which protocols the client supports (like `WM_DELETE_WINDOW`)
- `WM_STATE` — current state (normal, iconic, withdrawn)

The important ones for a basic WM:

**WM_DELETE_WINDOW** — if the client lists this in `WM_PROTOCOLS`, send a ClientMessage instead of killing the window. This lets the app save unsaved work before closing.

**WM_TAKE_FOCUS** — for apps that handle focus themselves. When you want to focus the window, send a ClientMessage and the app focuses its own input field.

**WM_NORMAL_HINTS** — size hints. Some apps set minimum size or fixed aspect ratio. If you're a tiling WM you might ignore this, but floating windows should respect it.

**WM_STATE** — you're supposed to set this on managed windows. Some apps check it. `NormalState` for visible, `IconicState` for minimized, `WithdrawnState` for not managed.

**WM_CLASS** — critical for window rules. The class name tells you what app this is. You want this to auto-float dialogs, send certain apps to specific desktops, etc.

ICCCM is showing its age. It doesn't know about multiple desktops, fullscreen, maximized state, struts (docking bars), or window types. That's what EWMH adds.

## EWMH: the modern one

EWMH is the Extended Window Manager Hints. It's what most modern apps and WMs actually use.

### root window properties

These go on the root window and describe the WM and desktop state:

**`_NET_SUPPORTED`** — list of EWMH atoms you support. Apps check this to know what features work.

**`_NET_CLIENT_LIST`** — all managed windows. Pagers and taskbars use this.

**`_NET_CLIENT_LIST_STACKING`** — same but in stacking order (bottom to top).

**`_NET_CURRENT_DESKTOP`** — which desktop is active.

**`_NET_NUMBER_OF_DESKTOPS`** — how many desktops exist.

**`_NET_DESKTOP_NAMES`** — names of desktops.

**`_NET_ACTIVE_WINDOW`** — currently focused window.

**`_NET_SUPPORTING_WM_CHECK`** — points to a child window with `_NET_WM_NAME` set to WM name. Apps use this to detect if a compliant WM is running.

### client window properties

Apps set these on their windows:

**`_NET_WM_NAME`** — UTF-8 window title. Prefer over `WM_NAME`.

**`_NET_WM_WINDOW_TYPE`** — what kind of window is this:

- `_NET_WM_WINDOW_TYPE_NORMAL` — regular app window
- `_NET_WM_WINDOW_TYPE_DOCK` — panels/bars (like polybar)
- `_NET_WM_WINDOW_TYPE_DIALOG` — dialog windows
- `_NET_WM_WINDOW_TYPE_SPLASH` — splash screens
- `_NET_WM_WINDOW_TYPE_UTILITY` — toolbars, palettes
- `_NET_WM_WINDOW_TYPE_NOTIFICATION` — notification popups

This is how you know to auto-float dialogs or keep docks above other windows.

**`_NET_WM_STATE`** — current window state flags:

- `_NET_WM_STATE_FULLSCREEN` — fullscreen
- `_NET_WM_STATE_MAXIMIZED_HORZ/VERT` — maximized
- `_NET_WM_STATE_HIDDEN` — minimized
- `_NET_WM_STATE_ABOVE` — always on top
- `_NET_WM_STATE_BELOW` — always on bottom
- `_NET_WM_STATE_DEMANDS_ATTENTION` — urgency hint

**`_NET_WM_DESKTOP`** — which desktop this window belongs to. `0xFFFFFFFF` means sticky (visible on all desktops).

**`_NET_WM_STRUT_PARTIAL`** — reserved screen space. Docks set this to tell the WM "don't tile windows in my area."

**`_NET_WM_PID`** — process ID. Useful for "which process owns this window?"

### client messages

Apps send these to the WM to request things:

**`_NET_ACTIVE_WINDOW`** — request focus for a window. Pagers use this.

**`_NET_CLOSE_WINDOW`** — request to close a window. Taskbars use this.

**`_NET_WM_STATE`** — request state change (toggle fullscreen, etc). The message includes the action (add/remove/toggle) and which state atoms.

**`_NET_CURRENT_DESKTOP`** — request desktop switch.

**`_NET_WM_DESKTOP`** — request to move window to different desktop.

## what I actually implemented

For a tiling WM, here's the bare minimum that makes things work:

**Must have:**

- `_NET_SUPPORTED` — advertise what you support
- `_NET_SUPPORTING_WM_CHECK` — prove you're EWMH compliant
- `_NET_CLIENT_LIST` — pagers need this
- `_NET_CURRENT_DESKTOP` / `_NET_NUMBER_OF_DESKTOPS` — desktop switching
- `_NET_ACTIVE_WINDOW` — focus tracking
- `_NET_WM_STATE` handling — especially fullscreen
- `_NET_WM_WINDOW_TYPE` — to identify docks, dialogs, splashes
- `_NET_WM_STRUT_PARTIAL` — to respect space reserved by bars
- `WM_DELETE_WINDOW` — to close windows properly

**Good to have:**

- `_NET_WM_NAME` — for UTF-8 titles
- `_NET_CLIENT_LIST_STACKING` — for stacking-aware pagers
- `_NET_DESKTOP_NAMES` — for named workspaces
- `_NET_WM_DESKTOP` — for moving windows between desktops

**Can skip in tiling WM:**

- `_NET_WM_STATE_MAXIMIZED_*` — tiling windows are already maximized in a sense
- `_NET_WM_MOVERESIZE` — for interactive resize/move, not really relevant if everything tiles

## handling state changes

When an app sends `_NET_WM_STATE` client message, it looks like:

- `data.l[0]` = action (0=remove, 1=add, 2=toggle)
- `data.l[1]` = first state atom
- `data.l[2]` = second state atom (optional)

So to handle fullscreen toggle:

```c
if (action == _NET_WM_STATE_TOGGLE) {
    if (is_fullscreen(win)) {
        unfullscreen(win);
    } else {
        fullscreen(win);
    }
}
```

You also need to update the property on the window so other apps can see the current state.

## the annoying parts

**Order matters.** Some apps expect you to set properties in specific order during initial manage. Java is notorious for this.

**UTF-8 everywhere.** `_NET_WM_NAME` is UTF-8. `WM_NAME` might be latin1 or something else. You need to handle both.

**Struts are per-monitor.** `_NET_WM_STRUT_PARTIAL` has 12 values: left, right, top, bottom, and start/end for each. A bar on the left side of monitor 2 has a specific left strut with start_y/end_y bounding its position.

**Desktop indices.** Apps expect 0-indexed desktops. `_NET_CURRENT_DESKTOP = 0` is the first desktop. Some early EWMH implementations got this wrong.

## notes

- ICCCM: https://tronche.com/gui/x/icccm/
- EWMH: https://specifications.freedesktop.org/wm-spec/latest/
- `xcb_ewmh.h` provides helpers for getting/setting these properties
- If something isn't working, check what you advertise in `_NET_SUPPORTED`. Apps check this.
- Run `xprop` on windows to see what properties they set. Useful for debugging.
