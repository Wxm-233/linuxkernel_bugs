## 现象 / symptom

在 `6.18.5` 上挂载并恢复一个由 syzkaller-style workload 触发的 GFS2 镜像时，`gfs2_recovery` workqueue 在 journal head 扫描阶段触发 `kernel BUG at block/bio.c:341`。调用栈为 `gfs2_recover_func()` -> `gfs2_find_jhead()` -> `gfs2_chain_bio()` -> `bio_chain()`。

## 触发路径 / repro context

从日志看，故障发生在 GFS2 recovery 线程处理 `jid=0` 的 journal 时。`gfs2_find_jhead()` 会遍历 journal extents，把日志块按 folio/page 聚合成 bio 读取；当当前 folio 已经有一部分被放进旧 bio，剩余部分又必须切到新 bio 时，会走 `off != 0` 的分裂路径，并调用 `gfs2_chain_bio()`。

这个路径通常要求文件系统块大小小于 `PAGE_SIZE`，并且 journal extent / bio 拼接状态恰好让同一 folio 被拆到两个 bio 中，因此普通配置不常见；但它并不是非法状态，fuzzing 用构造镜像可以稳定打到。

## 相关源码位置

- `linux-7.0/fs/gfs2/recovery.c:457` `gfs2_recover_func()`
- `linux-7.0/fs/gfs2/lops.c:394` `gfs2_end_log_read()`
- `linux-7.0/fs/gfs2/lops.c:484` `gfs2_chain_bio()`
- `linux-7.0/fs/gfs2/lops.c:500` `gfs2_find_jhead()`
- `linux-7.0/block/bio.c:381` `bio_chain()`

## Bug cause

`gfs2_find_jhead()` 在“同一 folio 需要拆到两个 read bio”时，把新 bio 传给了 `bio_chain(new, prev)`。但 GFS2 的 journal read 路径要求每个 bio 都保留自己的 `gfs2_end_log_read()` 和 `bi_private`，否则该 bio 覆盖到的 folio 就不会在 I/O 完成后被正确 `folio_end_read()`。

也就是说，这里把 block layer 的 chaining 语义用错了。该路径一旦被打到，要么像本次一样直接在 `bio_chain()` 里 `BUG`，要么即使不 `BUG`，child bio 的完成处理也会偏离 GFS2 预期。

## 根因分析 / root cause analysis

`gfs2_log_alloc_bio()` 为 journal read bio 设置了 `bi_end_io = gfs2_end_log_read`，而 `gfs2_end_log_read()` 会遍历该 bio 覆盖的所有 folio，记录错误并执行 `folio_end_read()`。这说明“每个 read bio 都必须自己执行 completion 回调”是 GFS2 逻辑的一部分。

但在 `gfs2_chain_bio()` 中，GFS2 为了把旧 bio 提交出去、同时开始构造下一个 bio，调用了 `bio_chain(new, prev)`。`bio_chain()` 会把 child bio 的 `bi_end_io` 改成 block layer 内部的 `bio_chain_endio`，并把 completion 关系挂到 parent 上。这种模式适用于 child bio 本身不再需要单独的 `bi_end_io` 语义的场景，例如 block 层的 split bio；而这里的 GFS2 read bio 恰恰需要保留 `gfs2_end_log_read()` 去解锁自己的 folio，因此 chaining 语义与 GFS2 的 read completion 需求冲突。

换句话说，问题不是“日志里偶然 warning 了一下”，而是 `gfs2_find_jhead()` 在一个合法但不常走到的 journal read 分裂路径上误用了 `bio_chain()`。syzkaller-style workload 只是把这个冷门路径放大了出来。

从当前 `linux-7.0` 源码看，这段实现仍然存在，没有看到已经在调用点被改成“每个新 bio 继续保留 `gfs2_end_log_read()` / `bi_private`”的直接证据，因此我不认为这是主线已经自然弱化掉的问题。

## 可能的修复方向 / possible fix

最直接的修复方向是：在 `gfs2_chain_bio()` 里不要对新的 journal read bio 调用 `bio_chain()`，而是保留 `prev` 的 `bi_end_io` / `bi_private` 给 `new`，然后单独提交 `prev`。这样每个 read bio 仍然会各自执行 `gfs2_end_log_read()`，与 GFS2 当前的 folio 完成处理逻辑一致。

一个可行的 draft patch 方向是：

- 新 bio 继续用 `bio_alloc()` 分配；
- `bio_clone_blkg_association(new, prev)` 保留；
- `new->bi_end_io = prev->bi_end_io`；
- `new->bi_private = prev->bi_private`；
- 删除 `bio_chain(new, prev)`，只提交 `prev`。

这个修复方向和当前崩溃栈、以及 `gfs2_end_log_read()` 的语义是一致的。但我没有在本地完成运行验证，因此仍有两点不确定性需要在邮件里明确写出：

- 它是基于当前材料给出的最可能修复，不保证已经是最终正确方案；
- 当前 helper 仍然保留“先分配 new，再提交 prev”的结构，这在 memory pressure 语义上是否还需要进一步调整，值得 maintainer 再看一眼。

## 值不值得提报

我明确倾向于把它归类为**源码问题**，不是单纯“找茬式”的古怪 warning。理由是：

- 触发点在 GFS2 对 `bio_chain()` 的使用方式本身，而不是日志打印过严；
- 该路径虽然冷门，但依赖的是合法的 journal read 分裂条件，不是内核外部注入的非法内存状态；
- 当前主线源码里这段调用模式仍在。

是否值得提 bug/patch：**值得提 report，也值得附上 draft patch**。不过它更像“冷门恢复路径中的真实逻辑 bug”，不是高频线上场景，因此优先级我会定为**中低**，而不是紧急高优先级。

## 简化复现说明

根据现有日志，我只能比较可靠地归纳为：“挂载一个会触发 GFS2 journal recovery 的构造镜像，并让 `gfs2_find_jhead()` 进入 split-folio read 路径”。目前**无法从现有材料中可靠提炼出一个简短、可信的最小 syscall reproducer 或 C reproducer**，因此不建议编造一个看起来像但实际上未必能触发的复现程序。
