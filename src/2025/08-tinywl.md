# TinyWL 源码学习记录

2025 年 5 月 22 日 后面补充上了一点 dwl 的东西

## Why are there "key" and "modifiers" two events at the same time?

根据协议，key 表示物理按键序列，而 modifiers 用来同步当前修饰键的逻辑状态

1. 客户端 enter 之后会收到一次 modifiers 同步
2. 修饰键 key 触发后也会收到 modifiers 状态激活

在 `wlroots/types/seat/wlr_seat_keyboard.c` 中可以看到调用顺序：

- `wlr_seat_keyboard_notify_enter`
- `wlr_seat_keyboard_enter`
- `wlr_seat_keyboard_send_modifiers`

第二种情况暂未发现 // TODO

## 对窗口的移动

客户端主动发出请求 `xdg_toplevel::move`，服务端在 `xdg_toplevel_request_move` 中处理

```c
server->grabbed_toplevel = toplevel;
server->cursor_mode = TINYWL_CURSOR_MOVE;

server->grab_x = server->cursor->x - toplevel->scene_tree->node.x;
server->grab_y = server->cursor->y - toplevel->scene_tree->node.y;
```

处理该请求主要为记录状态，包括希望移动的窗口、触发时的鼠标位置等。后续具体移动逻辑由服务端在鼠标事件中负责：

```c
*toplevel = server->grabbed_toplevel;

wlr_scene_node_set_position(
    &toplevel->scene_tree->node,
    server->cursor->x - server->grab_x,
    server->cursor->y - server->grab_y
);
```

使用“鼠标的实时位置”与请求触发时“窗口顶点与鼠标的相对距离”设置移动过程与最终的窗口位置。

## 对窗口的尺寸调整

与移动相似，但窗口需要更多调整，涉及到窗口的大小、位置改变。

- 记录光标位置与窗口边缘坐标
- 注意窗口最小尺寸（当调整左上边框时）
- 计算得到新窗口大小并设置
- 计算得到新窗口坐标并设置

对于坐标来说，`wlr_xdg_toplevel` 的坐标不等于 `wlr_surface` 的坐标，因此需要考虑两者之间的相对值。

## 键盘状态-焦点 surface

```c
struct wlr_surface* previous_surface = seat->keyboard_state.focused_surface;

// tinywl.c @focuse_toplevel
```

在 `wlroots` 的 seat 中这个状态鼠标键盘各有一个：

```c
struct wlr_surface *pointer_focus = wlr_seat->pointer_state.focused_surface;
if (pointer_focus != NULL &&
        wl_resource_get_client(pointer_focus->resource) == client) {
    wlr_seat->pointer_state.focused_client = seat_client;
}

struct wlr_surface *keyboard_focus = wlr_seat->keyboard_state.focused_surface;
if (keyboard_focus != NULL &&
        wl_resource_get_client(keyboard_focus->resource) == client) {
    wlr_seat->keyboard_state.focused_client = seat_client;
}

// types/seat/wlr_seat.c @static struct wlr_seat_client *seat_client_create()
```

当 `wlr_seat` 没有资源时，持有的焦点也要销毁（不过一般一个 WM 只有一个实例）

```c
if ((capabilities & WL_SEAT_CAPABILITY_POINTER) == 0) {
    if (focused_client != NULL && focused_surface != NULL) {
        seat_client_send_pointer_leave_raw(focused_client, focused_surface);
    }
}

// types/seat/wlr_seat.c @void wlr_seat_set_capabilities()
```

所以这个变量可以为空，仅表示当前 seat 相关的焦点状态。常规修改发生在一些 enter 事件上：

- `wlr_seat_keyboard_enter`
- `wlr_seat_pointer_enter`

不过注释说更应该使用 `wlr_seat_*_notify_enter`，内部会自行调整。

## `focus_toplevel`

主要在以下三个位置调用：

- `handle_keybinding` 做窗口轮换时重新获得焦点
- `server_cursor_button` 鼠标点击事件
- `xdg_toplevel_map` 新窗口生成时拿到焦点

对于这三种情况基本可以满足 `focuse_output` 的等效替换，不过要考虑不可最大化的窗口处理

// TODO

## `wlr_scene_tree` 的创建

在 `dwl.c` 的 `mapnotify` 函数中，有以下创建：

```c
c->scene = client_surface(c)->data = wlr_scene_tree_create(layers[LyrTile]);

c->scene_surface = c->type == XDGShell
    ? wlr_scene_xdg_surface_create(c->scene, c->surface.xdg)
    : wlr_scene_subsurface_tree_create(c->scene, client_surface(c));
c->scene->node.data = c->scene_surface->node.data = c;
```

这里的 `scene` `scene_tree` 类型均为 `wlr_scene_tree` 指针。对比两种函数：

```c
/**
 * Add a node displaying an xdg_surface and all of its sub-surfaces to the
 * scene-graph.
 *
 * The origin of the returned scene-graph node will match the top-left corner
 * of the xdg_surface window geometry.
 */
struct wlr_scene_tree *wlr_scene_xdg_surface_create(
	struct wlr_scene_tree *parent, struct wlr_xdg_surface *xdg_surface);

/**
 * Add a node displaying a surface and all of its sub-surfaces to the
 * scene-graph.
 */
struct wlr_scene_tree *wlr_scene_subsurface_tree_create(
	struct wlr_scene_tree *parent, struct wlr_surface *surface);
```

// 没有太大差别，XDG 和 XWAYLAND 不适配

