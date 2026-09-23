# Python / MATLAB 与 Vivado 持久交互(Tcl Socket Server)

- **日期**:2026-09-21
- **环境**:Vivado 2018.3(Tcl 8.5,Windows)/ Python 3.12 / MATLAB
- **标签**:`#Tcl` `#socket` `#vivado-mcp` `#Python` `#MATLAB`
- **状态**:已解决

## 需求

AI(vivado-mcp 控制的 Vivado 会话)之外,还希望 **Python / MATLAB 能持久地驱动同一个 Vivado**——采集 ADC 数据、读寄存器、跑 FFT 环路,而不是每次冷启 `vivado -mode batch`。

## 分层原则

```
AI/MCP  ──(MCP 协议, 9999)──►  Vivado 进程(常驻)
Python/ ──(Tcl over TCP, 5001)─► 同一个 Vivado 进程
MATLAB
```

- **不要把 MCP 当 MATLAB/Python 的 RPC**(协议封闭)。
- 正确做法:**复用 MCP 已经开着的那个 Vivado 会话,在它里面再开一个你自己的 Tcl socket server**。AI、Python、MATLAB 控制同一个 Vivado,天然持久。
- **控制面**(命令)走 socket;**数据面**(ADC 样本)走文件 / DMA,绝不逐点走 Tcl。

## 在 Vivado 里开 Tcl Server(事件驱动版)

> 关键:**不要用阻塞式 `while {[gets ...]}`**,那会在 GUI 事件循环里卡死/异常断开,表现为客户端 `ConnectionResetError 10054`。用 `fileevent`。

```tcl
proc vmcp_reader {chan} {
    if {[catch {gets $chan line} n] || $n < 0} {
        catch {close $chan}
        return
    }
    set line [string trim $line]
    if {$line eq ""} { return }
    if {[catch {uplevel #0 $line} res]} {
        catch {puts $chan "ERR $res"}
    } else {
        catch {puts $chan "OK $res"}
    }
    catch {flush $chan}
}

proc vmcp_accept {chan addr port} {
    fconfigure $chan -buffering line -blocking 0 -translation auto
    fileevent $chan readable [list vmcp_reader $chan]
}

if {[info exists ::vmcp_srv]} { catch {close $::vmcp_srv} }
set ::vmcp_srv [socket -server vmcp_accept 5001]
puts "tcl server on 5001"
```

- `uplevel #0` 必须在**全局**执行命令,否则找不到 Vivado 的全局命令。
- 返回单行 `OK ...` / `ERR ...`;大结果走文件。
- 会话必须是 **GUI 模式**(事件循环在跑);`tcl` 模式不会调度 socket 回调。

## Python 客户端(推荐,最省事)

```python
import socket
import numpy as np

class VivadoTcl:
    def __init__(self, host="127.0.0.1", port=5001):
        self.s = socket.create_connection((host, port), timeout=5)

    def cmd(self, tcl):
        self.s.sendall((tcl.replace("\n", " ") + "\n").encode())
        buf = b""
        while not buf.endswith(b"\n"):
            chunk = self.s.recv(4096)
            if not chunk:
                break
            buf += chunk
        return buf.decode().rstrip("\n")

v = VivadoTcl()
print(v.cmd("get_hw_devices"))

# 拿到样本后做 FFT(SFDR/SNR/ENOB)
x = np.fromfile("adc.bin", dtype=np.int16)[:2**16]
X = 20*np.log10(np.abs(np.fft.rfft(x*np.blackman(len(x)))) / (2**13*len(x)/2) + 1e-12)
```

- 用 `recv` 手动读到 `\n`,抗半包;比 `makefile().readline()` 稳。
- 连接对象常驻 = 持久化;别每条命令重连。

## MATLAB 客户端(注意版本)

`tcpclient` 是 R2015a 就有,但 **`writeline` / `readline` 是 R2019b 才加入**。低版本会报 `未定义函数或变量 'writeline'`。

**R2019b 之前**:用 `fprintf` 写、`fgetl` 读:

```matlab
t = tcpclient("127.0.0.1", 5001);
fprintf(t, "%s\n", "get_hw_devices");
resp = char(fgetl(t));
```

> 若 `fprintf(t, ...)` / `fwrite(t, ...)` 报 **"文件标识符无效"**,说明 `t` 不是有效对象(这台机器没有 `tcpclient`,或对象没建起来),不是命令写法问题。先 `exist('tcpclient')`(0=不存在)。

**最稳、不依赖任何工具箱:Java socket**(MATLAB 自带 Java,任何版本可用):

```matlab
function resp = vivado_java(cmd)
    persistent sock in out
    if isempty(sock) || ~sock.isConnected()
        import java.net.Socket
        import java.io.*
        sock = Socket('127.0.0.1', 5001);
        in   = BufferedReader(InputStreamReader(sock.getInputStream()));
        out  = PrintWriter(OutputStreamWriter(sock.getOutputStream()));
    end
    out.print([char(cmd) char(10)]);   % 命令 + \n
    out.flush();
    resp = char(in.readLine());
end
```

## 验证步骤(别一上来发真命令)

1. **服务端在听**:Vivado 里 `puts $::vmcp_srv` 返回句柄;PowerShell `Test-NetConnection 127.0.0.1 -Port 5001` 为 True。
2. **最简命令测通**:`expr {1+1}` 应返回 `OK 2`。
3. **确认解释器**:`info nameofexecutable`(应是 vivado.exe)、`version`。若返回 tclsh/xsct,说明连错解释器。
4. 再发真实命令。

## 避坑

- **阻塞式 server** → 客户端 `ConnectionResetError 10054`;改 `fileevent`。
- **端口冲突**:换端口(5000→5001)先排除占用。
- **AI(MCP)与 Python 并发写同一 Vivado**:同时发长命令会排队/交错;建议 AI 搭台、Python/MATLAB 跑环路,或加软锁。
- **Vivado 跑综合/实现时事件循环被占**,socket 命令会阻塞,别在长任务里指望秒回。
- **`get_property STATUS [get_runs impl_1]`** 若工程/run 不存在会返回 `ERR ...`(正常,便于区分),不是连接问题。
