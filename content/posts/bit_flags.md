+++
title = "Using Bit Flags to Pack Multiple States Efficiently"
date = 2025-01-21
description = "A simple way to embed multiple boolean conditions in a single integer."
[taxonomies]
tags = ["C", "Bit Manipulation", "Systems Programming"]
+++

---

Sometimes you need to track multiple boolean conditions at once. You could use a bunch of separate `bool` variables, or an array, or a struct with fields. But there's a cleaner way: pack them all into a single integer using bit flags.

---

## The Idea

Each bit in an integer represents one condition. Bit 0 is condition A, bit 1 is condition B, and so on. A 32-bit integer gives you 32 independent flags. You set, clear, and check them using bitwise operators. I personally like to think of it as a list of `"light switches"` that can be either on or off. In a 32-bit integer, you have 32 light switches, each one represents a different condition and can be either on or off.

```text
Bit position:    7   6   5   4   3   2   1   0
               +---+---+---+---+---+---+---+---+
               | 0 | 0 | 0 | 0 | 1 | 1 | 1 | 0 |  = 0x0E (14)
               +---+---+---+---+---+---+---+---+
                                 ^   ^   ^
                                 |   |   +-- FLAG_B (bit 1)
                                 |   +------ FLAG_C (bit 2)
                                 +---------- FLAG_D (bit 3)
```

The value `0x0E` (14 in decimal) means flags B, C, and D are set.

## Defining Flags

Use powers of 2 so each flag occupies exactly one bit:

```c
#define FLAG_A  (1 << 0)  /* 0x01 = 00000001 */
#define FLAG_B  (1 << 1)  /* 0x02 = 00000010 */
#define FLAG_C  (1 << 2)  /* 0x04 = 00000100 */
#define FLAG_D  (1 << 3)  /* 0x08 = 00001000 */
```

Or use an enum, which is cleaner:

```c
typedef enum {
    FLAG_NONE = 0,
    FLAG_A    = (1 << 0),  /* bit 0 */
    FLAG_B    = (1 << 1),  /* bit 1 */
    FLAG_C    = (1 << 2),  /* bit 2 */
    FLAG_D    = (1 << 3),  /* bit 3 */
} flags_t;
```

## Operations

**Set a flag:**

```c
flags |= FLAG_A;  /* turn on bit 0 */
```

**Clear a flag:**

```c
flags &= ~FLAG_A;  /* turn off bit 0 */
```

**Toggle a flag:**

```c
flags ^= FLAG_A;  /* flip bit 0 */
```

**Check if a flag is set:**

```c
if (flags & FLAG_A) {
    /* FLAG_A is set */
}
```

**Check multiple flags:**

```c
/* check if ANY of these are set */
if (flags & (FLAG_A | FLAG_B)) { ... }

/* check if ALL of these are set */
if ((flags & (FLAG_A | FLAG_B)) == (FLAG_A | FLAG_B)) { ... }
```

## Visualizing Multiple Flag Operations

Step by step when you combine flags:

```text
Start with empty flags:
flags = 0x00         00000000

Set FLAG_A (bit 0):
flags |= FLAG_A      00000000
                   | 00000001  (FLAG_A)
                   ----------
                     00000001  = 0x01

Set FLAG_C (bit 2):
flags |= FLAG_C      00000001
                   | 00000100  (FLAG_C)
                   ----------
                     00000101  = 0x05

Set FLAG_D (bit 3):
flags |= FLAG_D      00000101
                   | 00001000  (FLAG_D)
                   ----------
                     00001101  = 0x0D

Current state:
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |  = 0x0D (13)
+---+---+---+---+---+---+---+---+
  7   6   5   4   3   2   1   0
                  D   C       A   <- flags set
```

Now let's clear FLAG_C:

```text
Clear FLAG_C:
~FLAG_C              11111011  (invert FLAG_C)

flags &= ~FLAG_C     00001101
                   & 11111011
                   ----------
                     00001001  = 0x09

Result:
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 | 1 | 0 | 0 | 1 |  = 0x09 (9)
+---+---+---+---+---+---+---+---+
  7   6   5   4   3   2   1   0
                  D           A   ← flags set (C is gone)
```

## Real Example: Monitor State Changes

In my window manager, I track what changed when monitors are plugged in or unplugged:

```c
/* bit flags for monitor state changes */
typedef enum {
    _NONE        = (1 << 0),  /* 00000001 - no change */
    CONNECTED    = (1 << 1),  /* 00000010 - monitor connected */
    DISCONNECTED = (1 << 2),  /* 00000100 - monitor disconnected */
    LAYOUT       = (1 << 3),  /* 00001000 - resolution/position changed */
} monitor_state_t;
```

When I query the X server for monitor changes, I build up a bitmask:

```c
void update_monitors(uint32_t *changes) {
    /* start with "no change" flag */
    *changes = _NONE;

    /* ... query X server for outputs ... */

    if (new_monitor_added) {
        *changes &= ~_NONE;       /* clear the "no change" flag */
        *changes |= CONNECTED;    /* set the "connected" flag */
    }

    if (monitor_removed) {
        *changes &= ~_NONE;
        *changes |= DISCONNECTED;
    }

    if (resolution_changed) {
        *changes &= ~_NONE;
        *changes |= LAYOUT;
    }
}
```

Imagine a scenario: you plug in a new monitor AND that monitor has a different resolution than expected. Both events happen:

```text
Start:               00000001  (_NONE)

New monitor detected:
  clear _NONE        &= ~00000001  -> 00000000
  set CONNECTED      |=  00000010  -> 00000010

Resolution differs:
  set LAYOUT         |=  00001000  -> 00001010

Final value:
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 | 1 | 0 | 1 | 0 |  = 0x0A (10)
+---+---+---+---+---+---+---+---+
                  L       C
                  A       O
                  Y       N
                  O       N
                  U       E
                  T       C
                          T
                          E
                          D
```

Now I can handle both events:

```c
void handle_monitor_changes(void) {
    uint32_t m_change = _NONE;
    update_monitors(&m_change);

    if (m_change & _NONE) {
        /* nothing changed */
        return;
    }

    if (m_change & CONNECTED) {
        /* handle new monitor */
    }

    if (m_change & DISCONNECTED) {
        /* handle removed monitor */
    }

    if (m_change & LAYOUT) {
        /* handle resolution change */
    }
    /* ... handle other events ... */
}
```

With a single integer, I can track and respond to any combination of events.

## Validation Example: Red-Black Tree

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
    RB_VALID               = 0x00,  /* 00000000 - no violations */
    RB_INVALID_COLOR       = 0x01,  /* 00000001 - rule 1 broken */
    RB_RED_ROOT            = 0x02,  /* 00000010 - rule 2 broken */
    RB_NULL_NOT_BLACK      = 0x04,  /* 00000100 - rule 3 broken */
    RB_RED_CHILD_OF_RED    = 0x08,  /* 00001000 - rule 4 broken */
    RB_UNEQUAL_BLACK_PATHS = 0x10,  /* 00010000 - rule 5 broken */
} rb_violation_t;

typedef uint32_t rb_validation_t;
```

The validation function returns all violations at once:

```c
rb_validation_t validate_rbtree(node_t *root) {
    rb_validation_t result = RB_VALID;

    if (root->color == RED) {
        result |= RB_RED_ROOT;
    }

    if (has_red_child_of_red(root)) {
        result |= RB_RED_CHILD_OF_RED;
    }

    if (!paths_have_equal_black_count(root)) {
        result |= RB_UNEQUAL_BLACK_PATHS;
    }

    return result;
}
```

Say the tree has a red root AND a red node with a red child. The result looks like:

```text
RB_RED_ROOT         = 0x02 = 00000010
RB_RED_CHILD_OF_RED = 0x08 = 00001000
                           ----------
Combined (OR)       = 0x0A = 00001010

+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 | 1 | 0 | 1 | 0 |  = 0x0A (10)
+---+---+---+---+---+---+---+---+
  7   6   5   4   3   2   1   0
                  ^       ^
                  |       +-- RB_RED_ROOT (rule 2)
                  +---------- RB_RED_CHILD_OF_RED (rule 4)
```

Then you can check what's wrong:

```c
rb_validation_t v = validate_rbtree(tree);

if (v == RB_VALID) {
    printf("tree is valid\n");
} else {
    if (v & RB_RED_ROOT)
        printf("violation: root is red\n");
    if (v & RB_RED_CHILD_OF_RED)
        printf("violation: red node has red child\n");
    if (v & RB_UNEQUAL_BLACK_PATHS)
        printf("Violation: unequal black paths\n");
}
```

This is way cleaner than returning an array of error codes or having separate boolean fields for each violation.

## Why Use Bit Flags?

1. **Memory efficient** - One integer holds 32 (or 64) flags instead of 32 separate bools
2. **Fast** - Bitwise operations are single CPU instructions
3. **Convenient** - Test multiple conditions with one comparison
4. **Combinable** - Easy to pass around, store, and combine multiple states

The main downside is readability - `flags & FLAG_A` isn't as obvious as `flags.a` at first glance. But once you're used to the pattern, it's fine.

## Summary

- Each flag is a power of 2: `(1 << n)`
- Set with `|=`, clear with `&= ~`, check with `&`
- Good for tracking multiple boolean conditions in one value
- Common in systems programming, graphics, protocols, and anywhere you need compact state representation
