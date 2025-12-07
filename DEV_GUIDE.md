# btop 开发阅读指南（DEV_GUIDE）

本文档用于引导你系统化阅读并理解 `btop` 项目，重点在于：架构、性能优化点、多线程实现以及如何高效学习本项目的优秀代码结构。

## 一、总体目标（你将学到什么）
- 快速定位程序主循环、采集（collect）、绘制（draw）与输入处理的代码位置。
- 理解多线程模型、同步方法与可能的危险点。
- 识别项目中为性能做的优化，以及如何复现/验证这些优化效果。
- 学会如何在本地构建、调试与做轻量级性能分析。

## 二、推荐的阅读顺序（按先后）
1. `README.md` — 项目目的、构建流程与平台说明（快速浏览）。
2. `src/btop_shared.hpp` — 全局变量和 `Runner`、共享结构的声明（了解全局边界）。
3. `src/btop.cpp` — 程序入口、信号处理、线程创建与 `Runner` 主体（主循环和线程管理）。
4. `src/btop_input.cpp` — 输入轮询和中断机制（如何响应键盘/终端事件）。
5. `src/btop_draw.cpp` — 绘制逻辑（如何绘制 boxes、减少重绘）。
6. `src/*/btop_collect.cpp`（以平台为单位：`linux/`, `osx/`, `freebsd/` 等）— 平台特定的数据采集实现。重点查看 `linux/btop_collect.cpp`（含 GPU/PCIE 并行逻辑）。
7. `src/btop_tools.cpp` / `src/btop_tools.hpp` — 工具函数与通用帮助（字符串处理、时间、格式化、辅助原子等）。
8. `tests/` — 看项目如何写测试（`ctest` 配置与单测用例）。

建议：按此顺序阅读主线代码，遇到不理解的点再回溯到相关的头文件或实现文件。

## 三、带着这些问题去阅读（逐条回答会帮你理解设计意图）
- 主循环是什么样子？数据采集与绘制的分工如何？（查看 `Runner`）
- 程序如何避免不必要的重绘？哪些标志控制重绘频率？（查找 `force_redraw`, `background_update`, `resized`）
- 数据如何缓存与保留历史？为何使用 `deque`/`unordered_map`？
- 多线程有哪些线程？每个线程职责是什么？线程如何启动/停止？（`pthread_create`/`pthread_join`）
- 共享数据如何同步？哪里使用 `std::atomic`，哪里使用 `std::mutex`？为什么？
- 哪些系统调用可能成为瓶颈（例如 IOKit、sysctl、/proc 读写、GPU 库调用）？
- 编译选项或宏如何影响功能与性能（例如 `GPU_SUPPORT`、`STATIC`）？

每读完一个模块（如 `btop_collect`），尝试在代码里标注出：关键数据结构、性能相关逻辑、同步边界（谁写谁读谁锁）。

## 四、性能优化点与如何验证
- 已观察到的优化策略：
  - 将采集（IO）与绘制分离；尽量减少 UI 重绘。
  - 使用 `std::atomic` 作轻量状态同步，减少互斥开销。
  - 并行采集（`std::thread` / `std::async`）用于独立 GPU 或耗时子任务。
  - 对平台 API（如 IOKit）序列化调用（局部 mutex）以避免数据竞态或 API 不线程安全带来的崩溃。

验证方法：
1. 在本地用 `cmake -B build -G Ninja && cmake --build build` 编译（或用已有的 VS Code tasks）。
2. 用 profiler：macOS 用 `Instruments (Time Profiler)`、或 `dtrace`；Linux 用 `perf`。观察 `collect` vs `draw` 的 CPU 比例。
3. 用 `strace` / `dtruss` / `dtrace` 观察系统调用频率，检查是否存在频繁的 syscalls。
4. 加日志埋点：在 Runner 的 collect/draw 开始/结束处打印耗时（`Logger::debug`），运行一段时间，收集统计。
5. 在不同采样间隔（`--updates` 或 `update_ms`）下测试响应性与 CPU 使用。

## 五、多线程实现要点（代码定位与关注点）
- Runner（次线程）通过 `pthread_create` 启动：`src/btop.cpp`（搜索 `Runner::_runner`、`pthread_create` 行）。
- 线程间通信主要用 `std::atomic<bool>` 的标志（如 `Runner::active`, `Global::resized`），以及少量 `std::mutex` 在平台特定调用处（`src/osx/btop_collect.cpp` 的 `iokit_mutex`）。
- Linux GPU 子任务使用 `std::thread` 与 `std::async`（见 `src/linux/btop_collect.cpp`）。

关注点与建议实验：
1. 查看 `pthread_cancel` 的使用场景（是否安全），以及 `pthread_join` 的 timeout 行为。
2. 在 `Runner` 中临时记录 `atomic_wait_for` 的等待时长，检验是否有长时间阻塞。
3. 运行 ThreadSanitizer（TSAN）构建来检测竞态（见下节步骤）。

## 六、实战检查清单（Checklist）
1. 在本地成功构建并运行 `build/btop`。`./build/btop`。  
2. 运行测试：`ctest --test-dir build --output-on-failure`。  
3. 用 `Logger::debug` 插桩 collect/draw，运行并采集 30s 的日志，分析耗时分布。  
4. 用 profiler 定位 CPU 热点（Instruments / perf）。  
5. 用 TSAN 检查竞态：在 CMake 配置中添加 `-fsanitize=thread`（注意: TSAN 需要 clang 支持并可能影响运行）。  

示例命令（macOS / bash/zsh）：
```bash
# 配置 + 构建
mkdir -p build && cmake -B build -G Ninja && cmake --build build -- -j 0

# 运行测试
ctest --test-dir build --output-on-failure

# 用 Instruments（手动）或 perf（Linux）分析
```

## 七、关于学习策略：从最初版本看差异，还是先看最新版本？
我的建议是“先看最新实现（把能跑、能理解的业务线搞通），再回溯历史去看重要变更的 diff”——原因：

- 先看最新版的好处：
  - 代码是可运行且经过多次修正的“稳定形态”，能让你更快建立整体的心智模型（模块边界、数据流、API 使用）。
  - 阅读最新代码能直接在本地构建、调试、插桩，马上验证你的理解。

- 再回溯历史的好处：
  - 通过查看关键提交（例如引入 macOS 支持的 v1.1.0、GPU 支持引入、重大重构），你能理解为什么做出那些设计决策、有哪些替代实现被放弃、哪些兼容性问题曾存在。这样可以学到作者处理兼容性、性能和可维护性的思路。 
  - 特别有价值的是查看引入多线程、并发控制、或大型重构的 commit message 与 diff（这些处往往包含设计权衡的讨论）。

推荐步骤：
1. 先从 `main`（最新）克隆、构建并运行，跟随本指南按模块顺序阅读并在代码中加注释/笔记。  
2. 确认你理解了关键模块后，找历史上的“里程碑提交”（例如 README 中提到的 v1.1.0、v1.2.0、v1.3.0）或 `git log --grep` 搜索关键词（`macOS`, `GPU`, `runner`, `thread`），阅读这些提交的 diff 和讨论。  
3. 对照不同版本的实现，思考并记录：为什么要改变？性能/可维护性/兼容性 是否是驱动因素？  

这样可以兼顾“快速上手（最新）”与“深入理解设计演化（历史）”。

## 八、我可以帮你做的后续工作（可选）
- A1：把本指南添加到仓库（已在进行）并在 `README.md` 中加入链接。  
- A2：为每个关键函数/模块生成跳转链接（VS Code workspace quick links）或 TODO 注释，便于你逐步检视。  
- B：在 `Runner` 中加入临时计时埋点并运行一轮采样，返回汇总耗时。  
- C：配置一个 TSAN/ASAN 的 CMake 构建并运行测试，检测竞态/内存问题。  

---

如果你同意，我会把这份 `DEV_GUIDE.md` 提交并推送（我会完成该步骤）。之后我可以根据你想要的深入方向（例如插桩采样或 TSAN）继续执行。 

Happy hacking! 欢迎选下一步（比如让我添加跳转注释、插桩并运行一次采样，或配置 TSAN）。
