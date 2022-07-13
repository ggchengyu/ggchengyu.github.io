---
title: cannoli
date: 2022-07-13 11:49:05
tags:
- qemu-patch
- virtualization
categories:
- qemu
description:
- JIT hook in QEMU
---

# patch纪要
只记录大致添加的逻辑功能，学习tcg的部分在第二章节

```diff
--- a/include/tcg/tcg.h
+++ b/include/tcg/tcg.h
--- a/linux-user/main.c
+++ b/linux-user/main.c
+ // 添加自写库到linux-user和tcg中
+ // 目测是将cannoli作为qemu的可选功能加入linux-user
+...
```

```diff
--- a/tcg/i386/tcg-target.c.inc
+++ b/tcg/i386/tcg-target.c.inc
+ // 抢占R12，R13，R14寄存器为cannoli用

     TCG_REG_RBX,
+#ifndef CANNOLI
     TCG_REG_R12,
     TCG_REG_R13,
     TCG_REG_R14,
+#endif
     TCG_REG_R15,
...
static void tcg_out_qemu_ld_direct(TCGContext *s, TCGReg datalo, TCGReg datahi,
+#ifdef CANNOLI
+/* Add a PC argument so we know what instruction counter is doing the access */
+static void tcg_out_qemu_ld(TCGContext *s, const TCGArg *args, bool is64,
+    target_ulong pc)
+#else
 static void tcg_out_qemu_ld(TCGContext *s, const TCGArg *args, bool is64)
+#endif /* CANNOLI */


static void tcg_out_qemu_ld(TCGContext *s, const TCGArg *args, bool is64):
    opc = get_memop(oi);
+#ifdef CANNOLI
+ // avoid too large JIT target. Like 64-bit load on 32bit.
+ // 
+ tcg_out_mov(s, TCG_TYPE_PTR, TCG_REG_R14, addrlo)
+#endif
+
# if defined(CONFIG_SOFTMMU)
...
# endif
+
+#ifdef CANNOLI
+...
+ // avoid too large JIT target. Like 64-bit load on 32bit.
+ // inject shellcode
+ // Add a "pc" arg on tcg_out_qemu_ld, cannoli->lift_memop use "pc" arg.
+#endif  /* CANNOLI */
+...
+// 插入shellcode
+...

static void tcg_out_qemu_st(TCGContext *s, const TCGArg *args, bool is64):
opc = get_memop(oi);
+#if CANNOLI
+ // avoid too large JIT target.
+ // inject shellcode
+ size_t shellocode_size = cannoli->lift_memop(pc,
+       1, datalo, addrlo, opc & MO_SIZE, shellcode, sizeof(shellcode));
+ ...
+ for  ii < shellcode_size:
+ do
+   tcg_out8(s,shellcode[ii]);
+ done
+
+#endif /* CANNOLI */

//same code
# if defined(CONFIG_SOFTMMU)
...
#endif
+ // 插入shellcode

static void tcg_target_qemu_prologue(TCGContext *s):
    for (i = 0; i < ARRAY_SIZE(tcg_target_callee_save_regs); i++) {
        tcg_out_push(s, tcg_target_callee_save_regs[i]);
    }

+ // Purpose: function that is creating the "trampoline" that jumps into the JIT!
+ // Hook the first thing after registers are saved and switch into the JIT context.
+ // Why?: authors say it provides a low-frequency event that allows them to flush
+ // logs. Call into Rust and do a slightly more complicated operation.

+ // caller-save
+ 
+ // 3 * 8 = 24 , + 8 = 32 -> for 16-byte boundary to match x64 ABI
+ tcg_out_addi(s, TCG_REG_RSP, -32);
+ // ? rsp -> rdi
+ // is that use rsp as arg for cannoli->jit_entry ?
+ tcg_out_move(s, TCG_TYPE_PTR, TCG_REG_RDI, TCG_REG_RSP);
+ 
+ //I guess so
+ tcg_out_call(s, (void *)cannoli->jit_entry);
+
+ // store cannoli->jit_entry function return value to R12,R13,R14 , 
+ // which refer to regs that cannoli occupied.
+ tcg_out_ld(s , TCG_TYPE_PTR, TCG_REG_R12, TCG_REG_RSP, 0x00);
+ tcg_out_ld(s , TCG_TYPE_PTR, TCG_REG_R13, TCG_REG_RSP, 0x08);
+ tcg_out_ld(s , TCG_TYPE_PTR, TCG_REG_R14, TCG_REG_RSP, 0x10);
+ 
+ // caller-restore
+ ...
...
    if (have_avx2) {
        tcg_out_vex_opc(s, OPC_VZEROUPPER, 0, 0, 0, 0);
    }
+#ifdef CANNOLI
+ // caller-save
+
+ // take R12, R13, R14 values as args for cannoli->jit_exit function 
+ tcg_out_mov(s, TCG_TYPE_PTR, TCG_REG_RDI, TCG_REG_R12);
+ tcg_out_mov(s, TCG_TYPE_PTR, TCG_REG_RDI, TCG_REG_R13);
+ tcg_out_mov(s, TCG_TYPE_PTR, TCG_REG_RDI, TCG_REG_R14);
+ 
+ tcg_out_call(s, (void*)cannoli->jit_exit);
+
+ // caller-restore
+ ...

+ // 调整tcg_out_op原型（cannoli和非cannoli）

```

Q: what do they mean?
```diff
--- a/tcg/tcg.c
+++ b/tcg/tcg.c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb)
+ // Used to translate TCG?
+ ...
+ // decode ?
+ magic number?
++#ifdef CANNOLI
+    // Current target program counter. Updated by insn_start instructions
+    target_ulong cannoli_pc = (target_ulong)0xdeaddeaddeaddeadULL;
+#endif // CANNOLI
+ ...
+ size_t shellcode_size =
+                        cannoli->lift_instruction(
+                            cannoli_pc, shellcode, sizeof(shellcode));
+ ...
+ // 为了统计cannoli_pc，需要发出insn_start的opcode，因此做了一些额外的工作。。
+ TCGArg args[TCG_MAX_OP_ARGS] = { 0 };
+ int consts[TCG_MAX_OP_ARGS] = { 0 };
+
+ // Write in the PC into the `consts` array
+ *(target_ulong*)consts = s->gen_insn_data[num_insns][0];
+
+ // Emit the opcode
+ tcg_out_op(s, op->opc, args, consts);

...
+ // 调整tcg_out_op声明（cannoli和非cannoli）
...

static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op)
+ // 调整tcg_reg_alloc_op原型
+#ifdef CANNOLI
+static void tcg_reg_alloc_op(
+        TCGContext *s,
+        const TCGOp *op,
+        const target_ulong pc)
+#else
 static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op)
+#endif
...
+#if CANNOLI
+ tcg_out_op(s, op->opc, new_args, const_args, pc);
+#else
tcg_out_op(s, op->opc, new_args, const_args);
```


```diff
--- a/configure
+++ b/configure
+ // 添加cannoli至configure
+ // 添加CONFIG_CANNOLI至configrue
```

```diff
void cpu_loop_exit(CPUState *cpu)
+#ifdef CANNOLI
+    /* If we ever exit the CPU loop, perform a JIT exit */
+    if(cannoli && cannoli->jit_exit) {
+        CPUArchState *env = cpu->env_ptr;
+
+        cannoli->jit_exit(
+                env->cannoli_r12, env->cannoli_r13, env->cannoli_r14);
+    }
+#endif
```

```diff
target/${arch}/cpu.h
CPUArchState:
+ uint64_t cannoli_r12;
+ uint64_t cannoli_r13;
+ uint64_t cannoli_r14;
```


## what is important?
```diff
+/*
+ * Currently we only support Cannoli in qemu-user mode. In theory it would work
+ * in qemu-system, but it's kinda pointless if you don't have hooks/traces for
+ * context switches, page table changes, etc.
+ */
```
It means, the patch can be applied to QEMU in user-mode, but can't work in system-mode.


# TCG相关逻辑整理
为何插在tcg这些功能块？其主要功能是什么？
--- 描绘tcg的工作流程。

```C

/* tcg/i386/tcg-target.c.inc */
static void tcg_out_qemu_ld(TCGContext *s, const TCGArg *args, bool is64);
static void tcg_out_qemu_ld_direct(TCGContext *s, TCGReg datalo, TCGReg datahi,
                                   TCGReg base, int index, intptr_t ofs,
                                   int seg, bool is64, MemOp memop);
static void tcg_out_qemu_st(TCGContext *s, const TCGArg *args, bool is64);
static void tcg_target_qemu_prologue(TCGContext *s);


/* tcg/tcg.c */
int tcg_gen_code(TCGContext *s, TranslationBlock *tb);
static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op);
static void tcg_out_op(TCGContext *s, TCGOpcode opc,
                       const TCGArg args[TCG_MAX_OP_ARGS],
                       const int const_args[TCG_MAX_OP_ARGS]);

/* accel/tcg/cpu-common.c */
// 原本
void cpu_loop_exit(CPUState *cpu);

/* accel/tcg/cpu-exec.c */
static inline TranslationBlock * QEMU_DISABLE_CFI
cpu_tb_exec(CPUState *cpu, TranslationBlock *itb, int *tb_exit);

/* linux-user/signal.c */
static void host_signal_handler(int host_sig, siginfo_t *info, void *puc);
```