# 6. GPU 监控模块集成（NVIDIA/AMD/Intel）

目标：解释 btop 如何与各厂商的 GPU 监控接口对接（NVML、ROCm-SMI、Intel DRM/IOCTL），以及为什么某些监控需要更高权限。

相关文件：`src/linux/btop_collect.cpp`, `src/linux/intel_gpu_top/` 目录（若存在），`CMakeLists.txt` 中的 `GPU_SUPPORT` 选项

一、NVIDIA（NVML）
- NVML（NVIDIA Management Library）提供 C API 用于查询 GPU 利用率、温度、功耗等。
- btop 在编译/运行时加载并调用 nvml 库（动态链接），并解析返回数据到内部 `Gpu::gpu_info` 结构。

二、AMD（ROCm-SMI）
- ROCm-SMI 提供类似接口（rocm_smi_lib），需要在系统上安装对应库与驱动。

三、Intel GPU
- Intel GPU 的统计信息通常通过 DRM 接口或平台特定工具（如 `intel_gpu_top` 的相关实现）获取。部分信息需要对 `/dev/dri/` 等设备文件有读取权限，因此可能需要 root 或特权。

四、权限与访问控制
- 许多 GPU 统计需要读取内核接口或特殊设备节点；这些在默认用户下可能受限。btop 的 `make setcap` / `make setuid` 目标用于设置合适的 capabilities 或 suid，以允许非 root 用户读取所需信息。

五、实现细节与优化
- 并行化：对多 GPU 实例，btop 会并行采集每块 GPU 的信息以缩短总体延迟（见 `linux/btop_collect.cpp` 中 per-GPU 线程分配）。
- 动态库加载：通过dlopen/dlsym在运行时选择可用后端，避免静态依赖导致的兼容性问题。

六、学习价值与实验
- 本模块适合学习如何使用第三方厂商 SDK（NVML/ROCm）与动态加载，以及设备权限与安全边界如何影响可观测性。
