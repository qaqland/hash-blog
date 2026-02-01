# TinyWL 源码学习记录

2025 年 5 月 22 日 后面补充上了一点 dwl 的东西

## 双向链表 `wl_list`

`wl_list` 是 Wayland 中最常用的数据结构，其中包含两个指针分别指向前后节点：

```c
struct wl_list {
    struct wl_list *prev;
    struct wl_list *next;
};
```

一般区分头节点和内容节点。对头节点来说，需要显式手动初始化，让双向指针指向自身：

```c
void wl_list_init(struct wl_list *list) {
    list->prev = list;
    list->next = list;
}

// wayland/src/wayland-util.c
```

`wl_list` 的插入、判空等操作均为 O(1) 复杂度，但获得长度需要遍历全部节点。

对于内容节点，`wl_list` 作为结构体的一个成员，相关 method 仅关心 `wl_list` 成员指针，不关注完整结构体：

```c
struct wl_list list_head;

struct list_element {
    int data;
    struct wl_list link;
}
```

将结构体成员 `struct wl_list link` 插入链并保存（代码略）。读取时使用库中提供的宏：

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

## 动态数组 `wl_array`

动态数组在 Wayland Compositor 这边使用较少，作为一个容器像其他语言中的 Vector 般类似工作，但它只能增长，非常适合初始化按键绑定等一次性配置工作。

```c
struct wl_array {
    size_t size;
    size_t alloc;
    void *data;
};

// wayland/src/wayland-util.c
```

由于 `void *data` 不保存实际类型，所有数据理论上全部叠加堆放，因此必须数组元素大小相同才能安全遍历：

```c
struct ws_key *key;
wl_array_for_each(key, &server->keys) {
    // 这里 key 就是拿到的元素
}
```

```c
struct ws_key *key;
key = wl_array_add(&server->keys, sizeof(*key));
// 这里 key 就是拿到的元素容器
```

## 信号槽 `wl_signal`

信号槽是 `wl_list` 的封装，每次执行 `wl_signal_add` 会在链表尾部追加监听器。

```c
struct wl_signal {
    struct wl_list listener_list;
};

static inline void
wl_signal_add(struct wl_signal *signal, struct wl_listener *listener)
{
	wl_list_insert(signal->listener_list.prev, &listener->link);
}
```

当信号触发时，`emit` 函数遍历监听器链表，依次调用监听器中绑定的回调函数：

```c
static inline void
wl_signal_emit(struct wl_signal *signal, void *data)
{
	struct wl_listener *l, *next;

	wl_list_for_each_safe(l, next, &signal->listener_list, link)
		l->notify(l, data);
}

// wlroots 库中使用更安全的 wl_signal_emit_mutable()，原理相同。
```

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

