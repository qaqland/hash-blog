# TinyWL 源码学习记录

2025 年 5 月 22 日

## 数据结构 `wl_list`

`wl_list` 是标准的双向链表，其中第一个元素与最后一个元素首尾相连。使用中仅关心 `wl_list` 成员，不关注实际结构体类型：

```c
struct wl_list list_head;

struct element {
    int data;
    struct wl_list link;
}
```

对头节点来讲，需要显式手动初始化，双向指针指向自身：

```c
void wl_list_init(struct wl_list *list) {
    list->prev = list;
    list->next = list;
}

// wayland/src/wayland-util.c
```

因此对于头节点位置的插入、链表判空等操作均为 O(1) 复杂度。

对于元素，将结构体子成员 `struct wl_list link` 插入链并保存（代码略）。读取时使用库中提供的宏：

```c
#define wl_container_of(ptr, sample, member)    \
    (__typeof__(sample))((char *)(ptr) -        \
        offsetof(__typeof__(*sample), member))
```

- `sample` 是此处表示元素的指针类型，仅用做类型计算，因此传入 NULL 等皆可
- `ptr` 表示链表中保存的 `wl_list` 成员的指针
- `member` 表示元素的 `wl_list` 子成员的字段名

结构体成员的偏移量一般为非负的，所以拿 `ptr` 减去偏移量得到 `sample` 的地址；不同类型的指针做减法运算时，是以该类型大小为单位进行计算的，因此这里强转为 `char *` 占用 1 字节调整地址。

对链表的遍历库中有若干 helper：

```c
wl_list_for_each(pos, head, member)
wl_list_for_each_safe(pos, tmp, head, member)
wl_list_for_each_reverse(pos, head, member)
wl_list_for_each_reverse_safe(pos, tmp, head, member)
```

`pos` 和 `tmp` 表示元素的指针容器，`member` 与前文宏相同，是成员字段名，通常为 `link`。

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
