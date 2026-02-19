## x86 Segmentation, Descriptor Tables and Interrupt Handling

From the syscall note, we see from the trapframe

```c 
struct trapframe {
    uint edi;
    uint esi;
    uint ebp;
    uint oesp;
    uint ebx;
    uint edx;
    uint ecx;
    uint eax;

    ushort gs;
    ushort padding5;
    ushort fs;
    ushort padding5;
    ushort es;
    ushort padding5;
    ushort ds;
    ushort padding5;
    uint   trapno;

    // ! Pushed automatically by hardware
    uint   err;
    uint   eip;
    ushort cs;
    ushort padding5;
    uint   eflags;
    uint   esp;
    ushort ss;
    ushort padding6;
}

```

The question is why the hardware will push these registers and in such order.

### Interrupt

The hardware first push ss and esp, and then push eflags, cs and eip. Depends on the interrupt, the hardware will push the error code.

`ss` and `esp` are pushed when CPL (Current Privilege Level) <= DPL (Descriptor Privilege Level). When there is a change of privilege level (i.e. user -> kernel).

`eflags`, `cs` and `eip` are pushed during interrupt.


### Privilege Level (CPL vs RPL vs DPL)

| Privilege Level | Description |
| ----------------| -------------- |
| CPL | Current Privilege Level ( last 2 bits in cs )  |
| DPL | Descriptor Privilege Level ( In each descriptor )|
| RPL | Requested Privilege Level ( the segment Privilege Level that is requesting ) |

Ok so back to the discussion. During interrupt, remember the CPU access the IDT vector and look for 64th elements. We called that elemenet in the IDT vector a "gate descriptor"

`include/mmu.h`

```c
// Gate descriptors for interrupts and traps
struct gatedesc {
  uint off_15_0 : 16;   // low 16 bits of offset in segment
  uint cs : 16;         // code segment selector
  uint args : 5;        // # args, 0 for interrupt/trap gates
  uint rsv1 : 3;        // reserved
  uint type : 4;        // type(STS_{IG32,TG32})
  uint s : 1;           // must be 0 (system)
  uint dpl : 2;         // descriptor(meaning new) privilege level
  uint p : 1;           // Present
  uint off_31_16 : 16;  // high bits of offset in segment
};
```

The CPU check whether CPL >= GATE.DPL, if true, it push the `ss` and `esp`,  replace them with the target segment `ss` and `esp` (from TSS).

And then push/reserve `cs`, `eip` and `eflags`. Then set :

``` bash
CS = gate.target_selector
EIP = gate.target_offset -> &idt[INT_NO]
```

### Global Descriptor Table
