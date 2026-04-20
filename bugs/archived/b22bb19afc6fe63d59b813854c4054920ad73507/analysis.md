# possible deadlock in start_dirop

## 现象 / symptom

`report0` 显示 lockdep 报出 `WARNING: possible circular locking dependency detected`，复现内核为 `6.19.0-rc5-00042-g944aacb68baf`。告警点落在 `start_dirop()`，具体是在 `lookup_one_qstr_excl()` 为新 dentry 分配内存时，经由 `kmem_cache_alloc_lru_noprof()` 引入了 `fs_reclaim`，而当前线程已经持有目录 inode 的 `i_rwsem`。同一 bug 目录下后续样本还显示该告警在 `7.0.0-rc2-g0031c06807cf` 仍然存在，因此它不像是已经在新版本中自然消失的问题。

## 触发路径 / repro context

从 `report0` 的调用栈看，触发路径是：

`nl80211_new_key()` -> `rdev_add_key()` -> `ieee80211_add_key()` -> `ieee80211_key_link()` -> `ieee80211_debugfs_key_add()` -> `debugfs_create_dir()` -> `debugfs_start_creating()` -> `simple_start_creating()` -> `start_dirop()` -> `lookup_one_qstr_excl()` -> `d_alloc()` -> `kmem_cache_alloc_lru_noprof()`

这里的关键点不只是 “debugfs 创建目录会分配内存”，而是这个分配发生在已经持锁的路径里。`nl80211_pre_doit()` 会在整个 `nl80211_new_key()` 执行期间保持 `wiphy.mtx`，而 `start_dirop()` 在目录操作期间又会拿父目录 inode 的 `i_rwsem`。于是，新增 key 时顺手创建 debugfs 目录，就把 “无线子系统锁 + VFS 目录锁 + reclaim 语义” 串在了一起。

## 相关源码位置

- `net/wireless/nl80211.c:18046` `nl80211_pre_doit()`：进入 `nl80211` 命令后获取并保持 `wiphy.mtx`
- `net/wireless/nl80211.c:5153` `nl80211_new_key()`：处理新增 key 的 netlink 请求
- `net/mac80211/key.c:848` `ieee80211_key_link()`：将 key 挂到 mac80211 状态中
- `net/mac80211/key.c:943`：成功后立即调用 `ieee80211_debugfs_key_add(key)`
- `net/mac80211/debugfs_key.c:324` `ieee80211_debugfs_key_add()`：为 key 创建 debugfs 节点
- `net/mac80211/debugfs_key.c:336`：调用 `debugfs_create_dir()`
- `fs/debugfs/inode.c:362` `debugfs_start_creating()`：进入 debugfs 创建流程
- `fs/debugfs/inode.c:570` `debugfs_create_dir()`：为 debugfs 目录分配 inode/dentry
- `fs/namei.c:2916` `__start_dirop()`：拿父目录 `i_rwsem`
- `fs/namei.c:2937` `start_dirop()`：目录操作入口
- `kernel/relay.c:352` `relay_create_buf_file()`、`kernel/relay.c:380` `relay_open_buf()`、`kernel/relay.c:437` `relay_prepare_cpu()`、`kernel/relay.c:474` `relay_open()`：已有另一条锁链在 `relay_channels_mutex` 持锁期间进入 debugfs/VFS 创建路径

## Bug cause

这个 warning 的核心原因是：mac80211 在 “添加 key 的受锁关键路径” 里做了非必要的 debugfs 目录创建，而 debugfs/VFS 创建目录时会在持有父目录 `i_rwsem` 的情况下进行可能触发 reclaim 的内存分配。与此同时，系统里已经存在 `relay_channels_mutex -> debugfs_create_*() -> i_rwsem` 这条锁依赖链。两边一拼，lockdep 就得到 `fs_reclaim -> relay_channels_mutex -> i_rwsem -> fs_reclaim` 的闭环。

换句话说，问题不在于日志里单纯“碰巧 warning 了”，而在于一个本来可以延后或异步做的 debugfs 操作，被放进了持锁的 key 安装路径里，从而和 VFS/reclaim 语义发生了真实的锁顺序冲突。

## 根因分析 / root cause analysis

`ieee80211_key_link()` 的主体职责是把新 key 接入运行中的 mac80211 状态机，并替换旧 key。这个动作本身处在 `nl80211` 的受锁调用链中，`wiphy.mtx` 已经由 `nl80211_pre_doit()` 持有。函数在 `ieee80211_key_replace()` 成功后，马上执行 `ieee80211_debugfs_key_add()`。这一步属于调试可观测性增强，不是功能正确性所必需的状态更新。

`ieee80211_debugfs_key_add()` 里先创建 key 对应的 debugfs 目录，再创建一组属性文件。目录创建会走到 `debugfs_create_dir()`，而 debugfs 底层又通过 `simple_start_creating()` / `start_dirop()` 进入普通 VFS 目录创建前置流程。`__start_dirop()` 在拿到父目录 inode 的 `i_rwsem` 后做 `lookup_one_qstr_excl()`，后者会分配 dentry，因此 lockdep 将其视为 `fs_reclaim` 相关分配点。

`report0` 里已经给出了现成的反向依赖链：`relay_open_buf()` 在 `relay_channels_mutex` 持锁时也会通过 `relay_create_buf_file()` 进入 debugfs 创建路径，并最终获取同类目录 inode 锁。因此，只要另一条路径先拿了目录锁又进入 reclaim，理论上的死锁环就成立。syzkaller-style workload 只是把这条链条更高频地压了出来，但锁顺序本身并不依赖某个明显不现实的伪造前提。

## 可能的修复方向 / possible fix

最直接的修复思路，是不要在 `ieee80211_key_link()` 的持锁关键路径里同步创建 per-key debugfs 节点。更合理的方式是把 `ieee80211_debugfs_key_add()` 延后到锁外，或者投递到一个安全的异步上下文中完成。因为 debugfs 只影响调试可见性，不影响 key 的功能正确性，所以它不应与 key 安装动作绑定得这么紧。

如果短期内很难调整调用时序，另一类思路是尽量约束这段路径的分配语义，避免把普通 `GFP_KERNEL` reclaim 语义带进目录创建流程；但这条路会更脆弱，因为 dcache / debugfs / VFS 内部分配链较长，控制面不集中。相比之下，把 debugfs 创建移出受锁路径通常更清晰，也更容易向上游解释。
