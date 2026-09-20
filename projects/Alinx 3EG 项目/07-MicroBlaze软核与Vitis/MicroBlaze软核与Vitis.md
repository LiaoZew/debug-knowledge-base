# MicroBlaze 软核 / Vitis

> 项目:Alinx 3EG | 工具:Vivado 2024.2 BD / Vitis / xsct
> 状态:均已解决

## 7.1 下载 ELF 报 "MicroBlaze is held in reset"(`ext_reset_in` 悬空)

- **现象**:`xsct` 里 `dow` 报 `Cannot stop MicroBlaze. MicroBlaze is held in reset`,`state` = `Reset`。
- **根因**:`proc_sys_reset` 的 `ext_reset_in` 是**低有效**(`C_EXT_RESET_HIGH=0`),而 BD 对**未连接**的输入按 **0** 处理 → 等于**一直断言复位** → MicroBlaze 被永久复位。
- **解决**:`ext_reset_in` 接常量 **1**(低有效的"无效"电平);`aux_reset_in`(高有效)接 0;`mb_debug_sys_rst` 接 MDM;`dcm_locked` 接 1。**保留 `proc_sys_reset`**。
- **排查技巧**:把 `mb_reset` 引到板载 LED(复位断言=灭 / 释放=亮),再用"时钟分频闪灯"确认时钟,二分定位"是时钟问题还是复位问题"。
- **另一个坑**:若把 MicroBlaze `Reset` 硬接 0,CPU 会进入 `Running` 但 `Stalled on instruction fetch. PC=0x0`(缺少上电复位脉冲、未被正确初始化)。

## 7.2 改 BD 时钟后串口乱码(Vitis 应用时钟宏没同步)

- **现象**:软核串口回显字节部分或全部错误(如 4×`0x55` 收到 `55 55 55 FD`)。
- **根因**:`importsources` 把源文件**拷贝**进 Vitis 应用,改源文件不会自动同步;且 BD 时钟从 100MHz 改成 200MHz 后,应用里的 `UART_FCLK`(及 BSP 时钟宏)仍是旧值 → `BAUD_DIV` 算错 → 波特率翻倍 → 乱码。
- **解决**:改 BD 时钟后,**同步修改应用里的时钟宏并重新编译应用**;若 XSA 变了还需重建平台/BSP。

## 7.3 Vitis/XSCT 与硬件管理器冲突

- **现象**:BD/IP 操作报 `[Xicom 50-38] Unable to connect to debug core(s) on the target device`。
- **处理**:先 `close_hw_manager`(Vivado 侧占用 JTAG 会干扰 BD/IP 生成),再操作。
- **附**:2024.2 的 `xsct` 已弃用;平台/应用可用 `vitis -s`,或用一个 xsct 脚本在同一会话里 `platform create` + `platform generate` + `app create` + `importsources` + `app build`。

## 7.4 printf/scanf 增大代码,默认 8KB LMB 放不下

- **现象**:链接报 `.heap will not fit in region ... overflowed by N bytes`。
- **根因**:MicroBlaze 本地存储器(LMB BRAM)默认只有 **8KB**;加 printf/scanf 后代码增大(本项目 text+data+bss ≈ 10.9KB),连同 heap/stack 超出 8KB。
- **解决**:把 LMB 调大(本项目用 **32KB**)——重跑 MicroBlaze automation 时 `local_mem "32KB"`,或改 BD 里本地存储器容量。改后需**重建比特流 + 重导出 XSA + 重建平台/应用**。
- **注意**:地址/时钟若变化,应用里的 `UART_BASE`/`UART_FCLK` 要同步(本项目 uart 基址变成 `0x00020000`)。

## 7.5 MicroBlaze BD automation 在旧会话里静默失败

- **现象**:`apply_bd_automation -rule xilinx.com:bd_rule:microblaze` 执行完毕却**没生成** local memory / MDM / 互联,并报 `Could not find expected configuration value "disable"`。
- **处理**:**重启 Vivado 会话**(或新建工程)后重跑 automation 即成功。长时间、多次改动的会话会进入坏状态。
- **附带**:用 `clkrst` 自动化给 `clk_wiz` 建复位时,若 microblaze 自动化已建 `proc_sys_reset`,会提示 `rule ... was not applied`——此时跳过 `clkrst` 即可(复位已存在)。
