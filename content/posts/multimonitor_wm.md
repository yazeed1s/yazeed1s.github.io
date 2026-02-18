+++
title = "Multi-Monitor Support in a Tiling WM"
date = 2024-09-01
description = "How ZWM handles multiple monitors: linked list of monitors, per-monitor desktops, and hotplug."
[taxonomies]
tags = ["Linux", "x11", "Window Manager"]
+++

I figured I should write down how multi-monitor actually works in ZWM.

The core idea is that each monitor is independent: it has its own desktops, its own focused window, and its own layout state. When you switch desktops, you switch on that monitor only, and when you tile windows, you tile within that monitor's usable area.

## the data model

Monitors are a linked list, and each monitor holds an array of desktops, a pointer to the currently focused desktop, physical dimensions from RandR, padding for bars, and some identity info:

```c
struct monitor_t {
    desktop_t         **desktops;   // array of desktops
    desktop_t          *desk;       // currently focused desktop
    monitor_t          *next;       // next monitor in list
    rectangle_t         rectangle;  // physical dimensions
    padding_t           padding;    // strut padding (for bars)
    xcb_randr_output_t  randr_id;
    uint32_t            id;
    bool                is_focused;
    bool                is_primary;
    // ...
};
```

Three global pointers track state:

```c
monitor_t *head_monitor = NULL;  // start of linked list
monitor_t *curr_monitor = NULL;  // currently active monitor
monitor_t *prim_monitor = NULL;  // primary monitor (usually where bar lives)
```

Most operations use `curr_monitor`. When you spawn a window, it goes to `curr_monitor->desk`, and when you switch desktops, you're switching `curr_monitor->desktops[n]`.

## detecting monitors at startup

At startup, ZWM checks what's available:

1. RandR if present (preferred)
2. Xinerama if RandR isn't there
3. Fallback to the whole screen as one monitor

```c
query_xr = xcb_get_extension_data(wm->connection, &xcb_randr_id);
query_x  = xcb_get_extension_data(wm->connection, &xcb_xinerama_id);

if (query_xr->present) {
    using_xrandr = true;
    randr_base = query_xr->first_event;
    xcb_randr_select_input(wm->connection, wm->root_window,
                           XCB_RANDR_NOTIFY_MASK_SCREEN_CHANGE);
}
```

If RandR is present, I also subscribe to screen change events, which is how hotplug works later. For RandR, the setup iterates outputs, skips disconnected ones, and creates a `monitor_t` for each connected output with a valid CRTC, where each monitor gets its rectangle from the CRTC geometry.

## desktops per monitor

Each monitor gets its own set of desktops:

```c
curr->n_of_desktops = conf.virtual_desktops;
curr->desktops = malloc(sizeof(desktop_t *) * curr->n_of_desktops);

for (int j = 0; j < curr->n_of_desktops; j++) {
    desktop_t *d = init_desktop();
    d->id = (uint16_t)j;
    d->is_focused = (j == 0);
    curr->desktops[j] = d;
}
curr->desk = curr->desktops[0];
```

So if you have 2 monitors and 5 desktops configured, you actually have 10 desktops total, 5 on each monitor. Desktop 1 on monitor 1 is completely independent from desktop 1 on monitor 2.

Desktop switching only affects the current monitor:

```c
for (int i = 0; i < curr_monitor->n_of_desktops; ++i) {
    curr_monitor->desktops[i]->is_focused = (curr_monitor->desktops[i]->id == id);
    if (curr_monitor->desktops[i]->id == id) {
        curr_monitor->desk = curr_monitor->desktops[i];
    }
}
```

## which monitor is current

This is pointer-driven, meaning the monitor under the mouse pointer is the current one. When a new window appears, I check where the pointer is and update `curr_monitor`:

```c
if (multi_monitors) {
    monitor_t *mm = get_focused_monitor();
    if (mm && mm != curr_monitor) {
        curr_monitor = mm;
    }
}
```

`get_focused_monitor()` queries pointer position and finds which monitor rectangle contains it. Same thing happens on enter notify and motion notify events: when the pointer crosses into a different monitor, `curr_monitor` updates.

## geometry and layout

Each monitor has a usable area which is the physical rectangle minus strut padding (for bars like polybar):

```c
int32_t x = m->rectangle.x + m->padding.left;
int32_t y = m->rectangle.y + m->padding.top;
int32_t w = (int32_t)m->rectangle.width - m->padding.left - m->padding.right;
int32_t h = (int32_t)m->rectangle.height - m->padding.top - m->padding.bottom;
```

Layouts compute within this usable area, and gaps and borders are subtracted:

```c
rectangle_t usable = get_usable_area(m);
r->x = usable.x + conf.window_gap;
r->y = usable.y + conf.window_gap;
r->width  = usable.width  - 2 * conf.window_gap - 2 * conf.border_width;
r->height = usable.height - 2 * conf.window_gap - 2 * conf.border_width;
```

Fullscreen is also monitor-aware: a fullscreen window fills its monitor, not the whole X11 screen:

```c
monitor_t *m = get_monitor_by_window(win);
rectangle_t r = m ? m->rectangle : curr_monitor->rectangle;
resize_window(win, r.width, r.height);
move_window(win, r.x, r.y);
```

## cycling monitors

Keyboard shortcut to move focus between monitors:

```c
if (m && m != curr) {
    move_mouse_to_monitor(m);
}
```

I warp the pointer to the target monitor, and the enter/motion handlers then update `curr_monitor`. I don't directly assign `curr_monitor` here because I want the pointer position to stay in sync with which monitor is active.

## hotplug

RandR sends `XCB_RANDR_SCREEN_CHANGE_NOTIFY` when monitors connect or disconnect:

```c
if (using_xrandr &&
    event_type == randr_base + XCB_RANDR_SCREEN_CHANGE_NOTIFY) {
    handle_monitor_changes();
    return 0;
}
```

`handle_monitor_changes()` detects three cases: a new monitor connected (create monitor node, initialize desktops, add to linked list), a monitor geometry changed (update the rectangle), and a monitor disconnected, which is the tricky one.

When a monitor disconnects, its windows need to go somewhere, so I merge them into another monitor:

```c
for (int i = 0; i < om->n_of_desktops; i++) {
    desktop_t *od = om->desktops[i];
    desktop_t *nd = nm->desktops[i];
    // traverse old desktop tree
    // for each window: unlink from old, transfer to new
    // rearrange target tree
}
```

Windows from desktop 1 of the old monitor go to desktop 1 of the remaining monitor, and then I destroy the old monitor node.

## notes

- RandR is the modern way, Xinerama is legacy but still needed for some setups
- The coordinate space is global: monitor at x=0 is leftmost, next monitor starts at x=1920 (or whatever)
- Strut padding comes from EWMH, bars like polybar set `_NET_WM_STRUT_PARTIAL` and the WM respects it
- Primary monitor is queried from RandR, and if not set, I default to head of the list
