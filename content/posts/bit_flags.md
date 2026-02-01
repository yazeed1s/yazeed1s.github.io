+++
title = "Using Bit Flags to Pack Multiple States Efficiently"
date = 2025-01-21
description = "A simple way to embed multiple boolean conditions in a single integer."
[taxonomies]
tags = ["C", "Bit Manipulation", "Systems Programming"]
+++

---

Sometimes you need to track multiple related conditions. Using a struct of booleans works, but it gets clunky when checking combinations or passing state around.

Bit flags offer a simpler pattern: pack everything into a single integer. The real win isn't memory usage, it's composability. You get a single value representing your entire state that you can pass, store, and compare in one go.

---

## the pattern

Each bit represents one condition. Powers of 2 so they don't overlap:

```c
typedef enum {
    FLAG_NONE = 0,
    FLAG_A    = (1U << 0),
    FLAG_B    = (1U << 1),
    FLAG_C    = (1U << 2),
    FLAG_D    = (1U << 3),
} flags_t;
```

The `1U` avoids signed overflow when shifting into bit 31.

```text
Bit position:    7   6   5   4   3   2   1   0
               +---+---+---+---+---+---+---+---+
               | 0 | 0 | 0 | 0 | 1 | 1 | 1 | 0 |  = 0x0E
               +---+---+---+---+---+---+---+---+
                                 ^   ^   ^
                                 D   C   B
```

Set with `|=`, clear with `&= ~`, check with `&`:

```c
flags |= FLAG_A;           /* set */
flags &= ~FLAG_A;          /* clear */
if (flags & FLAG_A) {...}  /* check */

/* check if ALL of A and B are set */
if ((flags & (FLAG_A | FLAG_B)) == (FLAG_A | FLAG_B)) {...}
```

---

## example: monitor state changes

In my window manager, I track what changed when monitors are plugged in or unplugged:

```c
typedef enum {
    MON_CHANGE_NONE         = 0,
    MON_CHANGE_CONNECTED    = (1U << 0),
    MON_CHANGE_DISCONNECTED = (1U << 1),
    MON_CHANGE_LAYOUT       = (1U << 2),
} monitor_change_t;

void update_monitors(uint32_t *changes) {
    *changes = MON_CHANGE_NONE;

    if (new_monitor_added) {
        *changes |= MON_CHANGE_CONNECTED;
    }

    if (monitor_removed) {
        *changes |= MON_CHANGE_DISCONNECTED;
    }

    if (resolution_changed) {
        *changes |= MON_CHANGE_LAYOUT;
    }
}
```

If you plug in a monitor and its resolution differs from expected, both events fire:

```text
Start:               00000000  (MON_CHANGE_NONE)

New monitor detected:
  set CONNECTED      |=  00000001  -> 00000001

Resolution differs:
  set LAYOUT         |=  00000100  -> 00000101

Final:
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 | 0 | 1 | 0 | 1 |  = 0x05
+---+---+---+---+---+---+---+---+
  7   6   5   4   3   2   1   0
                      L       C
```

One integer captures CONNECTED + LAYOUT. Now I can handle both:

```c
void handle_monitor_changes(void) {
    uint32_t m_change = MON_CHANGE_NONE;
    update_monitors(&m_change);

    if (m_change == MON_CHANGE_NONE) return;

    if (m_change & MON_CHANGE_CONNECTED)    { /* ... */ }
    if (m_change & MON_CHANGE_DISCONNECTED) { /* ... */ }
    if (m_change & MON_CHANGE_LAYOUT)       { /* ... */ }
}
```

With a single integer, I can track and respond to any combination of events.

## Another example: Red-Black Tree

A while back I implemented a red-black tree. To validate the tree invariants, I used bit flags to track which rules were violated:

```c
/* Red-black tree rules:
 * 1- Every node is either red or black.
 * 2- Root is always black.
 * 3- Every leaf (NULL) is black.
 * 4- If a node is red, both its children are black.
 * 5- Every path from root to leaf has the same number of black nodes.
 */

typedef enum {
    RB_VALID               = 0,
    RB_RED_ROOT            = (1U << 0),
    RB_RED_CHILD_OF_RED    = (1U << 1),
    RB_UNEQUAL_BLACK_PATHS = (1U << 2),
} rb_violation_t;

uint32_t validate_rbtree(node_t *root) {
    uint32_t result = RB_VALID;

    if (root->color == RED)
        result |= RB_RED_ROOT;
    if (has_red_child_of_red(root))
        result |= RB_RED_CHILD_OF_RED;
    if (!paths_have_equal_black_count(root))
        result |= RB_UNEQUAL_BLACK_PATHS;

    return result;
}
```

The caller checks specific violations:

```c
uint32_t v = validate_rbtree(tree);

if (v == RB_VALID) {
    printf("tree is valid\n");
} else {
    if (v & RB_RED_ROOT)
        printf("violation: root is red\n");
    if (v & RB_RED_CHILD_OF_RED)
        printf("violation: red node has red child\n");
}
```

One return value, multiple possible problems. Cleaner than an array of error codes or a struct with boolean fields.

---

## why uint32_t, not uint8_t

You might think: "I only have 4 flags. Why not `uint8_t`?"

Integer promotion. In C, bitwise operations promote smaller types to `int`:

```c
uint8_t flags = 0xFF;
if (~flags == 0) { ... }  /* false! ~flags is 0xFFFFFF00, not 0x00 */
```

The comparison happens at `int` width before truncation. This is easy to write and hard to debug.

Shifts have their own footguns too: the shift happens after integer promotions, and shifting by >= the width of the promoted type is undefined behavior (e.g., `1U << 32` on a system where `unsigned int` is 32-bit).

If you really want `uint8_t`, you can store it as `uint8_t` and be explicit about casts in your bit-twiddling. But in most code, `uint32_t` is the boring option: it fits in a register, avoids promotion surprises, and gives you room for 32 flags.

---

## why not just use bools

A `bool` in C is typically 1 byte, not 1 bit. Arrays and structs of bools don't pack bits either (each element takes a full byte). Memory savings only matter when you have many flags (5+).

But memory isn't the point. The point is that bit flags compose. A single integer can represent "connected AND layout changed" or "disconnected OR error"—and you can pass that combined state to a function, store it, or compare it in one operation.

---

## when bit flags hurt

Bit flags aren't always better:

- **Readability.** `flags & FLAG_A` isn't as obvious as `state.is_connected` at first glance. If the code isn't systems-level or performance-sensitive, the clarity cost may not be worth it.

- **Debugging.** Printing `0x0A` is less helpful than printing `"connected=true, layout=true"`. You'll want helper functions to decode flags when debugging.

- **Overkill for few flags.** If you have 2-3 independent conditions and never combine them, a struct of bools is simpler.

The pattern shines when state composition matters (events, options, validation results, capability bits). It's worse when conditions are truly independent and never need to travel together.

---

## the takeaway

Bit flags are about composability, not memory. One integer can represent any combination of states. You can pass it, store it, compare it, mask it.

Use them when you're building up state from multiple sources and handling combinations. Use bools when you're not. The choice is about how state flows through your code, not how many bytes it takes.
