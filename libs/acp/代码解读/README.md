# deepagents-acp 源码解读

> 本文档为 `libs/acp` 包（`deepagents-acp` v0.1.8）的完整技术解读，目的是帮助你理解该系统的架构与实现，作为构建类似系统的参考。

## 目录结构

```
代码解读/
├── README.md                    ← 本文件：总览与导读
├── 01-架构总览.md               ← 系统架构、分层设计、核心概念
├── 02-核心模块/
│   ├── server.md                ← DeepAgentsServer 核心服务器
│   ├── adapter.md               ← 消息格式适配器（ACP ↔ LangChain）
│   ├── acp-filesystem-backend.md← ACP 文件系统后端
│   ├── logger.md                ← 日志系统
│   └── cli.md                   ← 命令行入口
├── 03-数据结构与类型.md          ← 所有关键 TypeScript 类型定义解析
├── 04-流程分析/
│   ├── 初始化握手流程.md         ← initialize 协议握手
│   ├── 会话生命周期.md           ← session/new → prompt → cancel
│   ├── 工具调用流程.md           ← Tool call 从触发到回调全流程
│   └── 会话持久化与恢复.md       ← LangGraph Checkpointer 机制
└── 05-扩展开发指南.md            ← 如何在此基础上构建自己的系统
```

## 快速导读

| 你想了解的问题                           | 看哪里                                                               |
| ---------------------------------------- | -------------------------------------------------------------------- |
| 这个包是做什么的？                       | [架构总览](./01-架构总览.md)                                         |
| ACP 协议是什么？                         | [架构总览 → ACP 协议概述](./01-架构总览.md#acp-协议概述)             |
| 服务器如何处理用户输入？                 | [server.md](./02-核心模块/server.md)                                 |
| ACP 消息格式如何转换为 LangChain？       | [adapter.md](./02-核心模块/adapter.md)                               |
| 文件读写如何通过 IDE 代理？              | [acp-filesystem-backend.md](./02-核心模块/acp-filesystem-backend.md) |
| SessionState / ToolCallInfo 等类型定义？ | [数据结构与类型](./03-数据结构与类型.md)                             |
| 一次对话从头到尾的完整流程？             | [会话生命周期](./04-流程分析/会话生命周期.md)                        |
| 工具调用（Tool call）怎么工作的？        | [工具调用流程](./04-流程分析/工具调用流程.md)                        |
| 我想构建一个类似的系统，怎么设计？       | [扩展开发指南](./05-扩展开发指南.md)                                 |

## 源码文件一览

| 文件                            | 行数  | 职责                          |
| ------------------------------- | ----- | ----------------------------- |
| `src/server.ts`                 | ~1419 | 核心服务器，所有 ACP 协议处理 |
| `src/adapter.ts`                | ~257  | ACP ↔ LangChain 消息格式互转  |
| `src/types.ts`                  | ~359  | 全部 TypeScript 类型定义      |
| `src/acp-filesystem-backend.ts` | ~95   | 文件操作代理给 IDE 客户端     |
| `src/logger.ts`                 | ~301  | 双通道日志（stderr + 文件）   |
| `src/cli.ts`                    | ~332  | CLI 入口，参数解析            |
| `src/index.ts`                  | ~79   | 公共 API 导出入口             |
