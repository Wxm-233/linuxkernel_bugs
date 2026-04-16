# 8125 bug 分析

## 现象

- `description` 给出的现象是 `WARNING in set_cpu_sibling_map`。
- `report0`/`log0` 显示 warning 发生在 AP 启动早期，调用栈为 `start_secondary -> ap_starting -> set_cpu_sibling_map()`。
- 启动参数里包含 `numa=fake=2`，日志也明确显示系统在非 NUMA 机器上伪造了两个 node：
  - `Faking node 0 ...`
  - `Faking node 1 ...`
- 同一份日志里，CPU 拓扑又显示这是单 package、单 node、4 cores、每核 1 thread：
  - `Max. logical packages: 1`
  - `Num. nodes per package: 1`
  - `Max. threads per core: 1`
  - `Num. cores per package: 4`

## 成因

这个 warning 的本质是：`set_cpu_sibling_map()` 在建立 x86 拓扑关系时，会检查“被识别为 sibling / LLC 共享 / L2 共享的 CPU 是否位于同一个 NUMA node”。如果不在同一个 node，就通过 `topology_sane()` 触发 `WARN_ONCE()`。

相关代码位于：

- `arch/x86/kernel/smpboot.c`
  - `topology_sane()`：如果两个拓扑相关 CPU 不在同一 node，则 warning
  - `match_llc()` / `match_smt()` / `match_l2c()`：在建立 sibling 关系前调用 `topology_sane()`
- `arch/x86/mm/numa.c`
  - `numa_init_array()`：对于 fake NUMA / 无真实 CPU-node 映射的场景，按 round-robin 给 CPU 分配 node
  - 注释明确写到：在 “NUMA emulation and faking node case” 下，这种 round-robin 初始化被认为是 “good enough”
- `mm/numa_emulation.c`
  - `numa_emulation()` 会根据 `numa=fake=` 修改 emulated node
  - 注释明确说明它会把 `__apicid_to_node[]` 改写成 emulated nid

因此，这个 bug 并不是 CPU 拓扑枚举本身坏了，而是：

1. 机器本身并没有真实的多 node CPU 拓扑，日志中的 CPU topology 也说明 package 内只有 1 个 node。
2. `numa=fake=2` 只是在内存层面伪造了两个 NUMA node。
3. fake NUMA 路径又把 CPU 的 node 归属改写成 emulated node，或者在缺少真实映射时按 round-robin 分配。
4. 这样会让同一个 package/同一个 LLC 域中的 CPU 被分到不同的 fake node。
5. `set_cpu_sibling_map()` 仍然按“真实硬件拓扑校验”的标准要求它们必须同 node，于是触发 warning。

从日志看，`Max. threads per core: 1`，所以这次更可能是 `match_llc()` 里的 `topology_sane()` 检查被触发，而不是 SMT 检查；但无论具体是 LLC 还是其他 sibling 层级，本质矛盾都一样：**fake NUMA 的 CPU-node 映射与真实 package/cache 拓扑不一致。**

## 可能的修复手段

### 方案 1：在 fake NUMA / NUMA emulation 场景下降级或跳过该一致性 warning

这是最直接、风险最小的修复方式。

思路：

- 在 `topology_sane()` 或其调用者中识别当前是否处于 `CONFIG_NUMA_EMU` + `numa=fake=` 的 emulated node 场景。
- 对 fake NUMA 场景，不再把“拓扑相关 CPU 不在同一 node”视为异常，而是直接返回 `false`（忽略该依赖）或者不触发 `WARN_ONCE()`。

优点：

- 不改变真实硬件 NUMA 场景的校验强度。
- 与当前 fake NUMA 的语义一致：它本来就不是物理拓扑，只是测试/模拟用的逻辑 node。

### 方案 2：让 fake NUMA 下的 CPU-node 映射保持物理 CPU 拓扑一致

思路：

- 不要在 fake NUMA 下把同一个 package / LLC / core 域里的 CPU 拆到不同 emulated node。
- 对于没有真实 CPU-node 信息的机器，至少保证同一个 package 的 CPU 落到同一个 emulated node，而不是简单 round-robin。

问题：

- 这会让 fake NUMA 更复杂，因为它原本主要是“伪造内存 node”，不是“伪造一套自洽的 CPU+cache+node 拓扑”。
- 还需要决定：一个物理 package 只有 1 个 node 时，多个 fake node 应该如何承载 CPU 归属。这个语义未必自然。

因此这个方案更重，且容易引入新的调度/拓扑副作用。

## 建议

更合理的修复方向是 **方案 1**：把 fake NUMA 当作一种已知例外，不对其套用真实硬件 NUMA 一致性检查。

原因是这里的 warning 来自“检查假设过强”，不是实际拓扑损坏。对 `numa=fake=` 这种测试场景继续保留硬性 warning，只会把可预期的模拟配置误报成内核异常。
