+++
title = "BSP Trees for Tiling Window Management"
date = 2024-07-21
description = "How I use binary space partitioning to manage windows in my X11 window manager."
[taxonomies]
tags = ["C", "spatial data structures", "tiling window manager"]
+++

At some point I decided to build my own tiling window manager from scratch. I was working with spatial data structures at work, BSP trees looked like a good fit for this kind of problem, and it also sounded like a fun challenge.

## what's a tiling window manager?

A tiling window manager arranges windows so they don't overlap, so instead of floating windows on top of each other like macOS or Windows it splits the screen into sections, where you open a window and it goes into a tile, open another and the screen splits, and you don't manually position anything.

Linux has a bunch of these like dwm, i3, and bspwm, and on Mac there's Yabai. They're not mainstream, but if you're into keyboard-driven workflows they're nice. The question I care about here is how you represent this layout in memory.

## what the layout data structure needs to do

Before picking a data structure it helps to think about what operations matter. At minimum a tiling window manager needs to insert a new window when one opens and split the focused region, delete a window when it closes and reclaim the space, resize regions by adjusting split ratios, traverse the layout to find the next or previous window, and render by walking the structure and applying positions to X11.

The shape of the data matters too, since window layouts are inherently hierarchical and spatial, where the screen splits into halves and those halves split again, so it's not a flat sequence at all. Whatever structure you pick should make the common operations cheap and match this hierarchical shape, otherwise you're fighting the data.

## why a linear structure falls apart

Given those constraints the obvious first thought is a linked list since it's simple and well understood, but there's a mismatch underneath, which is that linked lists are linear and window layouts are 2D. When windows tile they form a hierarchy, this half of the screen, that quarter, and so on, and a list doesn't capture that naturally, so every time you add or remove a window you'd have to recalculate all the rectangles from scratch, which works but it's more math than I want to deal with.

## trees handle it better

If the problem is hierarchical the structure should be too, and trees fit naturally, where the screen is the root, each split creates two children, windows live in the leaves, and the structure maps directly to what you see on screen. That's where BSP trees come in.

## binary space partitioning

BSP trees are a specific kind of tree built for recursive spatial division, so you start with the full screen, split it in half horizontally or vertically, and keep splitting as needed, where each split point becomes an internal node and each final region becomes a leaf. In my implementation the root node is the entire screen or monitor rectangle, internal nodes are split regions with no window and just children, and external or leaf nodes are the actual windows. Each node has either zero or two children, never one.

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

## the node structure

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

I include a parent pointer because some implementations skip it to save memory, but having it makes traversal and sibling access trivial and worth the extra 8 bytes per node. With the structure in place, the next parts are insertion and deletion.

## insertion

When you open a new window the manager finds the currently focused leaf node, turns it into an internal node, creates two new leaf nodes as children, moves the existing window into the first child, puts the new window into the second child, and splits the parent's rectangle between them. The split direction depends on the rectangle shape, so if it's wider than tall it splits vertically side by side, and otherwise it splits horizontally into top and bottom.

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

### visual example, adding windows

**Empty screen, null tree:**

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

## deletion

Insertion is the easy part, deletion is where tree operations get interesting. When you close a window the node gets removed, but you can't just delete it because the tree needs to stay valid, since when you remove a leaf its parent internal node now has only one child, which isn't valid, so the sibling takes over the parent's position.

### case 1, simple deletion (sibling is external, parent is root)

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

### case 2, deletion with grandparent (sibling is external)

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

### case 3, deletion when sibling is internal

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

The sibling inherits the parent's rectangle, so after deletion the remaining windows still fill the screen properly.

## other operations

Beyond insert and delete there's a bunch of other stuff the tree handles, like finding nodes by window ID with a recursive search, traversal to the next or previous window, resizing by adjusting split ratios and propagating changes down, layout switching where master and stack layouts recalculate all rectangles, and rendering with a BFS traversal that applies positions to the actual X11 windows. The traversal uses a simple queue for the BFS:

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

## notes

BSP trees aren't complicated once you see them as just recursive space division, where the tree structure matches the visual layout and makes most operations straightforward, so insertion splits a rectangle, deletion merges back up, and the tree adjusts as windows come and go. The full implementation handles more edge cases like floating windows, fullscreen, gaps, borders, and multiple monitors, but the core idea is what I described here. Code is at [github.com/yazeed1s/zwm](https://github.com/yazeed1s/zwm).
