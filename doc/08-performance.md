# 8. 性能优化手段解析

目标：列出 btop 在源码层面可观察到的性能优化手段，并给出验证 / 改进建议。

相关文件：`src/btop_tools.cpp`, `src/btop_collect.cpp`, `src/btop_draw.cpp`

一、已观察到的优化点
- 减少系统调用：合并输出到单一 buffer，减少 write() 次数。
- 使用原子与轻量同步而非粗粒度锁。
- 并行化耗时采集（GPU/PCIE）以降低总体采样延迟。
- 缓存静态信息（核数、磁盘/网卡列表）以免重复扫描 sysfs。

二、具体实现技巧
- IO 解析优化：使用单次 `read` + 手写解析函数（比流式解析更快），避免大量临时字符串分配。
- 字符串缓冲：使用 `std::string::reserve` 或静态 buffer，避免频繁 reallocation。

三、可能的进一步优化建议
- 使用 `mmap` 读取较大 /proc 文件（比如 /proc/kpageflags 或其他需要大量解析的文件）以减少拷贝。
- 对热点函数内联（适度）、减少异常/虚函数开销。

四、验证手段
- 用 profiler（Instruments/perf）定位热点；对热点函数做 micro-benchmark。
