# litmus-tests-riscv-ifetch

RISC-V instruction-fetch litmus tests. For each test, the verdict
line says whether the outcome it looks for can occur (allowed) or
cannot (forbidden) under four model configurations:

- `default`: the base RISC-V memory model with instruction fetch. A
  fetch is ordered before the accesses the fetched instruction
  makes, and `fence.i` orders a hart's earlier data accesses before
  its later fetches.

  ```
  ir1 = [I];(iico_data|iico_ctrl)+;[M & Exp]
  ir3 = [M & Exp];fencerel(Fence.i);[I]
  ```

- `Ziccid`: adds the Ziccid extension, which makes instruction fetch
  coherent with data. A hart's fetches occur in program order, and
  stores eventually become visible to instruction fetches without a
  `fence.i`.

  ```
  ir2 = [I];po;[I]
  ```

- `ir-decode`: an alternative rule under discussion that follows
  from assuming a hart decodes instructions in program order: each
  fetch is ordered before the hart's later memory accesses.

  ```
  ir_decode = [I];po;[Exp | (anyW & NExp)]
  ```

  > Alternative weaker variant only to explicit writes (bare minimum)

- `ir3-fetch`: an alternative rule under discussion under which
  `fence.i` also orders a hart's earlier fetches before its later
  fetches, not only its earlier data accesses.

  ```
  ir3f = [I];fencerel(Fence.i);[I]
  ```

### CoFF+fencei

```
RISCV CoFF+fencei
Variant=self
{ 0:t0=instr:"li t6, 2"; 0:a1=P1:mod; }
P0           | P1                ;
sw t0,0(a1)  | jal mod           ;
             | mv s0, t6         ;
             | fence.i           ;
             | jal mod           ;
             | mv s1, t6         ;
             | j end             ;
             | mod:              ;
             | li t6, 1          ;
             | jr ra             ;
             | end:              ;
exists (1:s0=2 /\ 1:s1=1)
```

A hart fetches the same address twice, with a `fence.i` between the
two fetches. Without Ziccid a hart's instruction fetches need not
occur in program order, so the second fetch can return the old
instruction even after the first has already seen the patched one.
The `fence.i` does not prevent this: it orders a hart's earlier data
accesses ahead of its later fetches, not one fetch ahead of another.

`default` allowed | `Ziccid` forbidden | `ir-decode` allowed | `ir3-fetch` forbidden

#### CoFF (no Fence.i)

Only forbidden with `Ziccid`

### CoFF+load-fencei

```
RISCV CoFF+load-fencei
Variant=self
{ 0:t0=instr:"lw t6, 0(a3)"; 0:a1=P1:mod;
  1:a3=z; z=2; }
P0           | P1                ;
sw t0,0(a1)  | jal mod           ;
             | mv s0, t6         ;
             | fence.i           ;
             | jal mod           ;
             | mv s1, t6         ;
             | j end             ;
             | mod:              ;
             | li t6, 1          ;
             | jr ra             ;
             | end:              ;
exists (1:s0=2 /\ 1:s1=1)
```

The same setup as CoFF+fencei, but the refetched instruction is
patched from a load-immediate into a load from memory. The executed
load gives the `fence.i` an explicit access to order against, so the
two fetches can no longer happen out of order. CoFF+fencei, with its
register-only instruction, leaves the `fence.i` nothing to anchor.

`default` forbidden | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` forbidden

### CoFW

```
RISCV CoFW
(* adapted from Brendan's DIC.nobackwards.litmus *)
Variant=self
{ 0:t0=instr:"nop"; 0:a1=P0:mod; 0:t1=0; }
P0           ;
mod:         ;
li t1, 1     ;
sw t0,0(a1)  ;
exists 0:t1=0
```

A hart fetches an instruction, then overwrites the same address with
a store later in program order. Without Ziccid, out-of-order fetch is
permitted, so the fetch can observe that later store instead of the
instruction originally there.

**`default` allowed !** | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

> This should not be allowed even without Ziccid

### ISA2.F+fence.w.w+fence.r.w+fencei

```
RISCV ISA2.F+fence.w.w+fence.r.w+fencei
Variant=self
{ 0:t0=instr:"li s2, 5"; 0:a1=P2:mod; 0:a2=y; 0:t1=1;
  1:a1=y; 1:a2=z; 1:t1=1;
  2:a1=z; }
P0           | P1           | P2           ;
sw t0,0(a1)  | lw s0,(a1)   | lw s1,(a1)   ;
fence w,w    | fence r,w    | fence.i      ;
sw t1,0(a2)  | sw t1,0(a2)  | mod:         ;
             |              | nop          ;
exists (1:s0=1 /\ 2:s1=1 /\ 2:s2=0)
```

An extended message-passing shape, with the flag relayed from the
writer through an intermediate hart to the hart that fetches the
patched code. The fences forbid the weak outcome even without
Ziccid, because the final hart's flag load sits before its
`fence.i` and gives it an access to order the fetch against.

`default` forbidden | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` forbidden

> Without Fence.i this should only be forbidden with `Ziccid`

### LB.FF

```
RISCV LB.FF
Variant=self
{ 0:t0=instr:"li s0, 2"; 0:a1=P1:mod1; 0:s0=0;
  1:t0=instr:"li s0, 2"; 1:a1=P0:mod0; 1:s0=0; }
P0           | P1           ;
mod0:        | mod1:        ;
li s0, 1     | li s0, 1     ;
sw t0,0(a1)  | sw t0,0(a1)  ;
exists (0:s0=2 /\ 1:s0=2)
```

The load-buffering shape with instruction fetches in place of the
usual data loads: each hart fetches its own site, then patches the
other hart's. Without Ziccid, out-of-order fetch permits the weak
outcome, where both fetches pick up the patch the other hart only
writes afterwards.

**`default` allowed !** | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

> cf. CoFW

### MP+fence.w.w+fencei

```
RISCV MP+fence.w.w+fencei
(* adapted from Brendan's DIC.fenceiMP.litmus *)
Variant=self
{ 0:a1=x; 0:a2=y; 0:t1=1; 1:a1=x; 1:a2=y; }
P0           | P1           ;
sw t1,0(a1)  | lw s0,(a2)   ;
fence w,w    | fence.i      ;
sw t1,0(a2)  | lw s1,(a1)   ;
exists (1:s0=1 /\ 1:s1=0)
```

Plain message passing with no self-modified code. The reader's
`fence.i` orders its earlier load ahead of the fetch of the later
one, and that fetch is ordered before its own load, so the two
reads cannot reorder in any configuration. Here `fence.i` acts as a
read-read barrier.

`default` forbidden | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` forbidden

### MP+fencei+fence.r.r

```
RISCV MP+fencei+fence.r.r
Variant=self
{ 0:a1=x; 0:a2=y; 0:t1=1; 1:a1=x; 1:a2=y; }
P0           | P1           ;
sw t1,0(a1)  | lw s0,(a2)   ;
fence.i      | fence r,r    ;
sw t1,0(a2)  | lw s1,(a1)   ;
exists (1:s0=1 /\ 1:s1=0)
```

The writer-side counterpart of MP+fence.w.w+fencei: the `fence.i`
now sits between the writer's two stores. It orders the earlier
store ahead of the fetch of the later store's instruction, and a
fetch is ordered before the store that instruction performs, so the
two stores cannot reorder. The `fence.i` acts as a write-write
barrier here, and with the reader's `fence r,r` the weak outcome is
forbidden in every configuration.

`default` forbidden | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` forbidden

### MP.F+load+fence.w.w

```
RISCV MP.F+load+fence.w.w
(* adapted from Brendan's DIC.fetchtoexecute.litmus *)
Variant=self
{ 0:t0=instr:"lw s0, 0(a2)"; 0:a1=P1:mod; 0:a2=z; 0:t1=1;
  1:a2=z; 1:s0=0; }
P0           | P1           ;
sw t1,0(a2)  | mod:         ;
fence w,w    | li s0, 2     ;
sw t0,0(a1)  |              ;
exists (1:s0=0)
```

An MP shape whose two reads on the reader are the implicit
instruction fetch and the explicit data load the fetched
instruction performs. The flag is the patched instruction itself, a
load of z, and the message passed is the writer's store to z. `ir1`
orders the fetch before that load, so the writer's fence is enough
to forbid the weak outcome, without fence.i or Ziccid.

`default` forbidden | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` forbidden

### MP.FF+jump-fencei

```
RISCV MP.FF+jump-fencei
(* adapted from Brendan's IDC.miniJit03+fencei.litmus *)
Variant=self
{ 0:t0=instr:"li s0, 3"; 0:a1=P1:L1;
  0:t1=instr:"j Ltarget"; 0:a2=P1:L2;
  1:s0=0; }
P0           | P1           ;
sw t0,0(a1)  | L2:          ;
fence w,w    | j L0         ;
sw t1,0(a2)  | L0:          ;
             | li s0, 1     ;
             | j Lend       ;
             | Ltarget:     ;
             | fence.i      ;
             | L1:          ;
             | li s0, 2     ;
             | Lend:        ;
exists (1:s0=2)
```

`default` allowed | `Ziccid` forbidden | `ir-decode` allowed | `ir3-fetch` forbidden

### MP.FF+patched-fence.i-ctrl

```
RISCV MP.FF+patched-fence.i-ctrl
Variant=self
{ 0:t0=instr:"li s9, 1"; 0:t1=instr:"fence.i";
  0:a1=P1:mod; 0:a2=P1:fi_slot;
  1:s8=0; 1:s9=0; }
P0           | P1           ;
sw t0,0(a1)  | fi_slot:     ;
fence w,w    |  j end       ;
sw t1,0(a2)  |  li s8, 1    ;
             | mod:         ;
             |  nop         ;
             | end:         ;
exists (1:s8=1 /\ 1:s9=0)
```

`default` allowed | `Ziccid` forbidden | `ir-decode` allowed | **`ir3-fetch` allowed !**

> `ir3-fetch` doesn't order its own fetch w.r.t. `po`-later fetches

### MP.FR+patched-fence.r.r-ctrl

```
RISCV MP.FR+patched-fence.r.r-ctrl
Variant=self
{ 0:t0=1; 0:t1=instr:"fence r,r";
  0:a1=z; 0:a2=P1:frr_slot;
  1:a1=z;
  1:s8=0; 1:s9=0; }
P0           | P1           ;
sw t0,0(a1)  | frr_slot:    ;
fence w,w    |  j end       ;
sw t1,0(a2)  |  li s8, 1    ;
             | mod:         ;
             |  lw s9,(a1)  ;
             | end:         ;
exists (1:s8=1 /\ 1:s9=0)
```

`default` allowed | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

### MP.FR+patched-nop-ctrl

```
RISCV MP.FR+patched-nop-ctrl
Variant=self
{ 0:t0=1; 0:t1=instr:"nop";
  0:a1=z; 0:a2=P1:frr_slot;
  1:a1=z;
  1:s8=0; 1:s9=0; }
P0           | P1           ;
sw t0,0(a1)  | frr_slot:    ;
fence w,w    |  j end       ;
sw t1,0(a2)  |  li s8, 1    ;
             | mod:         ;
             |  lw s9,(a1)  ;
             | end:         ;
exists (1:s8=1 /\ 1:s9=0)
```

`default` allowed | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

### MP.FR+fence.w.w+po

```
RISCV MP.FR+fence.w.w+po
Variant=self
{ 0:t0=instr:"li x10, 1"; 0:a1=P1:mod; 0:a2=x; 0:t1=1; 1:a2=x; 1:x10=0; }
 P0          | P1          ;
 sw t1,0(a2) | mod: nop    ;
 fence w,w   | lw s1,(a2)  ;
 sw t0,0(a1) |             ;
exists (1:x10=1 /\ 1:s1=0)
```

`default` allowed | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

### MP.RF+fence.w.w+addr-jr

```
RISCV MP.RF+fence.w.w+addr-jr
(* adapted from Brendan's DIC.miniJit01.litmus *)
Variant=self
{ [x]=P1:L0;
  0:t0=instr:"li s0, 3"; 0:a1=P1:L1; 0:a2=x;
  1:a2=x; 1:s0=0; }
P0           | P1           ;
sw t0,0(a1)  | ld s1,(a2)   ;
fence w,w    | jr s1        ;
sd a1,0(a2)  | L0:          ;
             | li s0, 1     ;
             | j Lend       ;
             | L1:          ;
             | li s0, 2     ;
             | Lend:        ;
exists (1:s0=2)
```

`default` allowed | `Ziccid` allowed | `ir-decode` allowed | `ir3-fetch` allowed

### MP.WW+irf+fence.r.r

```
RISCV MP.WW+irf+fence.r.r
(* adapted from Brendan's DIC.MP.WW+irf.litmus *)
Variant=self
{ 0:t0=instr:"li s0, 2"; 0:a1=P0:mod; 0:a2=x; 0:t1=1;
  1:a1=P0:mod; 1:a2=x; }
P0           | P1           ;
sw t0,0(a1)  | lw s1,(a2)   ;
mod:         | fence r,r    ;
li s0, 1     | lwu s2,0(a1)  ;
sw t1,0(a2)  |              ;
exists (0:s0=2 /\ 1:s1=1 /\ 1:s2=instr:"li s0, 1")
```

`default` allowed | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

> We probably want `1:s1=2` in the condition?

### SB+fetch-addr+fence.rw.rw

```
RISCV SB+fetch-addr+fence.rw.rw
(* adapted from Brendan's DIC.emailQ2.litmus *)
Variant=self
{ 0:t0=instr:"mv a1, a2"; 0:a0=P0:mod; 0:a1=y; 0:a2=x;
  1:a0=P0:mod; 1:a1=x; 1:t1=1; }
P0           | P1           ;
sw t0,0(a0)  | sw t1,0(a1)  ;
mod:         | fence rw,rw  ;
nop          | lwu s1,0(a0) ;
lwu s0,0(a1) |              ;
exists (0:a1=x /\ 0:s0=0 /\ 1:s1=instr:"nop")
```

`default` allowed | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed

### SB.RF+fence.w.r+po

```
RISCV SB.RF+fence.w.r+po
Variant=self
{ 0:t0=instr:"li s0, 2"; 0:a1=P1:mod; 0:a2=x;
  1:a2=x; 1:t1=1; 1:s0=0; }
P0           | P1           ;
sw t0,0(a1)  | sw t1,0(a2)  ;
fence w,r    | mod:         ;
lw s1,(a2)   | li s0, 1     ;
exists (0:s1=0 /\ 1:s0=1)
```

`default` allowed | `Ziccid` allowed | `ir-decode` allowed | `ir3-fetch` allowed

### SM+RW

```
RISCV SM+RW
Variant=self
{ 0:t0=instr:"li s0, 2"; 0:a1=P0:mod; 0:a2=x;
  1:t1=instr:"li s0, 3"; 1:a1=P0:mod; 1:a2=x; 1:t3=1; }
P0           | P1               ;
sw t0,0(a1)  | lwu s2,0(a1)      ;
lw s1,(a2)   | sw t1,0(a1)      ;
fence.i      | lwu s3,0(a1)      ;
mod:         | xor t2,s3,s3     ;
li s0, 1     | add a3,a2,t2     ;
             | sw t3,0(a3)      ;
exists (0:s1=1 /\ 1:s2=instr:"li s0, 2" /\ 1:s3=instr:"li s0, 3" /\ 0:s0=2)
```

`default` allowed | `Ziccid` allowed | `ir-decode` allowed | `ir3-fetch` allowed

### SM+RW+RW

```
RISCV SM+RW+RW
Variant=self
{ 0:t0=instr:"li s0, 2"; 0:a1=P0:mod; 0:a2=x;
  1:t1=instr:"li s0, 3"; 1:a1=P0:mod;
  2:a1=P0:mod; 2:a2=x; 2:t3=1; }
P0           | P1           | P2               ;
sw t0,0(a1)  | lwu s2,0(a1)  | lwu s3,0(a1)      ;
lw s1,(a2)   | sw t1,0(a1)  | xor t2,s3,s3     ;
fence.i      |              | add a3,a2,t2     ;
mod:         |              | sw t3,0(a3)      ;
li s0, 1     |              |                  ;
exists (0:s1=1 /\ 1:s2=instr:"li s0, 2" /\ 2:s3=instr:"li s0, 3" /\ 0:s0=2)
```

`default` forbidden | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` forbidden

### W+fencei+F

```
RISCV W+fencei+F
Variant=self
{ 0:t0=instr:"li s0, 1"; 0:a1=P1:mod; 1:s0=0; }
P0           | P1           ;
sw t0,0(a1)  | mod:         ;
fence.i      | nop          ;
exists 1:s0=0
```

`default` allowed | `Ziccid` allowed | `ir-decode` allowed | `ir3-fetch` allowed

### WRC.F.RF+po+fencei

```
RISCV WRC.F.RF+po+fencei
(* adapted from Brendan's DIC.WRC-inst.litmus *)
Variant=self
{ 0:t0=instr:"li s0, 1"; 0:a1=P0:mod;
  1:a1=P0:mod; 1:a2=x; 1:t1=1; 1:s0=0;
  2:a1=P0:mod; 2:a2=x; 2:s0=0; }
P0           | P1          | P2          ;
sw t0,0(a1)  | jalr a1     | lw s2,(a2)  ;
j end        | sw t1,(a2)  | fence.i     ;
mod:         |             | jalr a1     ;
nop          |             |             ;
jr ra        |             |             ;
end:         |             |             ;
exists (1:s0=1 /\ 2:s2=1 /\ 2:s0=0)
```

`default` allowed | `Ziccid` forbidden | `ir-decode` forbidden | `ir3-fetch` allowed
