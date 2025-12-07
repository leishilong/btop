# 5. 非阻塞键盘输入与特殊键解析

目标：解释 btop 如何把终端设置为 raw 模式、如何做非阻塞读取，并解析方向键与功能键等 ANSI 序列。

相关文件：`src/btop_input.cpp`, `src/btop.hpp`

一、设置终端为 raw 模式
- raw 模式要做的主要事：关闭 canonical 模式、关闭回显、关闭信号生成（如 Ctrl+C 的内核处理）等。通常调用 `tcgetattr` / `tcsetattr`，并保留原始设置以便恢复。

二、非阻塞读取与超时
- 两种常见方法：
  - 把文件描述符设置为非阻塞（`fcntl`），使用 `read()` 轮询（或使用 `usleep`）、或
  - 使用 `select()` / `poll()` / `ppoll()` 在带超时的等待上阻塞，然后 `read()` 读取可用字节（推荐）。
- btop 使用输入轮询与 `atomic` 标志来避免阻塞主线程（见 `Input::poll()` 与 `Input::interrupt()`）。

三、特殊键的 ANSI 序列解析
- 方向键（箭头）通常是 `ESC [ A/B/C/D`。
- 功能键与组合键：例如 `F1-F4`、`Home/End` 有不同的序列（`ESC O P` 或 `ESC [ 1 ~` 等），不同终端可能有差异。
- 解析策略：
  - 读取首个字节；若是 ESC（0x1b），继续在短时间窗口内读取后续字节，合并为一个序列再匹配已知模式。
  - 使用状态机或查表匹配常见序列，并对未知序列做容错（按字节回放或忽略）。

四、组合键支持（Ctrl/Alt/Shift）
- Ctrl 与字母通常是单字节（例如 Ctrl+C = 0x03）。Alt 常作为前缀 `ESC + key` 表示。

五、实验与验证
- 在 `btop_input.cpp` 增加打印解析到的控制码（Logger::debug），在不同终端模拟器（iTerm2、Terminal、Alacritty）下测试按键序列差异。

学习价值总结：掌握 raw 模式、select/poll 的超时读取、以及可移植的 ANSI 键序列解析策略，是构建交互式 CLI 的重要技能。
