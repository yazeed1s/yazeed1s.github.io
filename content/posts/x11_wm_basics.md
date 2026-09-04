+++
title = "How a Window Manager Talks to X11"
date = 2024-05-08
description = "XCB event masks, SubstructureRedirect, and how a WM becomes the WM."
[taxonomies]
tags = ["linux", "x11", "window manager"]
+++

When I started writing ZWM I didn't understand how a window manager intercepts window creation, because applications create windows by talking to the X server directly so how does the WM get in the middle? turns out the WM is just another X client, but it asks the server to redirect certain events to it instead of handling them normally.

## everything is events

X11 is event-driven, so when something happens like a key press or a window being created or resized the server generates an event, and clients tell the server which events they want to receive by setting event masks on windows.

In XCB, event masks are just bit flags you OR together:

```c
#define S_NOTIFY    (XCB_EVENT_MASK_SUBSTRUCTURE_NOTIFY)
#define S_REDIRECT  (XCB_EVENT_MASK_SUBSTRUCTURE_REDIRECT)
#define SUBSTRUCTURE (S_NOTIFY | S_REDIRECT)

#define PROPERTY_CHANGE (XCB_EVENT_MASK_PROPERTY_CHANGE)
#define FOCUS_CHANGE    (XCB_EVENT_MASK_FOCUS_CHANGE)
#define ENTER_WINDOW    (XCB_EVENT_MASK_ENTER_WINDOW)
#define LEAVE_WINDOW    (XCB_EVENT_MASK_LEAVE_WINDOW)
#define BUTTON_PRESS    (XCB_EVENT_MASK_BUTTON_PRESS)
```

You combine whatever you need for a specific window.

## the root window

There's one special window called the root window that covers the entire screen, and all other windows are children of it.

The root window is how the WM gets control, because by setting certain masks on it the WM intercepts events that would normally be handled by the server. In ZWM I define what the root window listens for:

```c
#define ROOT_EVENT_MASK \
    (SUBSTRUCTURE | BUTTON_PRESS | FOCUS_CHANGE | ENTER_WINDOW)
```

`SUBSTRUCTURE` is the key part. That's `SubstructureNotify | SubstructureRedirect`.

## SubstructureRedirectMask

When you set `XCB_EVENT_MASK_SUBSTRUCTURE_REDIRECT` on the root window, you're telling the X server not to automatically handle requests that affect the root window's children, and to send them to you instead.

What gets redirected:
- `XCB_MAP_REQUEST`, client wants to show a window
- `XCB_CONFIGURE_REQUEST`, client wants to move or resize
- `XCB_CIRCULATE_REQUEST`, client wants to change stacking order

Without this mask, when an app wants to map a window the server just shows it wherever the app asked, but with the mask you get a `MapRequest` event and decide what to do, which is how tiling WMs work, where the app says "make me 800x600" and the WM intercepts it, ignores the size, and tiles it wherever it should go.

## only one WM

Only one client can hold `SubstructureRedirectMask` on root at a time, so if another client already has it you get `BadAccess`, which is how you detect if another WM is already running.

## what clients listen for

Client windows need different masks than root. In ZWM:

```c
#define CLIENT_EVENT_MASK \
    (PROPERTY_CHANGE | FOCUS_CHANGE | ENTER_WINDOW | LEAVE_WINDOW)
```

- `PROPERTY_CHANGE`, know when window title or state changes
- `FOCUS_CHANGE`, track focus in and out
- `ENTER_WINDOW` and `LEAVE_WINDOW`, for focus-follows-mouse

## the event loop

XCB uses `xcb_wait_for_event()` which blocks until there's an event, and my event loop in ZWM looks like this:

```c
static void
event_loop(wm_t *w)
{
    xcb_event_t *event;
    while (!should_shutdown && (event = xcb_wait_for_event(w->connection))) {
        if (event->response_type == 0) {
            _FREE_(event);
            continue;
        }
        if (handle_event(event) != 0) {
            uint8_t type = event->response_type & ~0x80;
            char   *es   = xcb_event_to_string(type);
            _LOG_(ERROR, "error processing event: %s ", es);
        }
        _FREE_(event);
    }
}
```

The `& ~0x80` strips the "sent by SendEvent" flag. You dispatch on the event type.

## dispatching events

I use a table-driven approach where each event type maps to a handler function:

```c
static const event_handler_entry_t _handlers_[] = {
    // map request - window wants to be displayed
    DEFINE_MAPPING(XCB_MAP_REQUEST, handle_map_request),
    // unmap notify - window was hidden
    DEFINE_MAPPING(XCB_UNMAP_NOTIFY, handle_unmap_notify),
    // destroy notify - window was killed
    DEFINE_MAPPING(XCB_DESTROY_NOTIFY, handle_destroy_notify),
    // client message - EWMH requests from other apps
    DEFINE_MAPPING(XCB_CLIENT_MESSAGE, handle_client_message),
    // configure request - client wants to resize/move
    DEFINE_MAPPING(XCB_CONFIGURE_REQUEST, handle_configure_request),
    // enter notify - cursor entered window
    DEFINE_MAPPING(XCB_ENTER_NOTIFY, handle_enter_notify),
    // button press - mouse button clicked
    DEFINE_MAPPING(XCB_BUTTON_PRESS, handle_button_press_event),
    // key press - keyboard input
    DEFINE_MAPPING(XCB_KEY_PRESS, handle_key_press),
    // property notify - window property changed
    DEFINE_MAPPING(XCB_PROPERTY_NOTIFY, handle_property_notify),
};
```

Then dispatch is just a loop through the table:

```c
size_t n = sizeof(_handlers_) / sizeof(_handlers_[0]);
for (size_t i = 0; i < n; i++) {
    if (_handlers_[i].type == event_type) {
        return _handlers_[i].handle(event);
    }
}
```

## the important events

**MapRequest** is a window wanting to appear, so you decide if you manage it, where it goes, and what size it gets.

**ConfigureRequest** is a client wanting to move or resize, and for a tiling WM you mostly ignore the requested geometry, but for floating windows you might honor it.

**ClientMessage** is the EWMH protocol, where other apps send these to request things like:
- `_NET_WM_STATE`, toggle fullscreen, above, below
- `_NET_ACTIVE_WINDOW`, bring window to front
- `_NET_CLOSE_WINDOW`, close a window from a pager
- `_NET_CURRENT_DESKTOP`, switch desktop

**PropertyNotify** is a window title changing, WM hints updating, and so on.

**EnterNotify** is the cursor entering a window, used for focus-follows-mouse.

## reparenting vs non-reparenting

When a WM "manages" a window there are two approaches. Reparenting WMs create a frame window for each client, and the client window becomes a child of the frame:

```
Before managing:
  root -> client

After managing (reparenting):
  root -> frame -> client
```

The frame is where decorations go, so the title bar, borders, and close button, and when you move the frame the client moves with it. i3 does this, drawing frames around windows and using them for the colored borders and title bars. Non-reparenting WMs leave the client as a direct child of root:

```
After managing (non-reparenting):
  root -> client
```

There's no frame, so if you want borders you manipulate the client window's X11 border directly with `xcb_configure_window` and `XCB_CONFIG_WINDOW_BORDER_WIDTH`. dwm does this and ZWM does too. The tradeoff is that reparenting gives you more flexibility for decorations but adds complexity, since you have to handle ReparentNotify events, manage the frame lifecycle, and deal with apps that don't like being reparented, while non-reparenting is simpler but you're limited to what X11 window borders can do, which is basically solid color rectangles.

Some apps behave differently under reparenting WMs. Java apps are the worst, they historically had issues with reparenting and they still do, and they expect things to work a certain way in a certain order. Chrome and Electron apps sometimes need hints. Most modern apps handle it fine though.

## what I learned

The WM isn't special at all, it's just a client that grabbed redirect on root first. The X server doesn't even know what a "window manager" is, it just follows the rules about who gets which events.

## notes

- XCB is lower level than Xlib. No magic, you manage memory yourself.
- ICCCM and EWMH define conventions. Not enforced, just expected.
- `xcb_wait_for_event()` blocks. For non-blocking, use `xcb_poll_for_event()`.
- Event response type 0 means error, not a real event.
