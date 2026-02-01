+++
title = "BSP Trees for Tiling Window Management"
date = 2024-07-21
description = "How I use binary space partitioning to manage windows in my X11 window manager."
[taxonomies]
tags = ["C", "Spatial Data Structures", "Tiling Window Manager"]
+++

---

Some time ago, I decided to build my own tiling window manager from scratch. I was working with spatial data structures at work, and BSP trees seemed like a good fit for this kind of problem. And It all sounded like a fun challenge. 

---

## What's a Tiling Window Manager?

A tiling window manager arranges windows so they don't overlap. Instead of floating windows on top of each other like macOS or Windows, it splits the screen into sections. You open a window, it goes into a tile. Open another, the screen splits. You don't manually position anything.

Linux has a bunch of these (dwm, i3, bspwm). On Mac there's Yabai. They're not mainstream, but if you're into keyboard-driven workflows, they're nice.

The question is: how do you represent this layout in memory?

## What the Layout Data Structure Needs to Do

Before picking a data structure, it helps to think about what operations matter.

A tiling window manager, at minimum, needs to:

- **Insert** a new window when one opens, splitting the focused region
- **Delete** a window when it closes, reclaiming the space
- **Resize** regions by adjusting split ratios
- **Traverse** the layout to find the next/previous window
- **Render** by walking the structure and applying positions to X11

The shape of the data matters too. Window layouts are inherently hierarchical and spatial. The screen splits into halves, those halves split again. It's not a flat sequence at all.

Whatever structure we pick, it should make the common operations cheap and match this hierarchical shape. Otherwise we're fighting the data.

## Why a Linear Structure Falls Apart

Given those constraints, the obvious first thought is a linked list. Simple, well understood. But there's a mismatch: linked lists are linear, window layouts are 2D.

When windows tile, they form a hierarchy; this half of the screen, that quarter, and so on. A list doesn't capture this naturally. Every time you add or remove a window, you'd have to recalculate all the rectangles from scratch. It works, but it's more math than I want to deal with.

## Trees Handle It Better

So if the problem is hierarchical, the structure should be too.

Trees fit naturally. The screen is the root, each split creates two children. Windows live in the leaves. The structure maps directly to what you see on screen

That's where BSP trees come in.

## Binary Space Partitioning

BSP trees are a specific kind of tree designed for recursive spatial division. The idea is simple: start with the full screen, split it in half (horizontal or vertical), keep splitting as needed. Each split point becomes an internal node, each final region becomes a leaf.

In my implementation:

- **Root node** = the entire screen/monitor rectangle
- **Internal nodes** = split regions (no window, just children)
- **External/leaf nodes** = actual windows

Each node can have zero or two children. Never one.

```
    Tree Node Types:

           I           ROOT (root is INTERNAL unless it's a single leaf)
          / \
         /   \
        I     I        INTERNAL NODES (screen sections/partitions)
       / \   / \
      /   \ /   \
     E    E E    E     EXTERNAL NODES (windows/leaves)

    - Internal Nodes (I): represent screen partitions, no window
    - External Nodes (E): hold actual windows
```

## The Node Structure

Here's the actual node definition from my code:

```c
typedef struct node_t node_t;
struct node_t {
    node_t     *parent;
    node_t     *first_child;
    node_t     *second_child;
    client_t   *client;           // the window, only for leaves
    rectangle_t rectangle;        // position and size
    rectangle_t floating_rectangle;
    node_type_t node_type;        // ROOT_NODE, INTERNAL_NODE, EXTERNAL_NODE
    split_type_t split_type;      // HORIZONTAL, VERTICAL, or DYNAMIC
    double      split_ratio;      // how to divide the space (0.0-1.0)
    bool        is_focused;
    bool        is_master;
};
```

I include a parent pointer. Some implementations skip it to save memory, but having it makes traversal and sibling access trivial. Worth the extra 8 bytes per node.

With the structure in place, let's look at the core operations: insertion and deletion.

## Insertion

When you open a new window, here's what happens:

1. Find the currently focused leaf node
2. Turn it into an internal node
3. Create two new leaf nodes as children
4. Move the existing window into the first child
5. Put the new window into the second child
6. Split the parent's rectangle between them

The split direction depends on the rectangle shape; if it's wider than tall, split vertically (side by side). Otherwise, split horizontally (top and bottom).

```c
void
insert_node(node_t *node, node_t *new_node, layout_t layout)
{
    if (node == NULL || node->client == NULL) {
        return;
    }

    // change node type to INTERNAL if it's not ROOT
    if (!IS_ROOT(node))
        node->node_type = INTERNAL_NODE;

    // move existing client to first child
    node->first_child = create_node(node->client);
    node->first_child->parent = node;
    node->first_child->node_type = EXTERNAL_NODE;
    node->client = NULL;

    // new window becomes second child
    node->second_child = new_node;
    node->second_child->parent = node;
    node->second_child->node_type = EXTERNAL_NODE;

    // split the rectangle
    split_node(node, new_node);
}
```

The splitting logic:

```c
static void
split_rect(node_t *n, split_type_t s)
{
    const int16_t gap   = conf.window_gap - conf.border_width;
    const int16_t pgap  = conf.window_gap + conf.border_width;
    const double  ratio = normalize_split_ratio(n->split_ratio);
    node_t       *n1    = n->first_child;
    node_t       *n2    = n->second_child;
    rectangle_t   nr    = n->rectangle;
    bool          h     = (s == HORIZONTAL_TYPE);

    // first child rectangle
    n1->rectangle.x      = nr.x;
    n1->rectangle.y      = nr.y;
    n1->rectangle.width  = (h) ? (nr.width - gap) * ratio : nr.width;
    n1->rectangle.height = (h) ? nr.height : (nr.height - gap) * ratio;

    // second child rectangle
    n2->rectangle.x      = (h) ? nr.x + n1->rectangle.width + pgap : nr.x;
    n2->rectangle.y      = (h) ? nr.y : nr.y + n1->rectangle.height + pgap;
    n2->rectangle.width  = (h) ? nr.width - n1->rectangle.width - gap : nr.width;
    n2->rectangle.height = (h) ? nr.height : nr.height - n1->rectangle.height - gap;
}
```

### Visual Example: Adding Windows

**Empty screen - null tree:**

```
    Screen                          BSP-tree
    +-----------------------+
    |                       |         [NULL]
    |        empty          |
    |                       |
    +-----------------------+
```

**First window - takes the whole screen:**

```
    Screen                          BSP-tree
    +-----------------------+
    |                       |       ROOT_NODE
    |       window 1        |      (window 1 &
    |      rectangle 1      |      rectangle 1)
    +-----------------------+
```

**Second window - screen splits, root becomes internal:**

```
    Screen                          BSP-tree
    +-----------------------+
    |           |           |          ROOT_NODE
    |           |           |         (rectangle 1)
    | window 1  | window 2  |           /     \
    |   rect 2  |   rect 3  |          /       \
    |           |           |     EXTERNAL  EXTERNAL
    |           |           |     (win 1,   (win 2,
    +-----------------------+     rect 2)   rect 3)

    - Screen divided into rectangle 1 (whole screen)
    - Window 1 -> rectangle 2, Window 2 -> rectangle 3
    - Both rect 2 and rect 3 are inside rect 1
```

**Third window - one side splits again:**

```
    Screen                          BSP-tree
    +-----------------------+
    |           |           |            ROOT_NODE
    |           |  window 2 |            (rect 1)
    |           |   rect 4  |            /      \
    | window 1  |-----------|           /        \
    |   rect 2  |           |      EXTERNAL   INTERNAL
    |           |  window 3 |      (win 1,    (rect 3)
    |           |   rect 5  |      rect 2)     /    \
    +-----------------------+                 /      \
                                         EXTERNAL EXTERNAL
                                         (win 2,  (win 3,
                                         rect 4)  rect 5)

    - rectangle 3 (right half) splits into rect 4 and rect 5
    - Window 2 moves to rect 4, Window 3 takes rect 5
```

**Fourth window:**

```
    Screen                              BSP-tree
    +-----------------------+
    |  window 1 |  window 2 |                ROOT_NODE
    |   rect 6  |   rect 4  |                (rect 1)
    |-----------|-----------|               /        \
    |  window 4 |  window 3 |              /          \
    |   rect 7  |   rect 5  |        INTERNAL       INTERNAL
    +-----------------------+        (rect 2)       (rect 3)
                                      /    \         /    \
                                     /      \       /      \
                                   EXT     EXT    EXT      EXT
                                  (w1,    (w4,   (w2,     (w3,
                                   r6)     r7)    r4)      r5)
```

Each internal node's rectangle contains its children. The tree structure directly mirrors the screen layout.

## Deletion

Insertion is the easy part. Deletion is where tree operations get interesting.

When you close a window, the node gets removed. But you can't just delete it as the tree needs to stay valid.

The key insight: when you remove a leaf, its parent (an internal node) now has only one child. That's not valid. So the sibling "takes over" the parent's position.

### Case 1: Simple deletion (sibling is external, parent is root)

```
    Before:                             After:
    +-----------------------+           +-----------------------+
    |           |           |           |                       |
    | window 1  |  window 2 | delete    |                       |
    |  Node B   |  Node C   | ------>   |  Node A (now has      |
    |           |  (delete) |           |     window 1)         |
    +-----------------------+           +-----------------------+

    BSP-tree Before:                    BSP-tree After:
        ROOT_NODE                           ROOT_NODE
         Node A                              Node A
         /    \                            (window 1)
        /      \
   EXTERNAL  EXTERNAL
    Node B    Node C
   (win 1)   (win 2)
             [DELETE]

    - Node C is deleted
    - Node B's client moves up to Node A
    - Node A becomes a leaf with window 1
```

### Case 2: Deletion with grandparent (sibling is external)

```
    Before:                              After:
    +-----------------------+            +-----------------------+
    |           |           |            |           |           |
    |           |  window 2 |            |           |           |
    | window 1  |  Node B   |  delete    | window 1  |  window 2 |
    |  Node EA  |-----------|  ------>   |  Node EA  |  Node A   |
    |           |  window 3 |            |           | (was B)   |
    |           |  Node C   |            |           |           |
    +-----------------------+            +-----------------------+

    BSP-tree Before:                     BSP-tree After:
         ROOT_NODE                            ROOT_NODE
          Node PA                              Node PA
          /     \                              /     \
         /       \                            /       \
    EXTERNAL  INTERNAL                   EXTERNAL  EXTERNAL
     Node EA   Node A                     Node EA   Node A
    (win 1)    /    \                    (win 1)   (win 2)
              /      \
         EXTERNAL EXTERNAL
          Node B   Node C
         (win 2)  (win 3)
                  [DELETE]

    - Delete Node C (window 3)
    - Its sibling Node B takes Node A's place
    - Node B becomes External directly under root
    - Free Node A and Node C
```

### Case 3: Deletion when sibling is internal

```
    Before:                              After update links:       Final state:
    +-----------------------+            +-----------------------+  +-----------------------+
    |           |           |            |           |           |  |           |           |
    |           |  window 2 |            |  window 2 |  window 3 |  |  window 2 |  window 3 |
    | window 1  |  Node B   |            |  node B   |  node C   |  |  node B   |  node C   |
    |  Node EA  |-----------|  ------>   |           |           |  |           |           |
    | [DELETE]  |  window 3 |            +-----------------------+  +-----------------------+
    |           |  Node C   |
    +-----------------------+

    BSP-tree:
    Initial:                   After unlink:               Final:
         ROOT_NODE                  ROOT_NODE                   ROOT_NODE
          Node PA                    Node PA                     Node PA
          /     \                        \                       /     \
         /       \                        \                     /       \
    EXTERNAL  INTERNAL               INTERNAL               EXTERNAL EXTERNAL
     Node EA   Node A                 Node A                 Node B   Node C
    (win 1)    /    \                 /    \                (win 2)  (win 3)
    [DELETE]  /      \               /      \
         EXTERNAL EXTERNAL      EXTERNAL EXTERNAL
          Node B   Node C        Node B   Node C
         (win 2)  (win 3)       (win 2)  (win 3)

    - Delete Node EA (window 1)
    - Its sibling Node A is internal, has children B and C
    - Node A's children (B, C) get promoted to Node PA's children
    - Free Node EA and Node A
```

Here's the code:

```c
bool
unlink_node(node_t *n, desktop_t *d)
{
    if (d == NULL || n == NULL) {
        return false;
    }

    // if node is root, tree becomes empty
    if (n->parent == NULL) {
        d->tree = NULL;
        return true;
    }

    node_t *parent  = n->parent;
    node_t *sibling = get_sibling(n);
    node_t *grandparent = parent->parent;

    sibling->parent = grandparent;
    if (grandparent) {
        // replace parent with sibling in grandparent's children
        if (grandparent->first_child == parent) {
            grandparent->first_child = sibling;
        } else {
            grandparent->second_child = sibling;
        }
    } else {
        // sibling becomes new root
        sibling->node_type = ROOT_NODE;
        d->tree = sibling;
    }

    // cleanup
    parent->first_child = NULL;
    parent->second_child = NULL;
    free(parent);
    n->parent = NULL;
    return true;
}
```

The sibling inherits the parent's rectangle, so after deletion, the remaining windows still fill the screen properly.

## Other Operations

Beyond insert/delete, there's a bunch of other stuff the tree needs to handle:

- **Finding nodes by window ID** - recursive search through the tree
- **Traversal** - next/previous window in the tree
- **Resizing** - adjusting split ratios and propagating changes down
- **Layout switching** - master/stack layouts recalculate all rectangles
- **Rendering** - BFS traversal to apply positions to actual X11 windows

The tree traversal uses a simple queue for BFS:

```c
int
render_tree(node_t *node)
{
    queue_t *q = create_queue();
    enqueue(q, node);

    while (q->front) {
        node_t *current = dequeue(q);
        if (!IS_INTERNAL(current) && current->client) {
            tile(current);  // apply position to X11 window
            continue;
        }
        if (current->first_child)
            enqueue(q, current->first_child);
        if (current->second_child)
            enqueue(q, current->second_child);
    }

    free_queue(q);
    return 0;
}
```

## Conclusion

BSP trees aren't complicated once you see them as just recursive space division. The tree structure matches the visual layout, which makes most operations straightforward. Insertion splits a rectangle, deletion merges back up. Add/remove windows, the tree adjusts.

The full implementation handles more edge cases (floating windows, fullscreen, gaps, borders, multiple monitors), but the core idea is what I described above. If you're interested, the code is at [github.com/yazeed1s/zwm](https://github.com/yazeed1s/zwm).
