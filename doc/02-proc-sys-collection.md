# 2. 从 /proc 和 /sys 采集系统指标（CPU/内存/网络/磁盘）

目标：分析 btop 如何从 Linux 的 `/proc` 与 `/sys` 中采集实时系统指标，并解释关键算法（例如 CPU 两次采样、内存字段意义、网络速率计算）。

相关文件：`src/linux/btop_collect.cpp`, `src/cpu.cpp`（或 `src/*/cpu.cpp`）、`src/mem.cpp`, `src/net.cpp`, `src/disk.cpp`

一、CPU 使用率计算（两次采样的原因）
- /proc/stat 中每个 CPU 的累计时间字段是自系统启动以来的累积计数（user, nice, system, idle, iowait, irq, softirq, steal, guest 等）。
- 要计算百分比，必须在两个时间点采样，求差值：
  - delta_total = total2 - total1
  - delta_idle = idle2 - idle1
  - cpu_usage = (delta_total - delta_idle) / delta_total
- 这是因为采样间隔内的累积值增长代表该间隔内的实际消耗。

二、内存字段（如何区分 MemTotal/Available/Buffers/Cached）
- `/proc/meminfo` 常见字段：MemTotal, MemFree, MemAvailable, Buffers, Cached, SwapTotal, SwapFree 等。
- `MemAvailable` 是内核尝试估算的“应用可用内存”，比 `MemFree` 更有意义。btop 应使用 `MemAvailable` 作为可用内存基准（若存在）；否则用 MemFree + Buffers + Cached 的近似。

三、网络速率计算（/proc/net/dev）
- `/proc/net/dev` 每行包含接口累计的 rx_bytes、tx_bytes 等值。和 CPU 类似，要计算速率需两次采样：
  - delta_rx = rx2 - rx1; rate_rx = delta_rx / interval_seconds
- 注意：处理采样回绕（counter rollover）和接口 UP/DOWN（重置 counters）的情况。

四、磁盘 IO（/proc/diskstats 或 sysfs）
- `/proc/diskstats` 提供设备累计读写扇区、时间等信息。通过差分采样计算 IOPS、吞吐量。

五、实现注意点与优化
- 减少读取开销：批量读取并解析（一次 open/read），而非对每个字段单独系统调用。
- 使用 `std::string::reserve` 和手工 parse（如 `strtol`）来避免频繁内存分配。btop 在解析 /proc 时常用较轻量的解析函数（见 `btop_tools.cpp`）。
- 缓存静态信息（例如 CPU 核心数、网卡列表、磁盘列表），避免每次采样遍历 sysfs。

六、验证建议
- 在 `btop_collect.cpp` 增加采样日志（前后值）以验证差分计算是否正确。模拟短间隔高频次采样，验证是否出现负值或回绕处理问题。

学习价值总结：通过实现和验证差分采样逻辑，能深入理解 Linux 导出的监控接口与如何在用户空间高效地计算实时指标。
