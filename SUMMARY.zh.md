# Agent Status Signal 规范草案 · 中文摘要

**一句话**：给刘海、菜单栏宠物、锁屏、手表、车机、桌面硬件这类「环境化界面」定一个统一的 agent 状态格式，让它们不用再给每个 agent 单独写适配。

**为什么要做**
- coucou、Codogotchi 等项目各自去解析 Claude Code hooks、自定义 socket 格式、Paseo 的 WebSocket，重复造轮子。
- 现有协议各管一段：ACP 是一个客户端对一个会话（相关问题见 #554 回合结束、#2285 失败上报、#1979 批准丢上下文、#2444 子会话）；A2A 的任务状态面向 agent 之间；OTel GenAI 面向事后观测，连「agent 不出声了」都表达不了（#4187）。

**核心内容**
1. 7 个状态：idle、working、needs-approval、needs-input、stuck、done、failed，附转换规则。每条事件都带完整状态，漏了消息也能自己恢复；done 和 failed 会一直保留，直到被确认。
2. 一种事件格式 `agent.status`（附 JSON Schema）。要批准时必须带上具体要批准的动作；用 parentId 汇总子 agent；结束时带一句人话结果。可选的回复格式是 `agent.respond`，必须校验 requestId。
3. 传输：默认用本地 socket，没人监听时不阻塞；已有 daemon 的用 WebSocket；手机、车机、硬件由本地桥转发（0.1 版不覆盖）。
4. 映射：ACP、A2A、Claude Code hooks、Paseo 到 7 个状态的对照表（2026-10-27 已对照各项目源码/文档核实，附链接）。
5. 打扰策略（建议性的，借鉴座舱经验）：默认只显示汇总；卡住检测（3 分钟没进展、同一操作重复 3 次、连续重试）；打扰预算（需要人的事只提醒一次，完成的事只在长任务时提醒，专注、开会、开车时攒着等人回来再说）。

**开放问题**：放进 ACP 还是独立成规范；谁来判断 stuck；多个设备之间怎么同步「已经批准」；回复操作的安全要求；状态命名。

**文件**：[SPEC.md](SPEC.md)（英文全文）、[schema/status-0.1.json](schema/status-0.1.json)
