# Tcl 脚本

> 项目:Alinx 3EG | 环境:Vivado Tcl / PowerShell 5.1 / Windows COM
> 状态:均已解决

## 5.1 `too many nested evaluations (infinite loop?)`

- **现象**:串口驱动的 `serial::read` 等一调用就报此错。
- **根因**:命名空间内有与 **Tcl 内建命令同名**的 proc(`read`/`close`/`open`/`info`),内部未限定调用 `read $chan` 会解析到 `serial::read` 自身 → 无限递归。
- **解决**:内建命令一律加 `::` 前缀(`::read` / `::close` / `::open` / `::flush`);`serial::info` 改名 `serial::config` 避免遮蔽内建 `info`。

## 5.2 Vivado Tcl 打开 Windows 串口

- **可用**:`open "COM3:" r+` + `fconfigure $chan -mode 115200,n,8,1 -blocking 0 -buffering none -translation binary -encoding binary`。
- 报错信息含 `couldn't open serial "COM3:"`,说明 Tcl 识别为串口通道(非普通文件)。

## 5.3 PowerShell 脚本中文乱码 / 解析失败

- **现象**:`-File xxx.ps1` 报 `字符串缺少终止符` 或中文乱码。
- **根因**:PowerShell 5.1 默认按 GBK/ANSI 读取 `.ps1`,UTF-8(无 BOM)文件中的中文会破坏引号配对。
- **解决**:把 `.ps1` 存为 **UTF-8 with BOM**;或脚本内只用 ASCII 字符。

## 5.4 串口独占冲突

- **现象**:`permission denied` 打开 COM 失败。
- **根因**:串口同一时刻只能被一个进程持有(Tcl / PowerShell / 串口助手互斥)。
- **解决**:先关闭占用者(如 `SerialPortUtility`);用 `Get-Process` 排查占用进程。

## 5.5 串口句柄泄漏导致 COM 口一直 permission denied

- **现象**:某个串口脚本报错后,之后**任何**进程(Tcl / PowerShell / 串口助手)都打不开该 COM 口,一直 `permission denied`,但设备管理器里设备 `Status=OK`。杀掉占用进程也没用。
- **根因**:出错时 `serial::open` 已经把通道打开,但流程没走到 `serial::close`;随后若重新 `source` 驱动脚本,`variable chan ""` 会**重置**记录通道的变量,旧通道被孤立、再也关不掉,句柄一直被持有。
- **解决**:
  - **修复驱动**:变量初始化改为 `if {![info exists ::serial::chan]} { set ::serial::chan "" }`,重复 source 不重置。
  - **排查**:在持有句柄的进程里 `chan names` 列出通道并逐个 `chan close`。
  - **兜底**:结束持有句柄的进程(本项目实测是 **Vivado 进程**)即释放;或禁用/启用设备、重插 USB。
  - **定位**:`Get-Process vivado` 找到 Vivado 进程,`Stop-Process -Id <pid> -Force` 后 COM 口恢复。
