## 检查VT是否开启(是否开启支持虚拟化技术)

第一步，通过cpuid指令获取当前cpu是否支持VT，也就是cpuid(rax 为 1)

```assembly
;--------------------------------------------------------------;
;void cpuid(UINT64 *rax, UINT64 *rbx, UINT64 *rcx, UINT64 *rdx);
;--------------------------------------------------------------;
global _cpuid
_cpuid:
push rax
push rbx
push rcx
push rdx
push r8
push r9

;rdi ;param 1 (address of eax)
;rsi ;param 2 (address of ebx)
mov r8,rdx ;param 3 (address of ecx)
mov r9,rcx ;param 4 (address of edx)

mov rax,[rdi]
mov rbx,[rsi]
mov rcx,[r8]
mov rdx,[r9]

cpuid

mov [rdi],rax
mov [rsi],rbx
mov [r8],rcx
mov [r9],rdx

pop r9
pop r8
pop rdx
pop rcx
pop rbx
pop rax
ret
```

```c
UINT64 a,b,c,d;
a=0;
b=0;
c=0;
d=0;

a=1;

_cpuid(&a,&b,&c,&d); // 通过调用cpuid获取结果
if ((c >> 5) % 2) // rcx的第五位(从0开始)为1的话就支持VT
{
    Log("支持VT\n");
}
```

第二步，检查MSR寄存器查看bios中是否开启VT支持 ( VXMON开启虚拟化的指令 )

- **控制 VMXON 的 MSR**:
  - VMXON 还受 IA32_FEATURE_CONTROL MSR（MSR寄存器，地址为 3AH）的控制。这个 MSR 在逻辑处理器重置时被清零。
- **锁定位（位 0）**:
  - 如果锁定位（位 0）被清除，执行 VMXON 将引发通用保护异常。如果锁定位被设置，对该 MSR 的写入将引发通用保护异常；该 MSR 无法修改直到电源重置。系统 BIOS 可以使用这个位来提供一个设置选项以禁用对 VMX 的支持。
- **位 1 和 位 2**:
  - 位 1 用于在 SMX（Safer Mode Extensions）操作中启用 VMXON。如果这个位被清除，在 SMX 操作中执行 VMXON 将引发通用保护异常。
  - 位 2 用于在非 SMX 操作中启用 VMXON。如果这个位被清除，在非 SMX 操作中执行 VMXON 将引发通用保护异常。



第三步，检查CR0寄存器

CR0寄存器的PE位(位0，保护模式)，NE位(为5，浮点数异常的内部处理)，PG位(位31，分页机制)，是否为1(为1则开启)



第四步，设置CR4寄存器

1. **设置 CR4.VMXE[位 13] 为 1**:
   - 在系统软件可以进入 VMX 操作之前，它需要通过设置控制寄存器 CR4 的 VMXE 位（第 13 位）为 1 来启用 VMX。
2. **执行 VMXON 指令**:
   - 一旦 VMX 被启用，系统软件通过执行 VMXON 指令进入 VMX 操作模式。
   - 如果在 CR4.VMXE 为 0 的情况下执行 VMXON，将会引发无效操作码异常（#UD）。
3. **在 VMX 操作中，不能清除 CR4.VMXE**:
   - 一旦进入 VMX 操作，就不可能清除 CR4.VMXE 位。系统软件可以通过执行 VMXOFF 指令来离开 VMX 操作，然后在 VMX 操作之外清除 CR4.VMXE。



## Virtual Machine Extensions的大概执行流程

### 1. 检测支持，也就是之前提到的检测是否支持VT的四步

### 2. VMX操作的启用(VMXON)

- 启用VMX，也就是VT检测中的第四步
- 执行VMXON指令，并传递一个指向 VMXON 区域（一块特殊内存区域）的物理地址。成功执行 VMXON 指令后，处理器进入 VMX root 操作模式。



