# hw 命令名版本差异(`open_hw` vs `open_hw_manager`)

- **日期**:2026-09-21
- **环境**:Vivado 2018.3(Tcl 8.5)
- **标签**:`#Vivado` `#版本差异` `#hardware-manager` `#Tcl`
- **状态**:已解决

## 现象

在 Vivado Tcl 里执行 `open_hw_manager`,报:

```
ERR invalid command name "open_hw_manager"
```

但 `get_hw_devices` 却能用。

## 根因

**硬件管理器的开关命令名随 Vivado 版本变了:**

| 版本 | 打开 | 关闭 |
| --- | --- | --- |
| **2019.1 及以上** | `open_hw_manager` | `close_hw_manager` |
| **2018.3 及更早** | `open_hw` | `close_hw` |

本例是 Vivado 2018.3,`info commands open_hw*` 返回的是 `open_hw_target open_hw`,所以要用 **`open_hw`**。用 `get_hw_devices` 能用,是因为查询类命令名一直没变,容易让人误以为 `open_hw_manager` 也该在。

## 排查技巧:先探命令是否存在

换工具/换版本时,别猜命令名,先问解释器:

```tcl
info commands open_hw*        ;# 列出实际存在的开关命令
info nameofexecutable         ;# 确认是不是 vivado.exe
version                       ;# Vivado 版本
info tclversion               ;# Tcl 版本
```

## 2018.3 完整硬件管理序列

```tcl
open_hw
connect_hw_server -url localhost:3121
open_hw_target
get_hw_devices
refresh_hw_device [lindex [get_hw_devices] 0]
```

烧 bit:

```tcl
current_hw_device [lindex [get_hw_devices] 0]
set_property PROGRAM.FILE {D:/path/top.bit} [current_hw_device]
program_hw_devices [current_hw_device]
```

ILA 调试(加载 `.ltx` probes):

```tcl
set_property PROBES.FILE      {D:/path/top.ltx} [current_hw_device]
set_property FULL_PROBES.FILE {D:/path/top.ltx} [current_hw_device]
refresh_hw_device -update_hw_probes true [current_hw_device]
```

关闭:`close_hw`。

## 说明

- `connect_hw_server`、`open_hw_target`、`get_hw_devices`、`program_hw_devices`、`set_property PROGRAM.FILE/PROBES.FILE`、`refresh_hw_device` 这些**在 2018.3 与 2019+ 都一样**,只有开/关管理器那一对名字不同。
- `localhost:3121` 是 **hw_server** 默认端口;连不上先起 `hw_server`。
