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

## 对窗口的移动和缩放

