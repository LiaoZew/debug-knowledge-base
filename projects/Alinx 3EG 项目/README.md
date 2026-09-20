# Alinx 3EG 项目

- **简介**:Alinx AXU3EG / ACU3EG 开发板的 FPGA / Vivado / UART / SPI / MicroBlaze 调试记录
- **板卡**:Alinx AXU3EG / ACU3EG(`xazu3eg-sfvc784-1-i`)
- **工具链**:Vivado 2024.2,Windows / Vitis / xsct
- **原始记录**:工程内 `alinx_3eg_demo/DEBUG_NOTES.md`(保留未动,本目录为其结构化拆分)

## 模块索引

| 模块 | 说明 | 记录数 |
| --- | --- | --- |
| [环境与 MCP](01-环境与MCP/环境与MCP.md) | vivado-mcp 会话、端口、超时、报告命令 | 5 |
| [综合实现与时序](02-综合实现与时序/综合实现与时序.md) | 综合后 Hold 假象、烧板前兜底 | 2 |
| [RTL 逻辑问题](03-RTL逻辑问题/RTL逻辑问题.md) | 消抖、UART 极性与丢字节、重复发送、固件回显 | 5 |
| [烧板与在线调试](04-烧板与在线调试/烧板与在线调试.md) | bit/ltx、hw_axi/hw_vio、VIO 使能边沿检测 | 6 |
| [Tcl 脚本](05-Tcl脚本/Tcl脚本.md) | 无限递归、串口打开、PowerShell 编码、句柄泄漏 | 5 |
| [硬件与引脚](06-硬件与引脚/硬件与引脚.md) | USB 口、差分时钟 IOSTANDARD、引脚权威来源 | 3 |
| [MicroBlaze 软核与 Vitis](07-MicroBlaze软核与Vitis/MicroBlaze软核与Vitis.md) | 复位、时钟宏、LMB、BD automation | 5 |
| [SPI 子系统](08-SPI子系统/SPI子系统.md) | 边沿极性、FDCP 告警、AXI 从设备、LSB 丢失 | 6 |
| [频率计](09-频率计/频率计.md) | 原理精度、使用限制、自测、仿真 | 4 |
| [宽范围频率计](10-宽范围频率计/宽范围频率计.md) | 预分频、周期数、CDC 时序、高频源 | 6 |

## 本项目常见问题

见各模块文件,跨项目通用结论已提炼到 [docs/现象速查表.md](../../docs/现象速查表.md)。
