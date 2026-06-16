---
name: dl-inference
description: 通用深度学习推理项目指导。用于设计、实现、审查或重构模型推理与服务代码，包括命令行推理、批量或流式推理、模型加载、device/dtype 处理、gRPC 服务、runtime/runner 抽象和验证基准。
---

# 深度学习推理

用于通用深度学习推理系统。保持模型加载、任务推理、runtime 执行、服务协议和硬件加速策略彼此分离；不要把项目特定模型逻辑、协议解析或平台优化混进同一层。

## 架构

优先采用或收敛到这类顶层结构：

```text
__init__.py  commands  configs  inferencers  model_runners
models       servicer  proto    utils
```

- `commands`：CLI 入口、环境变量、字符串参数解析、日志初始化和服务启动。
- `configs`：可复用的 model/runtime/serving 配置；避免把临时 CLI 字符串向内传播。
- `models`：模型定义、适配器、注册和权重加载兼容层。
- `inferencers`：任务级推理编排，包括预处理、后处理、batch/stream 状态和结果对齐。
- `model_runners`：执行策略，如 eager、CUDA Graph、`torch.compile`、TensorRT、ONNXRuntime、NPU runner。
- `servicer` 与 `proto`：gRPC 协议边界、请求校验、响应封装和生成代码。
- `utils`：通用 I/O、数据变换、dtype/device 解析、日志和小型纯工具。

## CLI 与服务约定

统一暴露常用推理参数：`--model-path`、`--device`、`--dtype`、`--batch-size`、stream chunk 大小、最大输入长度、并发限制和随机种子。服务参数使用 `--host`、`--port`、`--workers`、`--max-message-mb`、timeout/keepalive 等清晰命名。

字符串解析放在 `commands` 或邻近 CLI utils；进入 inferencer/runtime 后只传框架原生类型。`dtype` 名称优先用 `fp32`、`bf16`、`fp16`、`int8`，构造模型前检查 device 与 dtype 兼容性。gRPC 的字节解码、schema 校验、错误映射和响应组装留在 `servicer`，不要进入核心 model runtime。

## Runner 与 Runtime

runtime 负责模型实例、device/dtype 放置、锁、缓存转移和基础 forward。runner 只表达某种执行策略，接口要窄，通常只接收已准备好的张量和缓存，并返回结果或显式降级。

不要让 CUDA Graph、编译、TensorRT 或平台特定逻辑污染 inferencer。加速路径必须有 eager 回退或明确失败模式，并记录是否启用、是否降级、降级原因和关键 shape/dtype/device。

## 验证

至少覆盖 eager 与 runner 的数值/事件对齐、离线与流式对齐、多路并发、长输入、空输入、非法 dtype/device、模型路径不存在和请求超限。性能报告使用输入时长除以计算耗时，同时关注 P50/P95 延迟、吞吐、显存和不同 dtype 矩阵。

Python 包与 CLI 约定联动 `python-poetry`；`.proto` 生成联动 `protos`；训练、导出和 checkpoint 侧问题使用 `dl-train`。
