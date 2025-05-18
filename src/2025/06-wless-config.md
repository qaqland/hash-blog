# Wless Config

配置文件主要分为三类，全部以环境变量的形式保存

## Output

**WLESS_OUTPUT_NAME**

显示器名称中的其它符号全部转为 `_`，配置格式为

```
分辨率[@{刷新率}][+{缩放比例}]
```

Example

```
WLESS_OUTPUT_DP_2=1920x1080+200%
WLESS_OUTPUT_HDMI_A_1=1920x1080@60
```

## Key

**WLESS_MAP_**
