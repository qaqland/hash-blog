# PA-SINK-NEW

日志中有见到

```
sink.c: Created sink 0 "alsa_output.platform-analog_less_machine.hdmi-stereo" with sample...
```

- 来自文件 `src/pulsecore/sink.c` 的函数 `pa_sink_new`
- 来自文件 `src/modules/alsa/alsa-sink.c` 的函数 `pa_alsa_sink_new`
- 来自文件 `src/modules/alsa/module-alsa-card.c` 的
  - `init_profile` 函数（`pa__init` 函数）
  - `card_set_profile` 函数（模块中没用到）
- 来自文件 `src/modules/alsa/module-alsa-sink.c` 的函数 `pa__init`（模块未加载）

在这里从 module-card-restore 中 profile 的 restore 完成了 sink 的初始化

日志中同样有见到

```
module-default-device-restore.c: Restoring default sink 'alsa_output.platform-analog_less_machine.hdmi-stereo'.
core.c: configured_default_sink: (unset) -> alsa_output.platform-analog_less_machine.hdmi-stereo
core.c: default_sink: (unset) -> alsa_output.platform-analog_less_machine.hdmi-stereo
```

来自文件 `src/modules/module-default-device-restore`，
这个模块只在载入时进行一次 load，其他时间执行 save，帮助恢复默认 sink/source。

1. `pa_core_set_configured_default_sink`
函数修改了 `pa_core->configured_default_sink`，旧的是 `pa_core->default_sink`
2. `pa_core_update_default_sink` 判断是否存在、对比字符串、对比优先级。

上述过程没有实现对 sink 的创建。

`pa_cli_command_sink_default` 作为 pactl 的入口也有使用上述函数，
但是这里有额外的对 sink 是否存在的判断。

<!-- 明天试一下 sink 不存在时新建会怎么样 -->

DDE 大佬说他们用的是 `pa_context_set_sink_port_by_index`，这是个客户端持有的 API。
我们在 PA 内部使用，需要考虑下有无更直接的办法。

```c
command_set_sink_or_source_port
pa_sink *sink = pa_namereg_get(c->protocol->core, name, PA_NAMEREG_SINK)
pa_sink_set_port(sink, port, true)  // 这里没什么问题，PA 服务端也能用
```

需要去 sink 能创建的位置加点日志，或者在 module-default-device-restore 中监听新
sink 的创建，如果匹配就切换过去。逻辑从 module-device-restore 的
`sink_new_hook_callback` 中找。这个回调函数对应的日志是：


```
module-device-restore.c: Restoring port for sink %s.
module-device-restore.c: Not restoring port for sink %s, because already set.
```

但是开机日志中没有这一部分（TODO 为什么），只有：

```
module-device-restore.c: Restoring volume for sink alsa_output.plat
```

这是 `PA_CORE_HOOK_SINK_FIXATE` 的事件监听，这个事件来自 `pa_sink_new`，在
`PA_CORE_HOOK_SINK_NEW` 之后，日志 `Created sink %u "%s" with sample spec...`
之前。
当前不确定 sink 创建之后有没有被销毁，如果没有销毁为什么 `pactl list sinks`
中没有出现。

pactl 主要为 `pa_context` 传入了回调函数

```c
void get_sink_info_callback(pa_context *c, const pa_sink_info *i) {
    if (short_list_format) {
        printf(i->index, i->name);
    }
}

pa_context_get_sink_info_list(c, get_sink_info_callback, NULL);

void command_get_info_list(pa_pdispatch *pd, uint32_t command, uint32_t tag, pa_tagstruct *t) {
    if (command == PA_COMMAND_GET_SINK_INFO_LIST)
        i = c->protocol->core->sinks;
    if (i) {
        PA_IDXSET_FOREACH(p, i, idx) // 临时元素 容器数组 索引
            if (command == PA_COMMAND_GET_SINK_INFO_LIST)
                sink_fill_tagstruct(c, reply, p);
    }
}
```

