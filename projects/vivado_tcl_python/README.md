# vivado_tcl_python

- **简介**:Vivado 通过 Tcl 与 Python / MATLAB 持久交互,以及 Vivado 版本相关的命令差异
- **环境**:Vivado 2018.3(Tcl 8.5)/ Python 3.12 / MATLAB
- **关键词**:vivado-mcp、Tcl socket server、hw 命令、ADC 数据/FFT 链路

## 模块索引

| 模块 | 说明 |
| --- | --- |
| [宿主持久交互](01-宿主持久交互/Tcl-Socket-Server与客户端.md) | 在 Vivado 里开 Tcl server,Python/MATLAB 常驻连接 |
| [版本命令差异](02-版本命令差异/hw命令名版本差异.md) | `open_hw` vs `open_hw_manager` 等 hw 命令名差异 |

## 本项目常见问题

见上方模块记录。跨项目通用结论提炼到 [docs/现象速查表.md](../../docs/现象速查表.md)。
