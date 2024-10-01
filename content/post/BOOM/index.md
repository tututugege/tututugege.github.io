---
title: RISCV-BOOM后端学习-01 Rename
description: BOOM处理器的学习记录
date: 2024-10-01 02:24:00.000 +0800
tags:
    - BOOM
    - CPU

categories:
    - CPU微架构
---

## 背景

最近要写超标量处理器的后端执行部分，看了姚永斌老师的《超标量处理器设计》后准备大展身手，结果发现很多细节实现方面存在问题，比如分支misprediction时各模块间如何配合清除错误的指令且不影响正常执行，因此想找一个开源的处理器学习一下，香山的代码相对来说比较复杂，最后选择了以更简单一点的BOOM作为参考，纯靠眼睛看代码结合文档理解，并且本人不太熟悉函数式编程，一些地方可能理解有误。

## 重命名

BOOM采用的是统一的物理寄存器实现重命名的设计，也就是所谓的“explicit renaming”，只有PRF，然后用单独的map table来管理映射关系，而另一种“implicit renaming”则是将结果写入ROB，然后ROB提交时才写回逻辑寄存器。

我原以为显式重命名应该是更先进的方式，但文档上说P4，ARM A57采用的是隐式重命名，更老的的MIPS R10K，Alpha 21264采用的是显式重命名，说明这两种设计算是各有取舍，上网查了一下Core架构和Zen架构都有显式的Rename alias table。

重命名模块的组成成分可以分为：Map Table、Free List、Busy Table三部分。

### Map Table

BOOM的map_table是常规的用逻辑寄存器号寻址的结构，我原以为能容纳更多checkpoint的CAM内容寻址结构会是主流，但根据目前看过的资料发现这种结构似乎不常用？典型例子我只看过姚老师书上的Alpha21264。


BOOM同时保存了推测执行的重命名表和提交之后才变化的committed map table，分支指令采用checkpoint/ snapshot保存当前的重命名表，misprediction时用其恢复，这种实现方面会导致checkpoint非常占面积，并且容纳的分支指令有限，不过free_list以及用于发射队列和ROB恢复的分支TAG也是有限的，在这种情况下即使重命名表checkpoint足够多也没法容纳过量的分支指令。


这部分代码还算比较好理解
```scala
  val map_table = RegInit(VecInit((0 until numLregs) map { i => i.U(pregSz.W) }))
  val com_map_table = RegInit(VecInit((0 until numLregs) map { i => i.U(pregSz.W) }))
  val br_snapshots = Reg(Vec(maxBrCount, Vec(numLregs, UInt(pregSz.W))))
```
`map_table`为推测执行的映射表，`com_map_table`为提交后更新的architecture map table，`br_snapshots`为checkpoint/snapshot，可以保存一整个映射表，maxBrCount默认为4，参数在`maxBrCount`中。
```scala
  val remap_pdsts = io.remap_reqs map (_.pdst)
  val remap_ldsts_oh = io.remap_reqs map (req => UIntToOH(req.ldst) & Fill(numLregs, req.valid.asUInt))

  val com_remap_pdsts = io.com_remap_reqs map (_.pdst)
  val com_remap_ldsts_oh = io.com_remap_reqs map (req => UIntToOH(req.ldst) & Fill(numLregs, req.valid.asUInt))

```
每个表项，检查每个修改重命名表的请求地址独热码的对应位来选择表项的下一个值，函数式编程实在不太容易一眼看出端倪，如果用verilog描述应该会很好理解。


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
这部分处理了当前处理的几条指令存在RAW情况下后面读寄存器指令的操作数来源问题，以及WAW情况下后一条指令需要保存的旧的映射关系来源的问题，它们来自于分配给前一条写寄存器指令分配到的preg号而非map_table读出来的值（函数式编程还是不太容易看啊）


### Free List
```scala
  val free_list = RegInit(UInt(numPregs.W), io.initial_allocation)
  val spec_alloc_list = RegInit(0.U(numPregs.W)) // free_list和~spec_alloc_list的区别？
  val br_alloc_lists = Reg(Vec(maxBrCount, UInt(numPregs.W)))
```
Free List有三部分，free_list保存当前空闲的寄存器，spec_alloc_list保存目前处于推测状态的分配的寄存器，br_alloc_lists对应分支tag的snapshot。


```scala
  spec_alloc_list := (spec_alloc_list | alloc_masks(0)) & ~dealloc_mask & ~com_despec
```
<br>

```scala
  val rollback_deallocs = spec_alloc_list & Fill(n, io.rollback)
```

最开始没太看懂spec_alloc_list的作用，总感觉似乎和free_list只是取反的一个关系，实际并非如此，spec_alloc_list用于rollback，也就是消除当前的推测状态，即上面的rollback_deallocs，一条指令的pdst在ROB提交时即消除了推测的分配状态，因此在spec_alloc_list要置为0，但是这个pdst只有作为某条指令的stale_pdst被提交时才会被释放放回free_list，输入的com_despec即为当前提交指令的pdst

### Busy Table

### 分支、rollback处理

