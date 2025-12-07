# 3. 多线程架构与线程间数据同步

目标：详细解读 btop 的并发模型、线程间如何通信与同步，以及为何使用原子/轻量同步的设计。指出可能的竞态风险与调试策略。

相关文件：`src/btop.cpp`, `src/btop_shared.hpp`, `src/btop_input.cpp`, `src/linux/btop_collect.cpp`（若启用 GPU）

一、线程模型概览
- btop 使用混合模型：
  - 主线程：初始化、信号处理、UI 主循环（和输入）
  - Runner（二级线程）：负责数据采集与（部分）绘制逻辑。通过 `pthread_create` 创建（见 `src/btop.cpp`）。
  - 平台/功能线程：例如 Linux 下 GPU/PCIE 使用 `std::thread` 和 `std::async` 来并行化设备采集。

二、主要同步原语
- `std::atomic<bool>`：用于标志通知（例如 `Runner::active`, `Global::resized`, `Global::thread_exception`），这种“标志+忙等待/短等待”模式成本低且被广泛使用。
- `std::mutex` / `std::lock_guard`：用于保护非线程安全的系统调用（例如 macOS 的 IOKit），或保护短期共享资源。
- `pthread_join` / `pthread_cancel`：用于线程生命周期管理；注意 `pthread_cancel` 可能需要小心资源释放。

三、为什么偏向“只读共享 + 原子更新”
- 监控工具的关键是低延迟与低开销：频繁使用 heavy-weight mutex 会影响性能。
- 通过把读操作多为无锁读取、写者在短时间内更新（并使用 atomic 或替换整个结构），可以减少锁冲突。

四、风险点（可能的竞态）
- 未对大型共享容器（如 process list）进行严格锁保护，若有并发写入可能出现未定义行为；需要审查哪些容器在采集时被 write，并确保主线程读取前写入已完成。
- `pthread_cancel` 取消线程时资源清理不完整可能导致内存泄漏或不一致状态。

五、调试与验证技巧
- 使用 ThreadSanitizer（TSAN）构建并运行测试以检测竞态：在 CMake 中添加 `-fsanitize=thread`（需 clang 支持）。
- 在关键同步点（atomic wait/trigger）插入 `Logger::debug` 时戳，观察延迟与交互顺序。

六、实战建议
- 若要在不牺牲正确性的前提下进一步优化并发：
  - 把共享结构改为 copy-on-write：后台线程构建完整数据结构后用 atomic 指针交换，主线程仅读取指针。
  - 对长时间运行的任务使用 `std::future`/`std::async` 并在超时后跳过结果合并，以避免阻塞主采集路径。

学习价值总结：掌握用原子信号减少锁带来的延迟、如何分层并行化采集任务，并学会用 TSAN 等工具验证并发正确性。
