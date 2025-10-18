# 数据结构

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

