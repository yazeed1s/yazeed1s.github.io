+++
title = "Function Pointers as Poor Man's Polymorphism"
date = 2026-01-12
description = "How C uses structs of function pointers to get dispatch without inheritance."
[taxonomies]
tags = ["C", "systems programming", "patterns"]
+++

C doesn't have interfaces, virtual methods, or inheritance. But it has function pointers. And a struct of function pointers is basically an interface.

This pattern is everywhere. The Linux kernel uses it for everything. I used it in ZWM without thinking about it as polymorphism until later.

## the basic pattern

Say you have different types that need to support the same operations. In C++ you'd make a base class with virtual methods. In C you make a struct of function pointers.

```c
typedef struct {
    int  (*open)(const char *path);
    int  (*read)(void *buf, size_t n);
    int  (*write)(const void *buf, size_t n);
    void (*close)(void);
} io_ops_t;
```

Then different implementations fill in different functions:

```c
static int file_open(const char *path)  { /* open a file */ }
static int file_read(void *buf, size_t n)  { /* read from file */ }
// ...

static int sock_open(const char *path)  { /* connect to address */ }
static int sock_read(void *buf, size_t n)  { /* recv from socket */ }
// ...

io_ops_t file_ops = { file_open, file_read, file_write, file_close };
io_ops_t sock_ops = { sock_open, sock_read, sock_write, sock_close };
```

Code that uses `io_ops_t` doesn't know if it's talking to a file or a socket. It just calls `ops->read(buf, n)`. That's polymorphism. No vtable keyword, no `virtual`, just a struct and some function pointers.

## the kernel does this constantly

The Linux kernel's VFS layer is built on this. Every filesystem implements a `struct file_operations`:

```c
struct file_operations {
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    int     (*open)(struct inode *, struct file *);
    int     (*release)(struct inode *, struct file *);
    int     (*mmap)(struct file *, struct vm_area_struct *);
    long    (*ioctl)(struct file *, unsigned int, unsigned long);
    // ... many more
};
```
You can find the full definition in `linux/fs.h`.

ext4 fills in ext4 functions. tmpfs fills in tmpfs functions. procfs fills in procfs functions. When you call `read()` from user space, the syscall handler looks up which `file_operations` struct is attached to your fd and calls the right `.read`. The VFS doesn't care what filesystem is underneath.

Same idea with `struct inode_operations`, `struct address_space_operations`, `struct net_device_ops`. The kernel is basically a giant collection of these interface structs.

## a simpler example: event dispatch

In ZWM I needed to dispatch X11 events to handler functions. Each event type maps to a function. The table-driven approach:

```c
typedef int (*event_handler_t)(xcb_generic_event_t *event);

typedef struct {
    uint8_t         type;
    event_handler_t handle;
} event_handler_entry_t;

static const event_handler_entry_t _handlers_[] = {
    { XCB_MAP_REQUEST,       handle_map_request },
    { XCB_UNMAP_NOTIFY,      handle_unmap_notify },
    { XCB_DESTROY_NOTIFY,    handle_destroy_notify },
    { XCB_CLIENT_MESSAGE,    handle_client_message },
    { XCB_CONFIGURE_REQUEST, handle_configure_request },
    { XCB_ENTER_NOTIFY,      handle_enter_notify },
    { XCB_KEY_PRESS,         handle_key_press },
};
```

Dispatch is a simple loop:

```c
for (size_t i = 0; i < n; i++) {
    if (_handlers_[i].type == event_type) {
        return _handlers_[i].handle(event);
    }
}
```

This isn't a struct-of-function-pointers in the interface sense, but it's the same idea. Data drives the dispatch instead of a switch statement. Adding a new event handler means adding a row, not modifying control flow.

A switch statement would work fine here. But the table is easier to read when you have many entries, and you can build it at runtime if you need to.

## the vtable pattern

Sometimes you want the function pointers attached to each instance, not shared globally. C++ does this automatically with virtual methods. In C you do it manually:

```c
typedef struct shape_t shape_t;

typedef struct {
    float (*area)(const shape_t *self);
    void  (*draw)(const shape_t *self);
} shape_ops_t;

struct shape_t {
    const shape_ops_t *ops;
    // shape data follows
};
```

Each shape type defines its own ops:

```c
typedef struct {
    shape_t base;
    float   radius;
} circle_t;

static float circle_area(const shape_t *self) {
    circle_t *c = (circle_t *)self;
    return 3.14159f * c->radius * c->radius;
}

static const shape_ops_t circle_ops = { circle_area, circle_draw };
```

Now any code that has a `shape_t *` can call `shape->ops->area(shape)` and it works for circles, rectangles, whatever. The cast from `shape_t *` to `circle_t *` works because `base` is the first member.

This is exactly what a C++ vtable is. The compiler generates the struct of function pointers and the dispatch. In C you write it yourself.

## where this falls apart

No type safety on the casts. You cast `shape_t *` to `circle_t *` and the compiler trusts you. Pass the wrong type and you get garbage, not a compile error.

No compiler enforcement that you filled in all the function pointers. Leave one NULL and you segfault at runtime. C++ gives you pure virtual and abstract classes. C gives you nothing. You can add runtime checks (`assert(ops->read != NULL)`) but it's manual.

Function signature mismatches are silent if you're not careful. If `ops->read` expects three arguments and you assign a function that takes two, some compilers will warn, some won't. Depends on how you typedef things.

And the `self` pointer pattern is verbose. Every method takes a `self` parameter. Every call passes it. C++ hides this with `this`. C makes you type it every time.

## it's still worth it

The pattern is still useful though. It gives you the main thing polymorphism provides: code that operates on behavior rather than concrete types. The VFS layer works because it dispatches through function pointers. Device drivers work the same way. My event loop works the same way.

You don't get compile-time safety. But you get a clean separation between interface and implementation in a language that has no keyword for either.

## notes

- GLib (GTK's base library) builds an entire object system on this. GObject has inheritance, interfaces, properties, signals. All in C. It's heavyweight but it works.
- The Linux kernel's `container_of` macro is how you go from the embedded base struct back to the containing struct. It's pointer arithmetic on struct offsets.
- SQLite uses this pattern for its VFS layer too. You can plug in custom file systems by providing a struct of function pointers.
- If you only need dispatch and not per-instance state, a simple function pointer array indexed by type is simpler than the vtable pattern.
