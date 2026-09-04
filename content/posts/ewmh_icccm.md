+++
title = "EWMH and ICCCM: What a WM Actually Needs to Implement"
date = 2024-06-02
description = "The two specs that define how X11 window managers communicate with clients."
[taxonomies]
tags = ["linux", "x11", "window manager"]
+++

EWMH and ICCCM are the two specs that define how window managers and clients talk to each other through properties on windows, atoms, and message formats. Without them every app would need to know specifically how to talk to your particular WM, and with them apps can just assume standard behavior. Every WM implements some subset of these, and here's what actually matters.

## ICCCM: the old one

ICCCM is the Inter-Client Communication Conventions Manual from 1988 and it covers the basics, so `WM_NAME` holds the window title, `WM_CLASS` holds the instance name and class name that you match windows to rules with, `WM_HINTS` carries the input focus model and icon and initial state, `WM_NORMAL_HINTS` carries size hints like min/max size and aspect ratio and resize increments, `WM_PROTOCOLS` lists which protocols the client supports such as `WM_DELETE_WINDOW`, and `WM_STATE` is the current state (normal, iconic, withdrawn).

Of those the ones that actually matter for a basic WM are a smaller set. `WM_DELETE_WINDOW` is the one where, if the client lists it in `WM_PROTOCOLS`, you send a ClientMessage instead of killing the window so the app can save unsaved work before it closes. `WM_TAKE_FOCUS` is for apps that handle focus themselves, where you send a ClientMessage when you want to focus the window and the app focuses its own input field. `WM_NORMAL_HINTS` is size hints again, and some apps set a minimum size or a fixed aspect ratio, so a tiling WM can mostly ignore this but floating windows should respect it. `WM_STATE` is something you're supposed to set on managed windows because some apps check it, with `NormalState` for visible, `IconicState` for minimized, and `WithdrawnState` for not managed. `WM_CLASS` is critical for window rules since the class name tells you what app this is, which is what you use to auto-float dialogs or send certain apps to specific desktops.

ICCCM is showing its age tho, it doesn't know about multiple desktops, fullscreen, maximized state, struts (docking bars), or window types, and that's what EWMH adds.

## EWMH: the modern one

EWMH is the Extended Window Manager Hints and it's what most modern apps and WMs actually use.

### root window properties

These go on the root window and describe the overall WM and desktop state. `_NET_SUPPORTED` is the list of EWMH atoms you support, which apps check to know what features work. `_NET_CLIENT_LIST` is all managed windows and `_NET_CLIENT_LIST_STACKING` is the same thing in stacking order from bottom to top, and pagers and taskbars use both. `_NET_CURRENT_DESKTOP` is which desktop is active, `_NET_NUMBER_OF_DESKTOPS` is how many exist, and `_NET_DESKTOP_NAMES` is their names. `_NET_ACTIVE_WINDOW` is the currently focused window. `_NET_SUPPORTING_WM_CHECK` points to a child window with `_NET_WM_NAME` set to the WM name, and apps use it to detect whether a compliant WM is running at all.

### client window properties

Apps set these on their own windows. `_NET_WM_NAME` is the UTF-8 window title and you should prefer it over `WM_NAME`. `_NET_WM_WINDOW_TYPE` says what kind of window it is, with `_NET_WM_WINDOW_TYPE_NORMAL` for a regular app window, `_DOCK` for panels and bars like polybar, `_DIALOG` for dialogs, `_SPLASH` for splash screens, `_UTILITY` for toolbars and palettes, and `_NOTIFICATION` for notification popups. This is how you know to auto-float dialogs or keep docks above other windows, and getting it right makes a huge difference in how the WM feels.

`_NET_WM_STATE` is the set of current state flags, so `_NET_WM_STATE_FULLSCREEN` for fullscreen, `_MAXIMIZED_HORZ` and `_MAXIMIZED_VERT` for maximized, `_HIDDEN` for minimized, `_ABOVE` and `_BELOW` for always on top or bottom, and `_DEMANDS_ATTENTION` for the urgency hint. `_NET_WM_DESKTOP` is which desktop the window belongs to, where `0xFFFFFFFF` means sticky and visible on all of them. `_NET_WM_STRUT_PARTIAL` is reserved screen space that docks set to tell the WM not to tile windows in their area. `_NET_WM_PID` is the process ID, which is useful when you want to know which process owns a window.

### client messages

Apps send these to the WM to request things and the WM decides whether to honor them. `_NET_ACTIVE_WINDOW` requests focus for a window and pagers use it. `_NET_CLOSE_WINDOW` requests a close and taskbars use it. `_NET_WM_STATE` requests a state change like toggling fullscreen, and the message carries the action (add, remove, toggle) plus which state atoms. `_NET_CURRENT_DESKTOP` requests a desktop switch and `_NET_WM_DESKTOP` requests moving a window to a different desktop.

## what I actually implemented

For a tiling WM the spec is big but you don't need all of it. The parts that have to be there are `_NET_SUPPORTED` to advertise what you support, `_NET_SUPPORTING_WM_CHECK` to prove you're EWMH compliant, `_NET_CLIENT_LIST` because pagers need it, `_NET_CURRENT_DESKTOP` and `_NET_NUMBER_OF_DESKTOPS` for desktop switching, `_NET_ACTIVE_WINDOW` for focus tracking, `_NET_WM_STATE` handling and especially fullscreen, `_NET_WM_WINDOW_TYPE` to identify docks and dialogs and splashes, `_NET_WM_STRUT_PARTIAL` to respect space reserved by bars, and `WM_DELETE_WINDOW` to close windows properly.

The nice-to-haves are `_NET_WM_NAME` for UTF-8 titles, `_NET_CLIENT_LIST_STACKING` for stacking-aware pagers, `_NET_DESKTOP_NAMES` for named workspaces, and `_NET_WM_DESKTOP` for moving windows between desktops. The things you can skip in a tiling WM are `_NET_WM_STATE_MAXIMIZED_*` since tiling windows are already maximized in a sense, and `_NET_WM_MOVERESIZE` for interactive resize and move, which isn't really relevant when everything tiles.

## handling state changes

When an app sends a `_NET_WM_STATE` client message, `data.l[0]` is the action (0 remove, 1 add, 2 toggle), `data.l[1]` is the first state atom, and `data.l[2]` is an optional second state atom. So a fullscreen toggle comes out roughly like this:

```c
if (action == _NET_WM_STATE_TOGGLE) {
    if (is_fullscreen(win)) {
        unfullscreen(win);
    } else {
        fullscreen(win);
    }
}
```

You also need to update the property on the window afterwards so other apps like pagers and taskbars can see the current state.

## the annoying parts

Order matters, because some apps expect you to set properties in a specific order during the initial manage, and Java is notorious for this. UTF-8 is another one, since `_NET_WM_NAME` is UTF-8 but `WM_NAME` might be latin1 or something else entirely, so you have to handle both. Struts are per-monitor and `_NET_WM_STRUT_PARTIAL` has 12 values covering left, right, top, bottom, and a start and end for each, so a bar on the left side of monitor 2 has a specific left strut with start_y and end_y bounding its position. Desktop indices trip people up too, because apps expect 0-indexed desktops where `_NET_CURRENT_DESKTOP = 0` is the first one, and some early EWMH implementations got this wrong.

## notes

- ICCCM: https://tronche.com/gui/x/icccm/
- EWMH: https://specifications.freedesktop.org/wm-spec/latest/
- `xcb_ewmh.h` provides helpers for getting/setting these properties
- If something isn't working, check what you advertise in `_NET_SUPPORTED`, apps check this
- Run `xprop` on windows to see what properties they set, useful for debugging
