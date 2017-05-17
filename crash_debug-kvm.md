
# load kvm driver

kvm is built as driver, so it must be loaded.
* use /proc/kcore to debug current running kernel
  * same to use dump file to debug kernel panic
* download dbg package of current kernel
```
$ sudo crash /proc/kcore usr/lib/debug/vmlinux-4.4.36-2
...
crash> mod -S ./tmp/usr/lib/debug/
     MODULE       NAME                    SIZE  OBJECT FILE
...
ffffffffa00e5900  kvm                   415201  ./tmp/usr/lib/debug/usr/lib/modules/4.4.36-2-pserver/kernel/arch/x86/kvm/kvm.ko
...
```

# find "struct kvm"

vm_list is a global variable of kvm driver. We can check that value.
```
crash> p vm_list
vm_list = $1 = {
  next = 0xffff8804063e89f8, 
  prev = 0xffff8804063e89f8
}
```

vm_list exists inside of "struct kvm". So we need offsetof macro to get the address of "struct kvm".
* offsetof shows the offset of vm_list is 0x9f8
* the address of struct kvm is 0xffff8804063e89f8 - 0x9f8
* use p/x to print data of structure
```
crash> macro define offsetof(t, f) &((t *) 0)->f)
crash> p offsetof(struct kvm, vm_list)
$2 = (struct list_head *) 0x9f8
crash> p/x *(struct kvm *)(0xffff8804063e89f8 - 0x9f8)
$3 = {
  mmu_lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 0x0
          }
        }
      }
    }
  }, 
  slots_lock = {
...
```

Use ps to find PID of qemu
* this server changed program name of qemu into kvm. kvm process is not kvm driver.
```
crash> ps | grep kvm
    257      2   0  ffff880408c67000  IN   0.0       0      0  [kvm-irqfd-clean]
   7704   7703   5  ffff880406395400  IN   2.5 9035580 426588  kvm
   7705   7703   3  ffff8803fbc12a00  IN   2.5 9035580 426588  kvm
   7706   7703   7  ffff8803fbc13800  IN   2.5 9035580 426588  kvm
   7707   7703   1  ffff8803fbc11c00  IN   2.5 9035580 426588  kvm
   7708   7703   4  ffff8803fbc10e00  IN   2.5 9035580 426588  kvm
   7709   7703   2  ffff8803fbc17000  IN   2.5 9035580 426588  kvm
   7710      2   4  ffff88040947e200  IN   0.0       0      0  [kvm-pit/7704]
...
crash> task 7704
PID: 7704   TASK: ffff880406395400  CPU: 5   COMMAND: "kvm"
crash> task 7704
PID: 7704   TASK: ffff880406395400  CPU: 5   COMMAND: "kvm"
struct task_struct {
  state = 1, 
  stack = 0xffff8802e0768000, 
  usage = {
    counter = 2
  ...
```

mm field of "struct kvm" and "struct task_struct" must be the same.
```
crash> task 7704 | grep 'mm ='
      start_comm = "\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"
  mm = 0xffff8800375503c0, 
  active_mm = 0xffff8800375503c0, 
  comm = "kvm\000\000)\000\000oot\000\000\000\000", 
crash> p/x ((struct kvm *)0xffff8804063e8000).mm
$6 = 0xffff8800375503c0
```

# read register values of VCPU

struct kvm has vcpus array that has CPU context of each virtual CPUs
* http://elixir.free-electrons.com/linux/latest/source/include/linux/kvm_host.h#L380
```
crash> p/x ((struct kvm *)0xffff8804063e8000).vcpus
$7 = {0xffff8803fbfa0000, 0xffff8800616a0000, 0xffff8802de498000, 0xffff8802de658000, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}
```

kvm_vcpu addresses are 0xffff8803fbfa0000, 0xffff8800616a0000, 0xffff8802de498000, 0xffff8802de658000 because there are 4 virtual CPUs.
And kvm_vcpu is included in "struct vcpu_vmx" or "struct vcpu_svm". The first element of "struct vcpu_vmx" and "struct vcpu_svm" is "struct kvm_vcpu". Therefore the address of kvm_vcpu is the address of vcpu_vmx or vcpu_svm.
* vmx is INTEL virtualization and svm is AMD

## INTEL/vmx case

crash can show a structure data with struct command
* struct STRUCT-NAME ADDRESS
* -o: print offset of each element
* "struct STRUCT-NAME" prints structure definition
```
crash> struct vcpu_vmx 0xffff8803fbfa0000
struct vcpu_vmx {
  vcpu = {
    kvm = 0xffff8804063e8000, 
    preempt_notifier = {
      link = {
        next = 0x0, 
        pprev = 0xffff8803fbc13a00
      }, 
      ops = 0xffffffffa020f800 <kvm_preempt_ops>
    }, 
    cpu = 0, 
    vcpu_id = 0, 
    srcu_idx = 0, 
```

register values of VCPU is stored in kvm_vcpu.arch.regs
* regs has http://elixir.free-electrons.com/linux/latest/source/arch/x86/include/asm/kvm_host.h#L127
```
crash> struct -x kvm_vcpu.arch 0xffff8803fbfa0000
  arch = {
    regs = {0x0, 0xffffffff8184d360, 0x0, 0xffffffff818d6da0, 0x6df4, 0x0, 0x0, 0x0, 0x0, 0x0, 0x12e31946, 0x58f3b4edd7a, 0xffffffff81800000, 0x0, 0xffffffff81800000, 0xffffffed, 0xffffffff8100e4a6}, 
    regs_avail = 0xffe1ffef, 
    regs_dirty = 0x10000, 
    cr0 = 0x80050033, 
crash> struct -x kvm_vcpu.arch.regs_avail 0xffff8803fbfa0000
  arch.regs_avail = 0xffe1ffef,
```

But the problem is RSP value in regs[VCPU_REGS_RSP = 4] is not valid because bit #4 is 0 in arch.regs_avail.
In this case, we should use other dynamic tools, for instance kprobe/jprobe, to check register values.


## AMD/svm case

AMD processor virtualization code uses "struct vcpu_svm" and there is "struct vmcb" to store register values of VCPUs.
* read vmcb pointer in struct vcpu_svm
* print struct vmcb
```
crash> struct vcpu_svm 0xffff881aa0eabec0 | grep vmcb
  vmcb = 0xffff881b32e3a000,
  vmcb_pa = 116817895424,
    vmcb = 0,
    vmcb_msrpm = 0,
    vmcb_iopm = 0,
crash> p /x *(struct vmcb*) 0xffff881b32e3a000
$15 = {
  control = {
    intercept_cr = 0x110011,
    intercept_dr = 0xff00ff,
    intercept_exceptions = 0x600c2,
    intercept = 0x2e7fbdc48027,
    reserved_1 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0},
    pause_filter_count = 0xbb8,
    iopm_base_pa = 0x1812f78000,
    msrpm_base_pa = 0x1b32ca2000,
    tsc_offset = 0xffe71a30089ab3e8,
    asid = 0x6951,
    tlb_ctl = 0x0,
    reserved_2 = {0x0, 0x0, 0x0},
    int_ctl = 0x10f0000,
    int_vector = 0x0,
    int_state = 0x0,
    reserved_3 = {0x0, 0x0, 0x0, 0x0},
  control = {
    intercept_cr = 0x110011,
    intercept_dr = 0xff00ff,
    intercept_exceptions = 0x600c2,
    intercept = 0x2e7fbdc48027,
    reserved_1 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0},
    pause_filter_count = 0xbb8,
    iopm_base_pa = 0x1812f78000,
    msrpm_base_pa = 0x1b32ca2000,
    tsc_offset = 0xffe71a30089ab3e8,
    asid = 0x6951,
    tlb_ctl = 0x0,
    reserved_2 = {0x0, 0x0, 0x0},
    int_ctl = 0x10f0000,
    int_vector = 0x0,
    int_state = 0x0,
    reserved_3 = {0x0, 0x0, 0x0, 0x0},
    exit_code = 0x60,
    exit_code_hi = 0x0,
    exit_info_1 = 0x0,
    exit_info_2 = 0x0,
    exit_int_info = 0x0,
    exit_int_info_err = 0x0,
    nested_ctl = 0x1,
    reserved_4 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
    event_inj = 0x0,
    event_inj_err = 0x0,
    nested_cr3 = 0xfdfd76000,
    lbr_ctl = 0x0,
    clean = 0x5f7,
    reserved_5 = 0x0,
    next_rip = 0x0,
    insn_len = 0x0,
    insn_bytes = {0x49, 0x8b, 0x55, 0x0, 0x48, 0x39, 0xea, 0x74, 0x15, 0x48, 0x85, 0xed, 0xf, 0x85, 0xf1},
    reserved_6 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0
, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0
x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x
0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0...}
  },
  save = {
    es = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0xffffffff,
      base = 0x0
    },
    cs = {
      selector = 0x10,
      attrib = 0xa9b,
      limit = 0xffffffff,
      base = 0x0
    },
    ss = {
      selector = 0x2b,
      attrib = 0x0,
      limit = 0xffffffff,
      base = 0x0
    },
    ds = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0xffffffff,
      base = 0x0
    },
    fs = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0xffffffff,
      base = 0x7ffaa942e700
    },
    gs = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0xffffffff,
      base = 0x0
    },
    gdtr = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0x7f,
      base = 0xffff88003de09000
    },
    ldtr = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0xffff,
      base = 0x0
    },
    idtr = {
      selector = 0x0,
      attrib = 0x0,
      limit = 0xfff,
      base = 0xffffffffff574000
    },
    tr = {
      selector = 0x40,
      attrib = 0x8b,
      limit = 0x2087,
      base = 0xffff88003de103c0
    },
    reserved_1 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0, 0x0},
    cpl = 0x0,
    reserved_2 = {0x0, 0x0, 0x0, 0x0},
    efer = 0x1d01,
    reserved_3 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0
, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
    cr4 = 0x407f0,
    cr3 = 0x36c9e000,
    cr0 = 0x8005003b,
    dr7 = 0x400,
    dr6 = 0xffff0ff0,
    rflags = 0x46,
    rip = 0xffffffff8152d1a9,
    reserved_4 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0
, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
    rsp = 0x7ffaa942de28,
    reserved_5 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
    rax = 0x0,
    star = 0x23001000000000,
    lstar = 0xffffffff8152c190,
    cstar = 0xffffffff8152ed50,
    sfmask = 0x47700,
    kernel_gs_base = 0xffff88003de00000,
    sysenter_cs = 0x10,
    sysenter_esp = 0x0,
    sysenter_eip = 0x8152eb10,
    cr2 = 0x7f4803ec4000,
    reserved_6 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
    g_pat = 0x7040600070406,
    dbgctl = 0x0,
    br_from = 0x0,
    br_to = 0x0,
    last_excp_from = 0x0,
    last_excp_to = 0x0
  }
}
```

# find what irq was injected by kvm

"struct kvm_vcpu" has interrupt number of Guest,  injected by kvm.
* 239 is "Local APIC timer interrupt"
```
crash> struct kvm_vcpu.arch.interrupt.nr 0xffff8802c4088000
  arch.interrupt.nr = 239 '\357'
```

# read registers of VCPU with jprobe
