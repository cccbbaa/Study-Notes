## Intel 处理器的虚拟化中启用单步执行模式

Intel 处理器，通过 VMCS（Virtual Machine Control Structure）提供的两个特性来实现单步执行模式：

1. **监视陷阱标志（Monitor Trap Flag）**:
   - 这是一种处理器基于的 VM 执行控制，它允许在执行每条虚拟机指令后立即触发 VM exit。这样，Hypervisor 可以在每条指令执行后进行控制和检查。
   - 这是实现单步调试的最直接方式，可以精确地控制虚拟机的执行流程。

```assembly
global getcpuinfo
getcpuinfo:
mov rax, qword [fs:0]
ret
```



```c
#define vm_execution_controls_cpu   0x4002
#define PBEF_MONITOR_TRAP_FLAG    (1<<27)
// 获取 VM 执行控制字段
ULONGLONG IA32_VMX_PROCBASED_CTLS=readMSR(0x482);

int vmx_enableProcBasedFeature(DWORD PBF)
{
  if (((IA32_VMX_PROCBASED_CTLS >> 32) & PBF) == PBF) //can it be set
  {
    vmwrite(vm_execution_controls_cpu, vmread(vm_execution_controls_cpu) | PBF);
    return 1;
  }
  else
    return 0;
}

int vmx_enableSingleStepModeIntelMTF(void)
{
    pcpuinfo c=getcpuinfo(); // 获取cpu信息的指针 通过汇编实现 cpuinfo的具体定义在cpuinfo.md
    // 启用监视陷阱标志
	if (vmx_enableProcBasedFeature(PBEF_MONITOR_TRAP_FLAG))
    {
      // sendstring("Using the monitor trap flag\n"); // log
      c->singleStepping.Method=1;// 可能用于指示单步执行应当采用的具体方法或策略
      return 1;
    }
}


```



2. **中断窗口退出（Interrupt Window Exiting）**:
   - 如果无法使用监视陷阱标志，可以使用中断窗口退出作为替代方案。这种方法会在虚拟机准备接收中断时触发 VM exit。
   - 该方法可能不如监视陷阱标志那样精确，因为它依赖于虚拟机中断的接收状态。

```c
#define vm_execution_controls_cpu   0x4002
#define PBEF_INTERRUPT_WINDOW_EXITING  (1<<2)
// 获取 VM 执行控制字段
ULONGLONG IA32_VMX_PROCBASED_CTLS=readMSR(0x482);

int vmx_enableProcBasedFeature(DWORD PBF)
{
  if (((IA32_VMX_PROCBASED_CTLS >> 32) & PBF) == PBF) //can it be set
  {
    vmwrite(vm_execution_controls_cpu, vmread(vm_execution_controls_cpu) | PBF);
    return 1;
  }
  else
    return 0;
}

int vmx_enableSingleStepModeIntelIWE(void)
{
    pcpuinfo c=getcpuinfo(); // 获取cpu信息的指针 通过汇编实现 cpuinfo的具体定义在cpuinfo.md
    c->singleStepping.Method=2; // 可能用于指示单步执行应当采用的具体方法或策略
    // 启用中断窗口退出
    if (vmx_enableProcBasedFeature(PBEF_INTERRUPT_WINDOW_EXITING))
    {
        // 读取 VM-entry 中断信息字段，然后右移 31 位，以获取该字段最高位的值。这一位表示是否有待处理的中断（pending interrupt）。 1 则有 0 则没有
        if ((vmread(vm_entry_interruptioninfo) >> 31)==0) 
			vmwrite(vm_guest_interruptability_state,2); // 设置虚拟机的中断不可屏蔽状态（interruptibility state）为 2
        //如果有待处理的中断（即最高位为 1），则建议虚拟机将首先处理该中断，然后停止。（未实现）

        sendstring("Using the interrupt window\n"); // log
        return 1;
    }
}

```



## AMD 处理器的单步执行模式

AMD 处理器使用的是 VMCB（Virtual Machine Control Block）来管理虚拟化，而不是 Intel 的 VMCS。在 AMD 的虚拟化实现中：

- 启用对外部中断（VINTR）和内部中断（INTR）的拦截。
- 设置中断拦截以捕获所有异常。
- 修改 VMCB 的清理位，表示拦截设置已更改。
- 设置 RFLAGS 的 TF（Trap Flag）位和 RF（Resume Flag）位，以启用单步模式。
- 对于第一次单步，记录旧的 EFER 和 SFMASK，以便之后恢复。

```c
typedef union
{
  UINT64 value;
  struct
  {
    unsigned CF     :1; // 0
    unsigned reserved1  :1; // 1
    unsigned PF     :1; // 2
    unsigned reserved2  :1; // 3
    unsigned AF     :1; // 4
    unsigned reserved3  :1; // 5
    unsigned ZF     :1; // 6
    unsigned SF     :1; // 7
    unsigned TF     :1; // 8
    unsigned IF     :1; // 9
    unsigned DF     :1; // 10
    unsigned OF     :1; // 11
    unsigned IOPL   :2; // 12+13
    unsigned NT     :1; // 14
    unsigned reserved4  :1; // 15
    unsigned RF     :1; // 16
    unsigned VM     :1; // 17
    unsigned AC     :1; // 18
    unsigned VIF    :1; // 19
    unsigned VIP    :1; // 20
    unsigned ID     :1; // 21
    unsigned reserved5  :11; // 22-63
    unsigned reserved6  :32; // 22-63
  }__attribute__((__packed__));
} __attribute__((__packed__)) RFLAGS,*PRFLAGS;

int vmx_enableSingleStepModeAMD(void)
{
    pcpuinfo c=getcpuinfo(); // 获取cpu信息的指针 通过汇编实现 cpuinfo的具体定义在cpuinfo.md
    
    //启用对外部中断（VINTR）和内部中断（INTR）的拦截。
    c->vmcb->InterceptVINTR=1;
    c->vmcb->InterceptINTR=1;
    c->vmcb->InterceptExceptions=0x0000ffff;
    
    // 在 AMD 的 SVM（Secure Virtual Machine）技术中
    // VMCB_CLEAN_BITS 字段用于标识 VMCB 中哪些部分自上次 VMRUN 指令执行以来未被修改。
    // 指定位为0表示被改变
    
    c->vmcb->VMCB_CLEAN_BITS&=~(1<<0); // 清理VMCB_CLEAN_BITS 中的第 0 位
    
    // 将整个 VMCB_CLEAN_BITS 字段清零，表示 VMCB 中的所有部分都已被修改。确保在下次虚拟机运行时，VMCB 中的所有数据都将被重新检查和处理。
    c->vmcb->VMCB_CLEAN_BITS=0;
    
    RFLAGS v;
    v.value=c->vmcb->RFLAGS;

    if (c->singleStepping.ReasonsPos==0) // 如果是第一次设置单步执行
      c->singleStepping.PreviousTFState=v.TF;// 记录原始的 TF（Trap Flag）状态

    v.TF=1; // 启用单步
    v.RF=1;
    if (v.IF) // 如果允许中断
      c->vmcb->INTERRUPT_SHADOW=1;// 启用中断阴影（Interrupt Shadow）
    // 中断阴影是一种处理器状态，用于指示处理器刚刚完成对某些特定指令（如 STI 或 MOV SS）的执行
    // 这些指令会影响中断的启用。在中断阴影状态中，处理器会延迟对中断的响应
    // 直到执行了另外一条指令后。这是为了处理与中断启用和禁用相关的复杂情况，特别是在多任务操作系统中。
    
    c->vmcb->RFLAGS=v.value;
    c->singleStepping.Method=3; //Trap flag
    
    //turn of syscall, and when syscall is executed, capture the UD, re-enable it, but change the flags mask to keep the TF enabled, and the step after that adjust R11 so that the TF is gone and restore the flags mask.  Then continue as usual;
    if (c->singleStepping.ReasonsPos==0)
    {
      c->singleStepping.PreviousEFER=c->vmcb->EFER;
      c->singleStepping.PreviousFMASK=c->vmcb->SFMASK;
      c->singleStepping.LastInstructionWasSyscall=0;

      c->vmcb->EFER&=0xfffffffffffffffeULL;
      c->vmcb->VMCB_CLEAN_BITS&=~(1<< 5); //efer got changed
    }


    return 1;
    
}
```

