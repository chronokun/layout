Layout
======

A simple/fast stacking box layout library. It's useful for calculating layouts
for things like 2D user interfaces. It compiles as C99 or C++. It's tested with
gcc (mingw64), VS2015, and clang/LLVM. There are only two important files,
[layout.h](layout.h) and [layout.c](layout.c).

![](https://raw.githubusercontent.com/wiki/randrew/layoutexample/ui_anim_small.gif)

*Layout* has no external dependencies, but does use `stdlib.h` and `string.h`
for `realloc` and `memset`. If your own project does not or cannot use these,
you can easily exchange them for something else. Only a few lines will need to
be edited.

*Layout* comes with a small set of tests as a build target, along with a
primitive benchmark and example usage of the library as a Lua .dll module.
However, if you want to use *Layout* in your own project, you can probably copy
[layout.h](layout.h) and [layout.c](layout.c) into your project's source tree
and use your own build system. You don't have to separately build *Layout* as a
shared library and link against it.

Building the tests, benchmarks and Lua module are handled by
[GENie](https://github.com/bkaradzic/GENie), but no executable binaries for
GENie are included in this source repository. [Download links for binary builds
of GENie are listed below](#download-genie). You will need to download (or
build yourself) a GENie executable and place it in your path or at the root of
this repository tree. If you want to build on a platform other than Windows,
you'll likely need to modify [genie.lua](genie.lua) for compatibility. Feel
free to open issues or pull requests.

*Layout* is based on the nice library
[oui](https://bitbucket.org/duangle/oui-blendish) by
[duangle](https://twitter.com/duangle). Unlike *oui*, *Layout* does not handle
anything related to user input, focus, or UI state.

Building
========

If you just want to use *Layout* in your own project, you can simply copy
[layout.h](layout.h) and [layout.c](layout.c) directly into your project. Take
a look at [genie.lua](genie.lua) for some recommended compiler and linker
options.

If you want to build *Layout*'s tests, benchmarks, or example Lua module, you
will first need to get (or make) a GENie binary and place it in your path or at
the root of this repository.

Download GENie
--------------

Linux:  
https://github.com/bkaradzic/bx/raw/master/tools/bin/linux/genie

OSX:  
https://github.com/bkaradzic/bx/raw/master/tools/bin/darwin/genie

Windows:  
https://github.com/bkaradzic/bx/raw/master/tools/bin/windows/genie.exe

Visual Studio 2015
------------------

```
genie.exe vs2015
start build/vs2015/layout.sln
```

GCC/MinGW/Clang
---------------

```
./genie gmake
```

and then run your `make` in the directory `build/gmake`. You will need to
specify a target and config. Here is an example for building the `tests` target
in Windows with the 64-bit release configuration using mingw64 in a bash-like
shell (for example, git bash):


```
./genie.exe gmake && mingw32-make.exe -C build/gmake tests config=release64
```

Options
-------

You can choose to build *Layout* to use either integer (int16) or floating
point (float) coordinates. Integer is the default, because UI and other 2D
layouts do not often use units smaller than a single point when aligning and
positioning elements. You can choose to use floating point instead of integer
by defining `LAY_FLOAT`. If you are building the tests, benchmarks, or Lua
module for *Layout*, you can configure this when you invoke GENie:

```
./genie gmake --coords=float
```

or if you want to specify integer (the default):

```
./genie gmake --coords=integer
```

If you are building *Layout* to use floating point coordinates, and if you want
to enforce SSE alignment for the vector coordinate types, you'll probably want
to add or tweak alignment specifiers for some types (or typedef them to m128)
in [layout.h](layout.h). The code is simple and should be easy to modify for
your purposes. If you do this, and if you also change the code to use your own
custom allocator, you might also need to guarantee that the starting addresses
of the buffers given to *Layout* by your allocator are 16-byte aligned. If you
do all of that, and if you build *Layout* as a shared library or a static
library without linker optimizations enabled in MSVC, you might want to also
consider using `__vectorcall`, which may help reduce overhead when calling
functions which receive or return SSE values.

Example
=======

```C
#include "layout.h"

// Let's pretend we're creating some kind of GUI with a master list on the
// left, and the content view on the right.

// We first need one of these
lay_context ctx;

// And we need to initialize it
lay_init_context(&ctx);

// The context will automatically resize its heap buffer to grow as needed
// during use. But we can avoid multiple reallocations by reserving as much
// space as we'll need up-front. Don't worry, lay_init_context doesn't do any
// allocations, so this is our first and only alloc.
lay_reserve_items_capacity(&ctx, 1024);

// Create our root item. Items are just 2D boxes.
lay_id root = lay_item(&ctx);

// Let's pretend we have a window in our game or OS of some known dimension.
// We'll want to explicitly set our root item to be that size.
lay_set_size_xy(&ctx, root, 1280, 720);

// Set our root item to arrange its children in a row, left-to-right, in the
// order they are inserted.
lay_set_contain(&ctx, root, LAY_ROW);

// Create the item for our master list.
lay_id master_list = lay_item(&ctx);
lay_insert(&ctx, root, master_list);
// Our master list has a specific fixed width, but we want it to fill all
// available vertical space.
lay_set_size_xy(&ctx, master_list, 400, 0);
// We set our item's behavior within its parent to desire filling up available
// vertical space.
lay_set_behave(&ctx, master_list, LAY_VFILL);
// And we set it so that it will lay out its children in a column,
// top-to-bottom, in the order they are inserted.
lay_set_contain(&ctx, master_list, LAY_COLUMN);

lay_id content_view = lay_item(&ctx);
lay_insert(&ctx, root, content_view);
// The content view just wants to fill up all of the remaining space, so we
// don't need to set any size on it.
//
// We could just set LAY_FILL here instead of bitwise-or'ing LAY_HFILL and
// LAY_VFILL, but I want to demonstrate that this is how you combine flags.
lay_set_behave(&ctx, content_view, LAY_HFILL | LAY_VFILL);

// Normally at this point, we would probably want to create items for our
// master list and our content view and insert them. This is just a dumb fake
// example, so let's move on to finishing up.

// Run the context -- this does all of the actual calculations.
lay_run_context(&ctx);

// Now we can get the calculated size of our items as 2D rectangles. The four
// components of the vector represent x and y of the top left corner, and then
// the width and height.
lay_vec4 master_list_rect = lay_get_rect(&ctx, master_list);
lay_vec4 content_view_rect = lay_get_rect(&ctx, content_view);

// master_list_rect  == {  0, 0, 400, 720}
// content_view_rect == {400, 0, 880, 720}

// If we're using an immediate-mode graphics library, we could draw our boxes
// with it now.
my_ui_library_draw_box_x_y_width_height(
    master_list_rect[0],
    master_list_rect[1],
    master_list_rect[2],
    master_list_rect[3]);

// You could also recursively go through the entire item hierarchy using
// lay_first_child and lay_next_sibling, or something like that.

// After you've used lay_run_context, the results should remain valid unless a
// reallocation occurs.
//
// However, while it's true that you could manually update the existing items
// in the context by using lay_set_size{_xy}, and then calling lay_run_context
// again, you might want to consider just rebuilding everything from scratch
// every frame. This is a lot easier to program than tedious fine-grained
// invalidation, and a context with thousands of items will probably still only
// take a handful of microseconds.
//
// There's no way to remove items -- once you create them and insert them,
// that's it. If we want to reset our context so that we can rebuild our layout
// tree from scratch, we use lay_reset_context:

lay_reset_context(&ctx);

// And now we could start over with creating the root item, inserting more
// items, etc. The reason we don't create a new context from scratch is that we
// want to reuse the buffer that was already allocated.

// But let's pretend we're shutting down our program -- we need to destroy our
// context.
lay_destroy_context(&ctx);

// The heap-allocated buffer is now freed. The context is now invalid for use
// until lay_init_context is called on it again.
```
