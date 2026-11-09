---
name: clamav-maintenance
description: Use only when the user explicitly requests enabling or using ClamAV for a specified security-maintenance task.
---

# ClamAV 按需安全维护

## 启用前提

只有用户明确要求启用或使用 ClamAV，才使用本技能。普通维护、提到“安全”或编辑本技能不构成启用。没有对应指令时停止，不探测程序、不联网更新病毒库、不扫描。

## 执行范围

- 先明确本次用途：安装、病毒库更新、扫描指定文件或目录、查看结果。只执行用户已要求的动作。
- 使用当前平台的官方安装方式，先核对现有安装；不把某台机器的路径、版本或已启用状态带到另一台。
- 扫描范围、病毒库更新的联网需要、结果保存位置及后续处置按任务核实。默认只报告，不自动删除、隔离或上传样本。
- 不默认启动 clamd、freshclam 后台进程，不建立计划任务或开机启动。持续运行需要用户单独明确要求。
- 结果保存在本机 local/ 或用户指定的私有目录，不公开原始路径、文件内容或扫描结果。
- 完成后记录实际执行、退出状态、病毒库日期、扫描范围、未处理项和服务启用状态；单次使用不代表永久启用。

ClamAV 为独立第三方工具，许可沿用 GPLv2；本仓库不捆绑程序或病毒库。官方文档：https://docs.clamav.net/ 。
