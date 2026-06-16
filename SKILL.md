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

项目根目录可放置 `scripts/`，用于封装引擎的本地冒烟测试、基准入口和一次性维护脚本；不要把它当作 `src/<package>/` 内部模块。

## CLI 与服务约定

CLI 参数、函数签名、README 示例和部署环境变量表按固定顺序组织：模型来源、runtime 推理参数、gRPC 服务参数、日志参数。模型来源放第一位且通常必填；runtime 参数包括 `--device`、`--dtype`、`--runner`、`--batch-size`、`--seed`、stream chunk 大小和最大输入长度；服务参数包括 `--host`、`--port`、`--workers`、`--max-message-mb`、timeout/keepalive；日志参数放最后。

环境变量名称保持一致并按分组排列：`MODEL_PATH`；`DEVICE`、`DTYPE`、`RUNNER`、`BATCH_SIZE`、`SEED`、`MAX_INPUT_LENGTH`、`CHUNK_SIZE`；`GRPC_HOST`、`GRPC_PORT`、`GRPC_WORKERS`、`GRPC_MAX_MESSAGE_MB`、`GRPC_TIMEOUT_MS`、`GRPC_KEEPALIVE_MS`、`GRPC_KEEPALIVE_TIMEOUT_MS`；`LOG_LEVEL`、`LOG_FORMAT`。

默认值只给部署上安全的参数：`MODEL_PATH` 不设默认值；runtime 默认保守，如 `DEVICE=cpu`、`DTYPE=fp32`、`RUNNER=eager`、`BATCH_SIZE=1`；gRPC 默认可用容器部署，如 `GRPC_HOST=0.0.0.0`、`GRPC_PORT=50051`、`GRPC_WORKERS=10`、`GRPC_MAX_MESSAGE_MB=100`；日志默认 `LOG_LEVEL=INFO`。不要默认猜测或下载模型，不要默认使用 CUDA 或加速 runner，不要对非法 dtype/device/runner 静默回退。

外部控制字符串必须在边界层解析：CLI/env 字符串放在 `commands` 或邻近 CLI utils；进入 inferencer/runtime 后只传 `torch.device`、`torch.dtype`、内部枚举、dataclass、NumPy array 或明确的领域对象。文本内容、资源路径和业务标签这类数据字符串可以保留，但 dtype/device/runner/log-level/单位值等控制字符串不要向底层传播。`dtype` 名称优先用 `fp32`、`bf16`、`fp16`、`int8`，构造模型前检查 device 与 dtype 兼容性。

## gRPC 与边界分层

按 `proto request -> servicer/service -> task adapter/inferencer -> runtime/runner -> model` 分层。数据跨层时应逐步变普通：protobuf 对象先在 servicer 中解包成 Python 原生数据，再由任务 adapter/inferencer 转成 NumPy 数据或领域对象；runtime/runner 再把 NumPy 输入转换为 Torch tensor 并放置到目标 device/dtype；结果按相反方向从 Python 结果对象封装回 protobuf response。

`servicer` 是协议适配层，只接触 `pb2`、`pb2_grpc` 和 `grpc.ServicerContext`。它负责 protobuf 字段校验、enum/Duration/repeated/bytes/path 等协议字段解包、请求级错误映射和响应组装；不要在 servicer 中混入模型特征构造、tensor 放置或 runtime/runner 策略。inferencer、runtime、runner、model 不接触 protobuf 或 gRPC 类型。

任务 adapter/inferencer 接收 Python 原生输入，负责把 bytes/list/dict/path 等数据适配为 NumPy array 或领域对象，并执行任务级预处理、batch/stream 编排和后处理；inferencer 的公开输入边界不要要求调用方传 Torch tensor。runtime 管模型实例、device/dtype、权重加载、锁、缓存和 NumPy 到 Torch 的转换；runner 只表达 eager、CUDA Graph、compile、TensorRT 等执行策略，通常只接收已准备好的 Torch tensor 和缓存；model 保持纯模型定义和 forward。

gRPC server 创建时将 `GRPC_MAX_MESSAGE_MB` 转成 bytes，并同时设置 `grpc.max_receive_message_length` 与 `grpc.max_send_message_length`。请求校验按必填字段、枚举和数值范围、payload 大小和空输入、格式对齐、解码适配、推理、输出封装的顺序执行。已知错误映射到明确状态码，如 `INVALID_ARGUMENT`、`NOT_FOUND`、`RESOURCE_EXHAUSTED`、`FAILED_PRECONDITION`、`DEADLINE_EXCEEDED`、`UNAVAILABLE`；未知异常记录 `exc_info=True` 后映射为 `INTERNAL`，不要让所有异常裸露成 `UNKNOWN`。

## Runner 与 Runtime

runtime 负责模型实例、device/dtype 放置、锁、缓存转移和基础 forward。runner 只表达某种执行策略，接口要窄，通常只接收已准备好的张量和缓存，并返回结果或显式降级。

不要让 CUDA Graph、编译、TensorRT 或平台特定逻辑污染 inferencer。加速路径必须有 eager 回退或明确失败模式，并记录是否启用、是否降级、降级原因和关键 shape/dtype/device。

## 验证

项目根目录 `scripts/` 应提供封装引擎的冒烟测试脚本，用最小真实或合成输入完成模型加载、runtime/runner 构造和一次端到端推理，并打印输入摘要、输出摘要、device/dtype/runner、耗时和关键 shape；脚本失败时返回非零退出码。冒烟脚本面向本地开发、镜像构建和部署前验证，不替代单元测试或完整基准。

至少覆盖 eager 与 runner 的数值/事件对齐、离线与流式对齐、多路并发、长输入、空输入、非法 dtype/device、模型路径不存在和请求超限。性能报告使用输入时长除以计算耗时，同时关注 P50/P95 延迟、吞吐、显存和不同 dtype 矩阵。

Python 包与 CLI 约定联动 `python-poetry`；`.proto` 生成联动 `protos`；训练、导出和 checkpoint 侧问题使用 `dl-train`。
