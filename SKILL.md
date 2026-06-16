---
name: dl-inference
description: 通用深度学习推理项目指导。用于设计、实现、审查或重构模型推理与服务代码，包括命令行推理、批量或流式推理、模型加载、device/dtype 处理、gRPC 服务、runtime/runner 抽象和验证基准。
---

# 深度学习推理

用于通用深度学习推理系统。将项目特定模型逻辑隔离在 loader、inferencer、runtime 和 runner 抽象之后。

## 架构

推荐顶层结构：

```text
__init__.py  commands  configs  inferencers  model_runners
models       servicer  proto    utils
```

- `commands`：CLI 入口和用户参数解析。
- `configs`：runtime/model 配置。
- `models`：模型定义、适配器和注册。
- `inferencers`：任务级推理流程、预处理、后处理、流式状态。
- `model_runners`：eager、CUDA Graph、compile、TensorRT 或平台相关执行策略。
- `servicer` 和 `proto`：gRPC 服务边界与协议代码。
- `utils`：通用 I/O、数据变换、日志和小型工具函数。

## CLI 与服务约定

统一暴露常用参数：`--model-path`、`--device`、`--dtype`、batch/stream chunk 设置，以及 `--host`、`--port`、worker/并发限制、最大消息大小、timeout/keepalive 等服务参数。字符串解析放在 `commands` 或其邻近工具中；推理代码只接收框架原生类型。

`dtype` 名称使用 `fp32`、`bf16`、`fp16` 等形式；构造模型前检查 device 兼容性。gRPC 请求解析、字节解码和响应组装不要进入核心 model runtime。

## 工作流

分离模型加载、推理编排、runtime 执行和加速 runner。接口应尽量窄，使 CPU、CUDA、NPU、eager、graph、compiled 路径能复用同一个 inferencer。

使用对齐检查、并发测试和速度指标验证，例如输入时长除以计算耗时。Python 包与 CLI 约定联动 `python-poetry`；`.proto` 生成联动 `protos`；训练侧问题使用 `dl-train`。
