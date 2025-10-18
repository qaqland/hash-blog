# Alt Tab 切换窗口的逻辑

2025年6月4日 这一篇日志是为了梳理 Wless 中窗口切换的代码实现逻辑

## 按键绑定

- `Alt` + `Tab`
- `Alt` + `Shift` + `Tab`
- `Win` + `Tab`
- `Win` + `Shift` + `Tab`

在 XKB 中这个键也叫 Super 键；注意 Shift 按下时，Tab 的枚举类型会发生改变。

## 按键过程

整个按键过程包括以下几个部分

1. 以 `Win` 或者 `Alt` 修饰键的按下为启动；
2. 以 `Tab` 的触发为窗口切换时机，可以多次触发，`Shift` 表示反向；
3. 最终以 `Win` 或者 `Alt` 的释放为完成。

## 窗口顺序

`wl_list` 双向链表（有环）保存所有 `ws-toplevel`，按照最近使用到最远使用的顺序保存。

```c
struct ws_toplevel *tmp_toplevel =
    s.win_toplevel
        ? s.win_toplevel
        : wl_container_of(s.toplevels.next, tmp_toplevel, link);
```

这里使用全局变量 `server.win_toplevel` 保存每次触发期间的暂存状态，当 `Alt` 或者 `Win` 释放时进行跳转（如果拿到的结果与当前窗口相同，应该会忽略）。

## 修饰键区别

使用 `Win` 修饰键切换窗口时，焦点屏幕（即 `focused_output`）将会保持固定，所有窗口依次切换到前台，**跳过**已出现在其它屏幕的窗口

> 考虑使用场景，用户把 Htop 固定在屏幕 A 当作仪表盘，使用此修饰键切换窗口时，可以避免轮换到此窗口。

使用 `Alt` 修饰键切换窗口时，若切换到的窗口已出现在其它屏幕，将会把焦点转移至该屏幕。此处逻辑与 Windows 等其它系统保持一致。

当用户仅有单块显示屏幕时，两种修饰键没有区别。

## 实现

对于 `Alt` 来说，不需要复杂的状态，每次取下一个窗口，参考 tinywl 中的实现：

```c
case XKB_KEY_F1:
    if (wl_list_length(&server->toplevels) < 2) {
        break;
    }
    struct tinywl_toplevel *next_toplevel =
        wl_container_of(server->toplevels.prev, next_toplevel, link);
    focus_toplevel(next_toplevel);
    break;
```

它这里做了极大的简化：

- 总是从最旧遍历到最新；
- 在最外层做的头节点判断；
- 每次都会修改链表状态。

我们需要额外针对头节点做处理，因为 Win 释放前 Tab 将会有多次改变，从上次切换窗口结束的 `server.win_topleve` 开始：

```c
struct wl_list *list = next ? tmp_toplevel->link.next
                : tmp_toplevel->link.prev;
if (list == &s.toplevels) {
    wlr_log(WLR_DEBUG, HEAD "skip head node");
    list = next ? list->next : list->prev;
}
```

对于单次 Tab，理论上应该只有一次遍历到头节点的机会（因为遍历的起始点总是会满足遍历需求），但是无法判断何时会到头节点。
