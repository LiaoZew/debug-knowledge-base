# SPI 子系统(主机 + 从机)

> 项目:Alinx 3EG | 接口:SPI 主机/从机 + AXI4-Lite
> 状态:均已解决

## 9.1 从机同步链的边沿极性陷阱

- **现象**:SPI 主机↔从机回环,主机收到全 `0x00`。
- **根因**:从机用 3 级同步链 `s = {s[1:0], sig}` 检测边沿。**`s[2]` 是较旧抽头(延迟值),`s[1]` 是较新抽头**。把最新值当成了延迟值,导致 `rise`/`fall` 完全反了,所有边沿都不触发。
- **正解**:
  ```verilog
  wire rise = ~old &  new;   // old=s[2], new=s[1]
  wire fall =  old & ~new;
  ```
- **同理**:CS 的 `cs_fall = old & ~new`(不是 `new & ~old`)。

## 9.2 `sclk_reg` 的 FDCP Critical Warning

- **现象**:实现报 `[Netlist 29-358] Reg '.../spi_master/u_spi/sclk_reg' of type 'FDCP' cannot be timed accurately`。
- **根因**:复位分支写成 `sclk <= cpol;`——复位值依赖**信号**,Vivado 只能推断成「异步置位 + 异步复位」的 FDCP。
- **处理**:复位分支改成常量 `sclk <= 1'b0;`(IDLE 状态 `sclk <= cpol` 会立刻纠正)。或接受该告警(本例功能与静态时序均正常:WNS 5.2ns / WHS 0.02ns)。

## 9.3 往 BD 加第二/第三个 AXI 从设备

- `set_property CONFIG.NUM_MI {3} [get_bd_cells microblaze_0_axi_periph]` 后,新出现的 `M01_ACLK/M01_ARESETN/M02_*` **不会自动连时钟/复位**,需手动接:
  ```tcl
  connect_bd_net [get_bd_pins clk_wiz_0/clk_out1] [get_bd_pins microblaze_0_axi_periph/M01_ACLK]
  connect_bd_net [get_bd_pins rst_clk_wiz_0_100M/peripheral_aresetn] \
                 [get_bd_pins microblaze_0_axi_periph/M01_ARESETN]
  ```
- **坑**:`connect_bd_net` 的第一个参数用 **net 对象**(如 `[get_bd_nets microblaze_0_Clk]`)会静默不生效;要用**源引脚**(`clk_wiz_0/clk_out1`)。接不上会留下悬空网(如 `M01_ARESETN_1`),校验时报「not connected to a valid clock source」。
- 地址用 `set_property offset <addr> [get_bd_addr_segs /microblaze_0/Data/SEG_<cell>_reg0]` 固定,避免和已有外设重叠。

## 9.4 Vitis 应用新增源文件

- 生成的 `Debug/src/subdir.mk` 是「自动生成」,但新增 `.c` 后**不会**自动更新,链接报一堆 `undefined reference`。
- **处理**:手动把新文件加进 `C_SRCS` / `OBJS` / `C_DEPS` 三段(每段一行 `\` 续行),再 `make -B all`。或者用 xsct/vitis 重新生成 makefile。

## 9.5 SPI 改成"整帧"(地址段+数据段)后 LSB 丢失

- **现象**:仿真全过,上板 **每个字节的 bit0 恒为 0**(`0x11` 读成 `0x10`,`0x33`→`0x32`)。
- **根因(两个叠加)**:
  1. 从机在**字节提交**时就把 `miso_out` 改成下一字节的 MSB——而这个时刻落在主机 `S_H1` 采样窗口的**中途**。
  2. 主机在 `S_H1` 里**每个周期都采样**(后值覆盖前值),于是最后一次采到的是已经变成"下一字节 MSB"的电平 → 本字节的 LSB 被吞掉。
- **修复**:
  - 从机:字节提交**不改** `miso_out`(下一字节 MSB 交给随后的换数据沿 `chg_edge` 驱动)。
  - 主机:每个相位**只在末尾采一次** —— CPHA=0 在 `S_H1` 的 `cnt==div-1`,CPHA=1 在 `S_H2` 的 `cnt==div-1`(既过了从机驱动时刻,又早于换数据)。
- **教训(重要)**:原 testbench 用 `clk_div=5`(SCLK 半周期仅 5 拍),与从机 3 拍同步链延迟同量级,**恰好把时序 bug 掩盖了**。实机 `clk_div=50` 才暴露。
  → **TB 的参数要贴近实机**(本项目已改为 50)。

## 9.6 往 BD 加第 4 个外设 / IP 源改了要重新打包

- 本工程最终 4 个外设:UART `0x20000`、SPI_M `0x30000`、SPI_S `0x40000`、FREQ `0x50000`。
- IP 源文件改动后必须 `package_*_ip.tcl` 重新打包,然后在工程里 `update_ip_catalog -rebuild` + `upgrade_ip`(BD 里该 IP 会显示 **locked**,不升级则仍用旧网表 —— 表现为改了 RTL 但综合结果不变)。
