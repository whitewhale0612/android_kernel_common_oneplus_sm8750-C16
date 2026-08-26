<div align="center">

# 🔥 OnePlus 13T SM8750 Custom Kernel

A performance-oriented custom kernel for the OnePlus 13T, based on Android Common Kernel 6.6.118.

[![Device][device-badge]][device-url]
![SoC][soc-badge]
[![Kernel][kernel-badge]][kernel-url]
![Platform][platform-badge]
![Compatibility][compatibility-badge]
![Release Policy][release-policy-badge]

</div>

[device-badge]: https://img.shields.io/badge/Device-OnePlus%2013T-EA0029?style=for-the-badge&logo=oneplus&logoColor=white
[device-url]: https://www.oneplus.com/
[soc-badge]: https://img.shields.io/badge/SoC-SM8750%20%7C%20Snapdragon%208%20Elite-red?style=for-the-badge
[kernel-badge]: https://img.shields.io/badge/Kernel-6.6.118-FCC624?style=for-the-badge&logo=linux&logoColor=black
[kernel-url]: https://www.kernel.org/
[platform-badge]: https://img.shields.io/badge/Platform-Android%20GKI-3DDC84?style=for-the-badge&logo=android&logoColor=white
[compatibility-badge]: https://img.shields.io/badge/Compatibility-%E6%AC%A7%2F%E5%8A%A0%2F%E7%9C%9F%208E%20%E7%90%86%E8%AE%BA%E9%80%9A%E7%94%A8-0078D4?style=for-the-badge
[release-policy-badge]: https://img.shields.io/badge/Source-Close%20but%20Free%20Release-F59E0B?style=for-the-badge

> [!IMPORTANT]
> The primary development and validation target is the OnePlus 13T (project `24821`).
> The repository also contains device-tree overwrite configurations for other OPLUS/OnePlus/realme
> 8 Elite devices, but theoretical compatibility does not mean that boot, functionality, or stability
> has been validated on those devices.

## ⚠️ Disclaimer

Flashing a custom kernel can cause **boot failure, data loss, or SafetyNet / Play Integrity failures**.
Back up the original kernel and important data before flashing. The author is not responsible for any
loss caused by using this kernel.

**By flashing this kernel, you acknowledge and accept these risks.**

## ✨ Core Features

### 🧊 Original Implementations

- **Crystal HybridSwap**: a private zram implementation that replaces standard `CONFIG_ZRAM` while
  retaining the `/dev/zramX`, `/sys/class/zram-control`, and common `/sys/block/zramX` userspace ABI,
  restoring the Hybridswap workflow.
- **ZMS packed backing store**: packs compressed objects into 4 KiB backing blocks and supports controlled
  writeback, batch-in, quotas, device-lifetime limits, and failure backoff. Unlike upstream compressed
  writeback, where zram first decompresses each slot into a full page and writes one backing block per page,
  ZMS stores the compressed stream directly and tries to place multiple objects in the same 4 KiB block.
  This reduces backend space usage and physical writes. Upstream does not merge compressed streams during
  writeback: regardless of the compressed size, one object consumes one 4 KiB block, while ZMS can save more space.
- **SDDC similarity compression**: uses aliases for identical compressed streams and deltas for similar
  streams, with fallback to the ordinary compression path when resources or validation are unavailable.
- **LZ4KD / LZ4KDS**: `LZ4KD` is a standalone ordinary compressor. The former “LZ4KD + SDDC”
  combination is now the standalone `LZ4KDS` algorithm, which is the default for the current 4 KiB
  build. `LZ4KD` and `LZ4KDS` are two distinct algorithms.
- **Resident SDDC compression**: manages validated alias/delta representations using reference
  generations, slot state, and wire-format checks; converted slots remain resident and are not written to ZMS.
- Supports idle, huge, and incompressible page writeback, prefetch, batch-in, per-memcg controls, eventfd
  pressure notifications, and layered statistics.
- Provides diagnostic nodes such as `hybridswap_report`, `hybridswap_crystal_stat`, `sddc_stat`, and `zms_stat`.

## 📊 Compression and Writeback Results

The following results are provided by this project. Higher ratios and throughput are better, while lower
latency is better. See the corresponding test record for the exact environment and dataset; these numbers
do not represent absolute performance on every device, temperature, or workload.

> [!WARNING]
> `LZ4KDS` still has a known readback stability issue: readback may return a wrong page or a zero page,
> which can crash applications or `system_server`. It is not recommended for daily use; the `LZ4KDS`
> result below is provided for testing reference only.

| Algorithm | Payload Ratio | Physical Ratio | Write MiB/s | 4K Rand Read MiB/s | 4K Mean / P99 |
|---|---:|---:|---:|---:|---:|
| LZ4 | 1.5715 | 1.5412 | 162.0 | 4985.5 | 5.70 / 9.28 µs |
| ZSTD | 1.9565 | 1.9121 | 28.6 | 1551.7 | 19.57 / 30.85 µs |
| LZ4KDS | **2.1750** | **2.1299** | 124.3 | 4412.2 | 6.53 / 12.22 µs |
| LZO-RLE | 1.5608 | 1.5328 | **168.4** | 4102.9 | 7.07 / 11.97 µs |

### 💾 Memory Reclaim and Low-Memory Tuning

- **MGLRU enabled by default**, including dirty/writeback handling, folio isolation, scan-batch tuning,
  and refault-path improvements.
- **UKSM enabled by default**, using the Android CPU governor together with rmap-walk, resource-release,
  and exceptional-path fixes.
- **Kcompressd-Unofficial**: asynchronously offloads part of kswapd swapout compression to a dedicated
  kernel thread.
- **le9uo working-set protection**: uses anonymous-page and clean-file-page watermarks to reduce working-set
  thrashing under memory pressure.
- Supports memcg-aware swap allocation and anonymous-only proactive reclaim.
- Includes latency and stability fixes across the page allocator, vmalloc, swap-fault, THP, and folio-reclaim paths.

### ⚡ CPU Scheduling and Power

- **Built-in HMBIRD scheduling class**, including cgroup deadlines, SLIM/WALT utilization tracking,
  shadow ticks, and runtime control interfaces.
- HMBIRD DSQ, remote-task pulling, timeout, irq_work, and fork failure-path fixes.
- Optimized EAS energy calculation, schedutil/iowait boost, idle load balancing, and overutilized detection.
- `SM_IDLE` scheduling and idle fast paths to reduce some idle re-entry overhead.
- Improved menu and TEO cpuidle decisions, including protection against powering down a CPU with pending IPIs.
- Power-efficient workqueues, scheduler clusters, and Android wakelock configuration tuning.
- **Boeffla Wakelock Blocker** with a configurable wakelock blocking interface.

### 💽 I/O and Filesystems

- **BFQ I/O scheduler** with cgroup support built into the kernel.
- F2FS compression, ATGC, and GC_MERGE; background GC runs with idle scheduling/I/O priority and includes
  multiple GC, unmount, and compression-writeback fixes.
- EROFS per-CPU kthread support and retry handling for failed LZ4 bounce-page allocation.
- zsmalloc compaction, zram memory tracking, and backing-device statistics provided by the Crystal data plane.
- 🛠️ TWRP / Recovery can be entered normally.

### 🌐 Networking

- **BBRv3** TCP congestion control, configured as the default TCP congestion control.
- **FQ-CoDel** as the default network queueing discipline, with FQ also available.
- TCP ECN negotiation enabled by default, along with mobile-latency-oriented TCP path adjustments.
- Android GKI networking features including netfilter, IP sets, XDP sockets, and multiple routing tables.

### 📱 SM8750 and OPLUS Adaptation

- **Device-tree overwriter** modifies or creates device-tree properties during kernel startup according to
  project-specific configuration; the current tree includes OnePlus 13T (`24821`) charging and monitoring settings.
- Configurations for OnePlus 13 (`23821`), OnePlus Ace 5 Pro (`24811`), and some realme projects are also
  included, but should be treated as unvalidated adaptation material.
- **Module overlay framework** can intercept module loads and replace userspace-provided modules with
  zstd-compressed versions embedded in the kernel.
- **Droidspace support**: the current GKI configuration supports running Droidspace (DroidSpaces).
- Device-specific fixes cover Qualcomm SCM, charging protocols, and OPLUS vendor compatibility paths.

## 🙏 Acknowledgements

This kernel integrates and adapts work from the following developers and projects, in no particular order:

| Developer | Contribution |
|:---|:---|
| [**brokestar233**](https://github.com/brokestar233) | Original Device-tree overwriter and Module overlay framework, plus SM8750 / OPLUS device adaptations and kernel optimizations |
| [**firelzrd**](https://github.com/firelzrd) | le9u/o and Kcompressd-Unofficial |
| [**CachyOS**](https://github.com/CachyOS/linux/commits/6.16/bbr3) | BBRv3 congestion control |
| [**sroeschus**](https://github.com/sroeschus/uksm) | UKSM, adapted from its 6.6 patch set |
| [**epicmann24**](https://github.com/epicmann24) | Boeffla Wakelock Blocker |

Thanks also to the Linux kernel community, Android Common Kernel, OPLUS/OnePlus platform developers,
CrystalFrostwork, and all contributors. Exact authorship, source, and `Signed-off-by` information is
available in the Git history and source file headers.
