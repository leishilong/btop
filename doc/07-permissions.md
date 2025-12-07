# 7. 权限模型与提权机制

目标：说明 btop 为何在某些信息采集需要更高权限、`make setcap`/`setuid` 的作用，以及沙箱（Snap/Flatpak）对运行的影响。

相关文件：`Makefile`, `CMakeLists.txt`（可能含 `setcap`、`setuid` 目标）

一、为什么需要 sudo 或 setcap？
- 原因在于某些内核接口或设备节点对普通用户不可读写（或需要 CAP_SYS_PTRACE、CAP_DAC_READ_SEARCH 等 capability）。例如：
  - 读取其他用户进程的详细信息或发送信号可能需要额外权限。
  - 访问 `/dev/dri/*` 或某些 sysfs 条目可能被权限限制。

二、`make setcap` 与 `make setuid` 的区别
- `setcap`：赋予二进制进程特定的 Linux capabilities（例如 `cap_sys_ptrace`），不需要整个进程以 root 身份运行，安全性更高。
- `setuid`：把二进制的所有者设置为 root 并设置 setuid 位，运行时以 root 权限执行（更危险，需谨慎）。

三、沙箱影响（Snap/Flatpak）
- 沙箱环境对文件系统与设备节点访问有严格限制，可能会阻止 btop 读取某些 /proc 或 /sys 文件。需要相应的接口声明或权限才能正常工作。

四、安全建议
- 优先推荐 `setcap` 指定最小能力而不是 `setuid root`。并在文档中清晰提示风险。
