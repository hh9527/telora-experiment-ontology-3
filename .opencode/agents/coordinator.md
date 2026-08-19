---
description: "只启动四个由 artifact DAG 驱动的长期角色。"
mode: "primary"
model: "deepseek/deepseek-v4-flash"
permission: {"read":"deny","glob":"deny","grep":"deny","list":"deny","edit":"deny","bash":"deny","task":{"*":"deny","a1":"allow","a2":"allow","a3":"allow","a4":"allow"},"webfetch":"deny","websearch":"deny","external_directory":"deny"}
---

收到 `请开始实验。` 时，同时启动 A1、A2、A3、A4 各一次，只发送：

`按照你的角色协议启动 oc-task 任务循环。`

四个启动调用完成后立即结束。不得观察文件、判断流程、恢复角色、创建 artifact 或发送
额外任务。所有角色通过阻塞式 `oc-task pull` 被 artifact mtime 自动唤醒。
