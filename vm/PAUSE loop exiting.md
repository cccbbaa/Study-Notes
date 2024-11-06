
(翻译自Vol.3C 25.6.13 Controls for PAUSE-Loop Exiting)
在支持将“PAUSE-loop exiting” VM 执行控制设置为 1 的处理器上，VM 执行控制字段包括以下 32 位字段：
- **PLE_Gap**：软件可以将此字段配置为循环中两次连续执行 PAUSE 指令之间的时间上限。(VMCS 0x4020)
- **PLE_Window**：软件可以将此字段配置为允许来宾在 PAUSE 循环中执行的时间上限。这些字段基于与时间戳计数器（TSC）相同速率运行的计数器来测量时间。有关 PAUSE-loop exiting 的更多详细信息，请参见第 26.1.3 节。(VMCS 0x4022)

(翻译自Vol.3C 26.1.3 Instructions That Cause VM Exits Conditionally)
**PAUSE.** 每个指令的行为取决于当前特权级（CPL）以及“PAUSE exiting”和“PAUSE-loop exiting” VM 执行控制的设置：
- **CPL = 0**：
  - 如果“PAUSE exiting”和“PAUSE-loop exiting” VM 执行控制都为 0，PAUSE 指令正常执行。
  - 如果“PAUSE exiting” VM 执行控制为 1，PAUSE 指令将导致 VM 退出（如果 CPL = 0 且“PAUSE exiting” VM 执行控制为 1，则忽略“PAUSE-loop exiting” VM 执行控制）。
  - 如果“PAUSE exiting” VM 执行控制为 0 而“PAUSE-loop exiting” VM 执行控制为 1，则适用以下处理：
    - 处理器确定本次在 CPL 0 下执行 PAUSE 指令与上一次在 CPL 0 下执行 PAUSE 指令之间的时间。如果此时间超过了 VM 执行控制字段 PLE_Gap 的值，处理器将认为这是循环中第一次执行 PAUSE 指令（在 VM 进入后首次在 CPL 0 下执行 PAUSE 指令时，也会这样处理）。
    - 否则，处理器将确定自最近一次被认为是循环中第一次执行的 PAUSE 指令以来所经过的时间。如果此时间超过了 VM 执行控制字段 PLE_Window 的值，将发生 VM 退出。
    - 为了进行这些计算，时间是基于与时间戳计数器（TSC）相同速率运行的计数器来测量的。
- **CPL > 0**：
  - 如果“PAUSE exiting” VM 执行控制为 0，PAUSE 指令正常执行。
  - 如果“PAUSE exiting” VM 执行控制为 1，PAUSE 指令将导致 VM 退出。若 CPL > 0，则忽略“PAUSE-loop exiting” VM 执行控制。