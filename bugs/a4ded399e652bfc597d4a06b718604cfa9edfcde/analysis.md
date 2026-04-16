# OCFS2 lockdep WARNING 分析

## 现象 / symptom

`report0` 显示在 `6.19.0-rc5-00042-g944aacb68baf` 上触发了
lockdep WARNING：

- `WARNING: possible circular locking dependency detected`
- 当前获取的新锁是
  `&ocfs2_sysfile_lock_key[LOCAL_ALLOC_SYSTEM_INODE]`
- 当前任务已经持有 `&oi->ip_alloc_sem`

lockdep 给出的闭环核心是：

`LOCAL_ALLOC_SYSTEM_INODE inode_lock`
`-> ip_xattr_sem -> ip_alloc_sem -> LOCAL_ALLOC_SYSTEM_INODE inode_lock`

这说明 OCFS2 内部至少存在一条可达的锁顺序反转路径，属于潜在
死锁问题，而不只是普通健壮性告警。

## Bug cause

这个问题的核心很直接：

写路径里，`ocfs2_write_begin()` 先拿目标 inode 的
`ip_alloc_sem`，随后在本地分配器路径中进入
`ocfs2_reserve_local_alloc_bits()`，再去拿 local alloc system
inode 的 `inode_lock`。

而另一条路径里，创建/ACL/xattr 相关流程已经建立了相反方向的依赖：
先拿到 system inode / journal 相关锁，再进入
`ip_xattr_sem`，并在 xattr 清理路径中继续获取 `ip_alloc_sem`。

两条路径合起来形成闭环，所以这更像是 OCFS2 内部锁层级定义不一致，
而不是单纯 fuzz 人为拼出的假阳性。

## 触发路径 / repro context

从 `report0` 看，告警直接发生在 buffered write 路径：

`ksys_write`
`-> vfs_write`
`-> ocfs2_file_write_iter`
`-> __generic_file_write_iter`
`-> generic_perform_write`
`-> ocfs2_write_begin`
`-> ocfs2_write_begin_nolock`
`-> ocfs2_lock_allocators`
`-> ocfs2_reserve_clusters_with_limit`
`-> ocfs2_reserve_local_alloc_bits`
`-> inode_lock(local_alloc_inode)`

此时任务已经持有：

- 普通 VFS inode 的 `inode_lock`
- `OCFS2_I(inode)->ip_alloc_sem`

lockdep 给出的反向依赖来源主要有两组：

1. xattr 删除路径

`ocfs2_xattr_set()`
`-> down_write(ip_xattr_sem)`
`-> ...`
`-> ocfs2_try_remove_refcount_tree()`
`-> down_write(ip_alloc_sem)`

2. 创建/ACL 路径

`ocfs2_mknod()`
`-> ocfs2_reserve_clusters()`
`-> ocfs2_start_trans()`
`-> ocfs2_init_acl()`
`-> down_read(dir->ip_xattr_sem)`

再结合卸载/本地分配器关闭路径上的
`journal->j_trans_barrier -> sb_internal -> LOCAL_ALLOC_SYSTEM_INODE`
依赖，最终把环闭合起来。

## 相关源码位置

关键位置如下：

- `fs/ocfs2/aops.c`
  - `ocfs2_write_begin()` 在进入写路径后先获取
    `OCFS2_I(inode)->ip_alloc_sem`
- `fs/ocfs2/localalloc.c`
  - `ocfs2_reserve_local_alloc_bits()` 获取 local alloc system inode，
    然后执行 `inode_lock(local_alloc_inode)`
- `fs/ocfs2/xattr.c`
  - `ocfs2_xattr_set()` 先获取 `ip_xattr_sem`
  - 在删除 xattr 成功后调用 `ocfs2_try_remove_refcount_tree()`
- `fs/ocfs2/refcounttree.c`
  - `ocfs2_try_remove_refcount_tree()` 的锁顺序是
    `ip_xattr_sem -> ip_alloc_sem`
- `fs/ocfs2/acl.c`
  - `ocfs2_init_acl()` 在新建 inode 流程中读取父目录默认 ACL，
    会获取 `dir->ip_xattr_sem`
- `fs/ocfs2/namei.c`
  - `ocfs2_mknod()` 在保留 clusters、开启事务后调用
    `ocfs2_init_acl()`

## 根因分析 / root cause analysis

根因不是单个函数漏解锁，而是 OCFS2 内多条元数据路径对锁层级的假设
并不一致。

更具体地说：

1. 写路径把 `ip_alloc_sem` 视为“先拿的 inode 内部资源锁”
   然后再深入到 local allocator，并对 system inode 加
   `inode_lock`。

2. xattr / ACL / refcount 清理路径把 `ip_xattr_sem` 和
   `ip_alloc_sem` 当成另一组内部层级使用，其中
   `ocfs2_try_remove_refcount_tree()` 明确要求
   `ip_xattr_sem -> ip_alloc_sem`。

3. local allocator 不是普通对象，而是 `LOCAL_ALLOC_SYSTEM_INODE`
   对应的 system inode。它和事务、superblock 内部锁、卸载路径
   之间本来就有额外依赖，因此一旦写路径在持有 `ip_alloc_sem`
   后再去拿它的 `inode_lock`，就很容易和创建/ACL/xattr 路径形成
   跨对象闭环。

换句话说，问题本质是：

- 普通 inode 的 `ip_alloc_sem`
- 普通 inode 的 `ip_xattr_sem`
- local alloc system inode 的 `inode_lock`
- journal/sb 相关锁

这几类锁没有被维护成一个全局一致的偏序。

## 可能的修复方向 / possible fix

比较现实的修复方向有三类：

1. 调整写路径的锁边界

尽量避免在持有普通 inode 的 `ip_alloc_sem` 时进入
`ocfs2_reserve_local_alloc_bits()` 并获取 local alloc system inode
的 `inode_lock`。如果能把 local alloc 预留提前，或拆到
`ip_alloc_sem` 之外，最直接。

2. 明确并统一 `ip_xattr_sem` / `ip_alloc_sem` 的层级

如果 OCFS2 设计上要求 xattr 永远在 alloc 之前，或反之，就需要把
写路径、xattr 路径、refcount 清理路径统一到同一个顺序，避免当前
一部分代码隐含 `alloc -> system inode`，另一部分代码隐含
`system inode/journal -> xattr -> alloc`。

3. 避免在 xattr 删除后同步进入 refcount tree 清理

`ocfs2_xattr_set()` 在释放 `ip_xattr_sem` 后立即调用
`ocfs2_try_remove_refcount_tree()`。虽然这里已经释放了
`ip_xattr_sem`，但 lockdep 历史依赖说明这条路径仍在整体锁图中起到
关键作用。把 refcount tree 的删除改成更晚的、独立上下文中的清理，
也许能打断环的一边。

## 结论

我的判断是：这更倾向于**源代码里的真实锁序问题**，不是“刻意测试
条件导致的古怪 WARNING”。

理由：

- lockdep 给出的闭环涉及多个正常元数据路径，而不是单条非常牵强的
  异常分支。
- 相关调用链都在 OCFS2 正常功能范围内：写入、创建、ACL/xattr、
  refcount 清理、local allocator、事务/卸载。
- `ocfs2_try_remove_refcount_tree()` 和 `ocfs2_write_begin()`
  的锁顺序从源码上看确实不一致，具备明确代码依据。

建议：

- **值得向 maintainer 发 report**
- **值得讨论并准备 patch**
- 优先级我认为是**中等**，因为当前看到的是 lockdep WARNING，
  不是已确认的现场死锁，但它揭示的是潜在真实死锁环，不应忽略

