---
title: RISCV-BOOM后端学习-01 Rename
description: BOOM处理器的学习记录
date: 2024-10-03 04:20:00.000 +0800
tags:
    - BOOM
    - CPU

categories:
    - CPU微架构
---

## 背景

最近要写超标量处理器的后端执行部分，看了姚永斌老师的《超标量处理器设计》后准备大展身手，结果发现很多细节实现方面存在问题，比如分支misprediction时各模块间如何配合清除错误的指令且不影响正常执行，因此想找一个开源的处理器学习一下，香山的代码相对来说比较复杂，最后选择了以更简单一点的BOOM作为参考，纯靠眼睛看代码结合文档理解，并且本人不太熟悉函数式编程，一些地方可能理解有误。

## 重命名

BOOM采用的是统一的物理寄存器实现重命名的设计，也就是所谓的“explicit renaming”，只有PRF，然后用单独的map table来管理映射关系，而另一种“implicitrenaming”
则是将结果写入ROB，然后ROB提交时才写回逻辑寄存器。

我原以为显式重命名应该是更先进的方式，但文档上说P4，ARM A57采用的是隐式重命名，更老的的MIPS R10K，Alpha21264采用的是显式重命名，说明这两种设计算是各有取舍。

重命名模块的组成成分可以分为：Map Table、Free List、Busy Table三部分。

![rename-pipeline](rename-pipline.png)

### Map Table

BOOM的map_table是常规的用逻辑寄存器号寻址的结构，我原以为能容纳更多checkpoint的CAM内容寻址结构会是主流，但根据目前看过的资料发现这种结构似乎不常用？典型例子我只看过姚老师书上的Alpha21264。

BOOM对每个分支TAG保存了完整的`map_table`，这种实现方面会导致checkpoint非常占面积，并且容纳的分支指令有限，不过free_list以及用于发射队列和ROB恢复的分支TAG也是有限的，在这种情况下即使使用CAM结构的重命名表checkpoint足够多也没法容纳过量的分支指令。

BOOM有两种实现，可以同时保存了推测执行的重命名表和提交之后才变化的committed map table， 

misprediction时用其恢复，也可以用ROB中记录的旧映射关系恢复，

这部分代码还算比较好理解
```scala
  val map_table = RegInit(VecInit((0 until numLregs) map { i => i.U(pregSz.W) }))
  val com_map_table = RegInit(VecInit((0 until numLregs) map { i => i.U(pregSz.W) }))
  val br_snapshots = Reg(Vec(maxBrCount, Vec(numLregs, UInt(pregSz.W))))
```
`map_table`为推测执行的映射表，`com_map_table`为提交后更新的
architecture map table，根据rollback的处理方法决定是否存在，
`br_snapshots`为checkpointsnapshot，可以保存一整个映射表，maxBrCount默认为4

```scala
  val remap_pdsts = io.remap_reqs map (_.pdst)
  val remap_ldsts_oh = io.remap_reqs map (req => UIntToOH(req.ldst) & Fill(numLregs, req.valid.asUInt))

  val com_remap_pdsts = io.com_remap_reqs map (_.pdst)
  val com_remap_ldsts_oh = io.com_remap_reqs map (req => UIntToOH(req.ldst) & Fill(numLregs, req.valid.asUInt))
```
修改重命名表的请求对应寄存器号的独热码，
函数式编程实在不太容易一眼看出端倪，如果用verilog描述应该会很好理解。

```scala
  // Figure out the new mappings seen by each pipeline slot.
  for (i <- 0 until numLregs) {
    val remapped_row = (remap_ldsts_oh.map(ldst => ldst(i)) zip remap_pdsts)
      .scanLeft(map_table(i)) {case (pdst, (ldst, new_pdst)) => Mux(ldst, new_pdst, pdst)}

    val com_remapped_row = (com_remap_ldsts_oh.map(ldst => ldst(i)) zip com_remap_pdsts)
      .scanLeft(com_map_table(i)) {case (pdst, (ldst, new_pdst)) => Mux(ldst, new_pdst, pdst)}

    for (j <- 0 until plWidth+1) {
      remap_table(j)(i) := remapped_row(j)
      com_remap_table(j)(i) := com_remapped_row(j)
    }
  }
```
这部分是算出了`remap_table`和`com_remap_table`的中间态，也就是每条指令执行完后相应的map_table长啥样，
这样在有多条分支指令时可以填入当前每条指令的snapshot，`remap_table`的第一维长度为`plWidth + 1`，
第0个元素即为原来的`map_table`

```scala
  // Create snapshots of new mappings.
  if (enableSuperscalarSnapshots) {
    for (i <- 0 until plWidth+1) {
      when (io.ren_br_tags(i).valid) {
        br_snapshots(io.ren_br_tags(i).bits) := remap_table(i)
      }
    }
  } else {
    assert(PopCount(io.ren_br_tags.map(_.valid)) <= 1.U)
    val do_br_snapshot = io.ren_br_tags.map(_.valid).reduce(_||_)
    val br_snapshot_tag   = Mux1H(io.ren_br_tags.map(_.valid), io.ren_br_tags.map(_.bits))
    val br_snapshot_table = Mux1H(io.ren_br_tags.map(_.valid), remap_table)
    when (do_br_snapshot) {
      br_snapshots(br_snapshot_tag) := br_snapshot_table
    }
  }

```
这部分就是在保存每条分支指令的snapshot，一种是一次可以有多条分支指令，
另一种是只能有一条。最开始没看懂为啥循环长度为`plWidth + 1`，
输入的`ren_br_tags`长度也为`plWidth + 1`，而ex3版本这部分循环长度为`plWidth`，然后发现

```scala
  ren2_br_tags(0).valid := false.B
  ren2_br_tags(0).bits  := DontCare
```

```scala
    ren2_br_tags(w+1).valid := ren2_fire(w) && ren2_uops(w).allocate_brtag
    ren2_br_tags(w+1).bits  := ren2_uops(w).br_tag
```

输入的0端口其实就是无效的，其他每部分都往后偏移一个单位，其实真正有效的输入只有`plWidth`个，
然后每个输入都往后挪一个单位就行，这么处理可能是为了代码更好看，多出来的信号应该是会被优化掉的。

```scala
for (i <- 0 until plWidth) {
    io.map_resps(i).prs1       := (0 until i).foldLeft(map_table(io.map_reqs(i).lrs1)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === io.map_reqs(i).lrs1, io.remap_reqs(k).pdst, p))
    io.map_resps(i).prs2       := (0 until i).foldLeft(map_table(io.map_reqs(i).lrs2)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === io.map_reqs(i).lrs2, io.remap_reqs(k).pdst, p))
    io.map_resps(i).prs3       := (0 until i).foldLeft(map_table(io.map_reqs(i).lrs3)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === io.map_reqs(i).lrs3, io.remap_reqs(k).pdst, p))
    io.map_resps(i).stale_pdst := (0 until i).foldLeft(map_table(io.map_reqs(i).ldst)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === io.map_reqs(i).ldst, io.remap_reqs(k).pdst, p))

    if (!float) io.map_resps(i).prs3 := DontCare
  }
```
这部分处理的是`map_table`的前递，实际上输入的`remap_reqs`并非当前的`map_reqs`对应的pdst，
这部分是上一个周期的指令分配到的目的寄存器，此时还未将映射关系写入`map_table`
后面在`rename-stage`中还需要处理同时重命名的指令的前递问题。

### Free List
```scala
  val free_list = RegInit(UInt(numPregs.W), io.initial_allocation)
  val spec_alloc_list = RegInit(0.U(numPregs.W)) // free_list和~spec_alloc_list的区别？
  val br_alloc_lists = Reg(Vec(maxBrCount, UInt(numPregs.W)))
```
Free List有三部分，`free_list`保存当前空闲的寄存器，
`spec_alloc_list`保存目前处于推测状态的分配的寄存器，
`br_alloc_lists`对应分支tag的snapshot。


```scala
  spec_alloc_list := (spec_alloc_list | alloc_masks(0)) & ~dealloc_mask & ~com_despec
```
<br>

```scala
  val rollback_deallocs = spec_alloc_list & Fill(n, io.rollback)
```

最开始没太看懂`spec_alloc_list`的作用，总感觉似乎和`free_list`只是取反的一个关系，
实际并非如此，目前我的理解是`spec_alloc_list`用于rollback，也就是消除当前的推测状态，
即上面的`rollback_deallocs`，一条指令的pdst在ROB提交时即消除了推测的分配状态，
因此在`spec_alloc_list`中要置为0，
但是这个pdst只有作为某条指令的`stale_pdst`被提交时才会被释放放回`free_list`，
所以`spec_alloc_list`和`~free_list`并不相等，输入的com_despec即为当前提交指令的pdst

```scala
  // Update branch snapshots
  for (i <- 0 until maxBrCount) {
    val updated_br_alloc_list = if (isImm) {
      // Immediates clear the busy table when they read, potentially before older branches resolve.
      // Thus the branch alloc lists must be updated as well
      br_alloc_lists(i) & ~br_deallocs & ~com_deallocs | alloc_masks(0)
    } else {
      br_alloc_lists(i) & ~br_deallocs | alloc_masks(0)
    }
    br_alloc_lists(i) := updated_br_alloc_list
  }
```
这部分就是更新一下`br_alloc_lists`，但是这个`isImm`目前看不太明白，
它对应了`ImmRenameStage`类，字面意思理解似乎是立即数操作的重命名，
对比了一下这部分是v4新增的部分，网上没有太多资料

```scala
  if (enableSuperscalarSnapshots) {
    val br_slots = VecInit(io.ren_br_tags.map(tag => tag.valid)).asUInt
    // Create branch allocation lists.
    for (i <- 0 until maxBrCount) {
      val list_req = VecInit(io.ren_br_tags.map(tag => tag.bits === i.U)).asUInt & br_slots
      val new_list = list_req.orR
      when (new_list) {
        br_alloc_lists(i) := Mux1H(list_req, alloc_masks)
      }
    }
  } else {
    assert(PopCount(io.ren_br_tags.map(_.valid)) <= 1.U)
    val do_br_snapshot = io.ren_br_tags.map(_.valid).reduce(_||_)
    val br_snapshot_tag   = Mux1H(io.ren_br_tags.map(_.valid), io.ren_br_tags.map(_.bits))
    val br_snapshot_list  = Mux1H(io.ren_br_tags.map(_.valid), alloc_masks)
    when (do_br_snapshot) {
      br_alloc_lists(br_snapshot_tag) := br_snapshot_list
    }
  }

```
这里对分支br_alloc_lists的记录巧用了前面的`scanRight`，
`alloc_masks`每个元素对应的恰好就是该指令以及之后的指令alloc到的物理寄存器，
函数式编程还真有用，虽然还是不好理解

### Busy Table
```scala
  val wakeups = io.wakeups.map { w =>
    val wu = Wire(Valid(new Wakeup))
    wu.valid := RegNext(w.valid) && ((RegNext(w.bits.speculative_mask) & io.child_rebusys) === 0.U)
    wu.bits  := RegNext(w.bits)
    wu
  }
```
wakeup的信息会先寄存一拍再处理

```scala
  // Read the busy table.
  for (i <- 0 until plWidth) {
    val prs1_match = wakeups.map { w => w.valid && w.bits.uop.pdst === io.ren_uops(i).prs1 }
    val prs2_match = wakeups.map { w => w.valid && w.bits.uop.pdst === io.ren_uops(i).prs2 }
    val prs3_match = wakeups.map { w => w.valid && w.bits.uop.pdst === io.ren_uops(i).prs3 }

    io.busy_resps(i).prs1_busy := busy_table(io.ren_uops(i).prs1)
    io.busy_resps(i).prs2_busy := busy_table(io.ren_uops(i).prs2)
    io.busy_resps(i).prs3_busy := busy_table(io.ren_uops(i).prs3)

    when (prs1_match.reduce(_||_)) {
      io.busy_resps(i).prs1_busy := Mux1H(prs1_match, wakeups.map { w => w.valid && w.bits.rebusy })
    }
    when (prs2_match.reduce(_||_)) {
      io.busy_resps(i).prs2_busy := Mux1H(prs2_match, wakeups.map { w => w.valid && w.bits.rebusy })
    }
    when (prs3_match.reduce(_||_)) {
      io.busy_resps(i).prs3_busy := Mux1H(prs3_match, wakeups.map { w => w.valid && w.bits.rebusy })
    }

    if (!float) io.busy_resps(i).prs3_busy := false.B

  }
```

`busy_table`的读逻辑会判断当前寄存下来的unbusy寄存器号是否和新增的rebusy寄存器号一致，
如果一致则会有一个前递逻辑，理论上刚unbusy的寄存器估计也不会马上被释放，我估计这种情况不会太多

## rename-stage

BOOM有三种Rename模块，用于浮点运算和普通整型运算的`RenameStage`，用于SFB优化的`PredRenameStage`
以及貌似是用于优化立即数操作的`ImmRenameStage`，后两个目前不深入研究，主要看常规的`RenameStage`

重命名阶段被切分为了两级流水线，
第一级完成读`map_table`，以及向`free_list`发出读请求，
第二级向`map_table`发出更新请求，同时用`free_list`申请到的寄存器完成同时重命名指令间的旁路判断，
也同时向`map_table`和`free_list`发送`br_tag`，读`busy_table`。


```scala
  val io = IO(new Bundle {
    val ren_stalls = Output(Vec(plWidth, Bool()))

    val kill = Input(Bool())

    val dec_fire  = Input(Vec(plWidth, Bool())) // will commit state updates
    val dec_uops  = Input(Vec(plWidth, new MicroOp()))

    // physical specifiers available AND busy/ready status available.
    val ren2_mask = Vec(plWidth, Output(Bool())) // mask of valid instructions
    val ren2_uops = Vec(plWidth, Output(new MicroOp()))

    // branch resolution (execute)
    val brupdate = Input(new BrUpdateInfo())

    val dis_fire  = Input(Vec(coreWidth, Bool()))
    val dis_ready = Input(Bool())

    // wakeup ports
    val wakeups = Flipped(Vec(numWbPorts, Valid(new Wakeup)))
    val child_rebusys = Input(UInt(aluWidth.W))

    // commit stage
    val com_valids = Input(Vec(plWidth, Bool()))
    val com_uops = Input(Vec(plWidth, new MicroOp()))
    val rollback = Input(Bool())

    val debug_rob_empty = Input(Bool())
  })
```

rename-stage的输入主要来自decode部分，其次有来自执行阶段的分支信息以及唤醒信息，
`child_rebusys`的作用目前不太清楚，还有来自下一流水级的控制信号`dis_fire`和`dis_ready`
以及来自`commit`阶段的信息，输出的是重命名后的指令微操作


```scala
  override def BypassAllocations(uop: MicroOp, older_uops: Seq[MicroOp], alloc_reqs: Seq[Bool]): MicroOp = {
    if (older_uops.size == 0) {
      uop
    } else {
      val bypassed_uop = Wire(new MicroOp)
      bypassed_uop := uop

      val bypass_hits_rs1 = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.lrs1 }
      val bypass_hits_rs2 = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.lrs2 }
      val bypass_hits_rs3 = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.lrs3 }
      val bypass_hits_dst = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.ldst }

      val bypass_sel_rs1 = PriorityEncoderOH(bypass_hits_rs1.reverse).reverse
      val bypass_sel_rs2 = PriorityEncoderOH(bypass_hits_rs2.reverse).reverse
      val bypass_sel_rs3 = PriorityEncoderOH(bypass_hits_rs3.reverse).reverse
      val bypass_sel_dst = PriorityEncoderOH(bypass_hits_dst.reverse).reverse

      val do_bypass_rs1 = bypass_hits_rs1.reduce(_||_)
      val do_bypass_rs2 = bypass_hits_rs2.reduce(_||_)
      val do_bypass_rs3 = bypass_hits_rs3.reduce(_||_)
      val do_bypass_dst = bypass_hits_dst.reduce(_||_)

      val bypass_pdsts = older_uops.map(_.pdst)

      when (do_bypass_rs1) { bypassed_uop.prs1       := Mux1H(bypass_sel_rs1, bypass_pdsts) }
      when (do_bypass_rs2) { bypassed_uop.prs2       := Mux1H(bypass_sel_rs2, bypass_pdsts) }
      when (do_bypass_rs3) { bypassed_uop.prs3       := Mux1H(bypass_sel_rs3, bypass_pdsts) }
      when (do_bypass_dst) { bypassed_uop.stale_pdst := Mux1H(bypass_sel_dst, bypass_pdsts) }

      bypassed_uop.prs1_busy := uop.prs1_busy || do_bypass_rs1
      bypassed_uop.prs2_busy := uop.prs2_busy || do_bypass_rs2
      bypassed_uop.prs3_busy := uop.prs3_busy || do_bypass_rs3

      if (int) {
        bypassed_uop.prs3      := DontCare
        bypassed_uop.prs3_busy := false.B
      }

      bypassed_uop
    }
  }
```

```scala
    val bypassed_uop = Wire(new MicroOp)
    bypassed_uop := BypassAllocations(ren2_uops(w), ren2_uops.take(w), ren2_alloc_reqs.take(w))

    io.ren2_uops(w) := GetNewUopAndBrMask(bypassed_uop, io.brupdate)
```

这段很长的一段bypass处理的就是同时重命名的指令之间的写后读关系，`older_uops`即为`ren2_uops`的前几项，
多个写后读只取最近那一个值，所以用`PriorityEncoderOH`

```scala
    when (io.kill) {
      r_valid := false.B
    } .elsewhen (ren2_ready) {
      r_valid := ren1_fire(w)
      next_uop := ren1_uops(w)
    } .otherwise {
      r_valid := r_valid && !ren2_fire(w) // clear bit if uop gets dispatched
      next_uop := r_uop
    }
```

```scala
  val dis_stalls = dis_hazards.scanLeft(false.B) ((s,h) => s || h).takeRight(coreWidth)
  dis_fire := dis_valids zip dis_stalls map {case (v,s) => v && !s}
  dis_ready := !dis_stalls.last
```

这部分是rename阶段流水线寄存器的控制逻辑，kill时将valid置为false，根据输入的`dis_ready`可以看出
只有当dispatch部分可以接受所有指令时`dis_ready`才拉高，而`dis_fire`则是每个dispatch的端口
都有对应的信号，所以当`dis_ready`拉高时`rename-stage`中的寄存器可以接受新的一批指令，
否则能dispatch的指令会先被dispatch，存在hazard的指令暂存保持不变
然后等这一批指令都能dispatch，即`dis_ready`拉高后才能接受下一批指令

```scala
    r_uop := GetNewUopAndBrMask(BypassAllocations(next_uop, ren2_uops, ren2_alloc_fire), io.brupdate)
```

这部分对`next_uop`和`ren2_uops`之间做了一个前递处理以及一个目前还不太清楚的关于`br_mask`的操作。
按照我目前的理解，这部分前递实际上已经在`map_table`内部完成，似乎不是很有必要？

各模块间的连线基本都比较明确

```scala
  for (w <- 0 until plWidth) {
    val can_allocate = freelist.io.alloc_pregs(w).valid

    // Push back against Decode stage if Rename1 can't proceed.
    io.ren_stalls(w) := (ren2_uops(w).dst_rtype === rtype) && !can_allocate

    val bypassed_uop = Wire(new MicroOp)
    bypassed_uop := BypassAllocations(ren2_uops(w), ren2_uops.take(w), ren2_alloc_reqs.take(w))

    io.ren2_uops(w) := GetNewUopAndBrMask(bypassed_uop, io.brupdate)
  }
```

当物理寄存器个数不够时，对应端口会拉高`ren_stalls`信号，
然后`dis_ready`为false，从而使得`dec_ready`为false

