# 环境与 MCP(vivado-mcp)

> 项目:Alinx 3EG | 板卡:`xazu3eg-sfvc784-1-i` | 工具:Vivado 2024.2 / vivado-mcp
> 状态:均已解决

## 1.1 多会话端口冲突 `port 9999 busy`

- **现象**:`vivado_start_session` 报 `vivado-mcp: port 9999 busy, exiting`,随后 MCP 命令报 `会话不存在`。
- **原因**:9999 是单实例端口,同一端口只能有一个 Vivado 会话。
- **解决**:先 `vivado_stop_session` 旧会话再启动新会话;要真正多开用 `port=0`(自动分配)或显式不同端口。

## 1.2 长任务 MCP 超时 ≠ 失败

- **现象**:`run_synthesis` / `run_implementation` / `generate_bitstream` 返回 `MCP error -32001: Request timed out`。
- **原因**:MCP 客户端等待超时,但 Vivado 里命令仍在后台执行。
- **解决**:不要重发;用 `vivado_get_run_progress(run_name)` 轮询 STATUS/PROGRESS。跑长任务优先用 Python 轮询型工具(`run_synthesis`/`run_implementation`),而非 `run_tcl`。

## 1.3 `run_tcl` 超时会中断后台命令

- **现象**:用 `run_tcl` 跑 `sw_loopback 115200 12` 并设 `timeout=3`,返回超时,但后台命令**没有继续完成**(后续发现数据未被处理)。
- **原因**:`run_tcl` 的超时会中断该命令,与综合/实现工具不同。
- **解决**:长命令给足够大的 `timeout` 同步等待,不要依赖"超时后继续跑"。

## 1.4 `report_*` 报 `No open design`

- **现象**:`report_utilization` / `report_timing_summary` 报 `ERROR: [Common 17-53] No open design`。
- **原因**:报告命令需要已打开的设计。
- **解决**:先 `open_run synth_1`(或 `impl_1`)再打报告。

## 1.5 IP 的 OOC 时钟周期警告

- **现象**:综合日志 `Clock period '10.000' ... different from the actual clock period '5.000'`。
- **原因**:IP 在默认时钟周期下做 out-of-context 综合的正常提示。
- **解决**:忽略,不影响。
