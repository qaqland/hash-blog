# TinyWL

2025 年 5 月 22 日

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

// TODO
