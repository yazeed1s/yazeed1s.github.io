+++
title = "Using Bit Flags to Pack Multiple States Efficiently"
date = 2024-10-01
description = "A simple way to embed multiple boolean conditions in a single integer."
[taxonomies]
tags = ["C", "bit manipulation", "systems programming"]
+++

A struct of booleans is fine until you need to check combinations or pass the whole state around, and then it gets clunky. I ended up using bit flags for this in a few places, one integer where each bit is a condition, and I care less about the memory than about having the whole state in one value I can pass, store and compare.

## the pattern

Each flag is a different power of 2 so they never overlap (`1U` so shifting into bit 31 isn't signed overflow):

```c
typedef enum {
    FLAG_NONE = 0,
    FLAG_A    = (1U << 0),
    FLAG_B    = (1U << 1),
    FLAG_C    = (1U << 2),
    FLAG_D    = (1U << 3),
} flags_t;

flags |= FLAG_A;           /* set */
flags &= ~FLAG_A;          /* clear */
if (flags & FLAG_A) {...}  /* check */

/* both A and B set */
if ((flags & (FLAG_A | FLAG_B)) == (FLAG_A | FLAG_B)) {...}
```

## monitor changes in my WM

In my window manager, when monitors get plugged or unplugged I collect what changed into one value:

```c
typedef enum {
    MON_CHANGE_NONE         = 0,
    MON_CHANGE_CONNECTED    = (1U << 0),
    MON_CHANGE_DISCONNECTED = (1U << 1),
    MON_CHANGE_LAYOUT       = (1U << 2),
} monitor_change_t;

void update_monitors(uint32_t *changes) {
    *changes = MON_CHANGE_NONE;
    if (new_monitor_added)  *changes |= MON_CHANGE_CONNECTED;
    if (monitor_removed)    *changes |= MON_CHANGE_DISCONNECTED;
    if (resolution_changed) *changes |= MON_CHANGE_LAYOUT;
}
```

Plug in a monitor with an unexpected resolution and both bits get set, `0b101`, so the handler sees CONNECTED and LAYOUT in the same value and deals with each:

```c
uint32_t m_change = MON_CHANGE_NONE;
update_monitors(&m_change);
if (m_change == MON_CHANGE_NONE) return;

if (m_change & MON_CHANGE_CONNECTED)    { /* ... */ }
if (m_change & MON_CHANGE_DISCONNECTED) { /* ... */ }
if (m_change & MON_CHANGE_LAYOUT)       { /* ... */ }
```

I used the same thing when I wrote a red-black tree, the validator returns one integer with a bit per broken rule (`RB_RED_ROOT`, `RB_RED_CHILD_OF_RED`, `RB_UNEQUAL_BLACK_PATHS`), and `0` means the tree is valid. One return value can carry several problems at once, which beats an array of error codes.

## why uint32_t and not uint8_t

With 4 flags `uint8_t` looks like the obvious choice, but C promotes small types to `int` before bitwise ops:

```c
uint8_t flags = 0xFF;
if (~flags == 0) { ... }  /* false, ~flags is 0xFFFFFF00 as int */
```

That's easy to write and annoying to debug. Shifts get promoted too, and shifting by the width of the type or more is undefined (`1U << 32` with 32-bit `unsigned int`). You can store flags as `uint8_t` and cast carefully, but `uint32_t` is the boring choice, fits in a register, skips the promotion surprises, and gives you 32 flags.

## when I wouldn't

`flags & FLAG_A` reads worse than `state.is_connected`, and printing `0x0A` while debugging tells you less than `connected=true layout=true`, so you end up writing a decode helper. If I have two or three conditions that never travel together, a struct of bools is simpler and I'd just use that. Flags are worth it when the combination itself means something, events, options, validation results, capabilities.
