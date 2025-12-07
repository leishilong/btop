# 10. 错误处理与日志系统

目标：解析 btop 的日志/错误处理模式，如何在调试/生产环境下使用 `--debug` 打印更多信息来定位问题。

相关文件：`src/btop_tools.cpp`（Logger 实现）、`src/btop.cpp`（异常处理点）

一、日志系统能力
- Logger 提供多级日志（DEBUG/INFO/WARNING/ERROR），并写入到 log 文件（位置由 Config 决定）。
- 在关键异常点程序会设置 `Global::exit_error_msg` 并通过 `clean_quit()` 做统一退出处理，确保终端被恢复。

二、异常与错误策略
- 大多数平台 API 调用错误通过返回码检查与异常捕获（`try/catch`）来处理；在不可恢复时设置退出信息并优雅退出。

三、调试建议
- 使用 `--debug` 或设置 Logger 级别开启更多输出；在源码中临时增加 `Logger::debug` 埋点以收集时序信息。
