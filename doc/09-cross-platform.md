# 9. 跨平台支持（Linux/macOS/FreeBSD）实现要点

目标：说明 btop 如何通过条件编译和平台抽象支持多平台，并列出平台差异与实现细节。

相关文件：`src/osx/`, `src/linux/`, `src/freebsd/`, `CMakeLists.txt`

一、实现策略
- 使用 `#ifdef __linux__`, `#ifdef __APPLE__`, `#ifdef __FreeBSD__` 等宏，在平台子目录下实现 platform-specific 的 `btop_collect.cpp`，主逻辑通过统一接口调用。

二、平台差异示例
- macOS：使用 IOKit/SMC 查询温度与电池信息（见 `src/osx/smc.cpp`、`sensors.cpp`），并需注意 API 的线程安全（使用 mutex 序列化）。
- FreeBSD：使用 kvm、sysctl 等接口，文件和字段名与 Linux 不同。

三、可移植性注意点
- 字符宽度、终端行为、文件路径与权限模型在不同系统上可能不同，需在测试矩阵中覆盖。
