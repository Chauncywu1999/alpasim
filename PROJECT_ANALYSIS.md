# AlpaSim 项目分析与总览

> 本文面向首次接触 AlpaSim 的研究者与工程师，基于仓库内 README、设计文档与各模块说明，梳理架构、模块职责、运行流程、数据产物、配置与运维要点，并给出快速上手与扩展建议。

---

## 项目概览

AlpaSim 是一个面向自动驾驶研究的、模块化的仿真平台。它以“传感器拟真、水平扩展、研究可hack性”为核心目标，采用 Python 微服务 + gRPC 的方式，将传感器模拟、车辆控制与动力学、物理约束、驾驶策略、评估与日志等环节解耦，支持单机与多机的弹性部署。

- 传感器拟真：接入神经渲染（NRE）生成高保真相机帧，支持传感器噪声与环境条件配置。
- 研究友好：Python 实现 + gRPC 接口，便于替换/插拔自研组件。
- 横向扩展：多微服务架构，支持在多 GPU/多结点扩展各组件的副本和并发能力。

核心文档：
- 设计概览：[docs/DESIGN.md](docs/DESIGN.md)
- 入门教程：[docs/TUTORIAL.md](docs/TUTORIAL.md)
- Onboarding（依赖与环境）：[docs/ONBOARDING.md](docs/ONBOARDING.md)
- 运维与性能调优：[docs/OPERATIONS.md](docs/OPERATIONS.md)
- 数据管线与 ASL 日志：[docs/DATA_PIPELINE.md](docs/DATA_PIPELINE.md)
- 顶层说明：[README.md](README.md)

---

## 架构总览

AlpaSim 以 `runtime` 为中心节点，通过 gRPC 与各微服务交互，形成闭环：

1. `runtime` 维护世界状态与事件时钟，协调服务调用并记录日志。
2. `sensorsim`（外部 NRE 服务）依据状态生成相机帧。
3. `driver`（驾驶策略）根据传感器与导航输入输出 `ego` 轨迹/控制意图。
4. `controller`（车辆模型 + 控制器）将轨迹/意图转为实际车辆执行并返回状态。
5. `physics`（地面约束）对 `ego` 与非 `ego` 行为施加地面/法向约束。
6. `runtime` 汇总状态、产出 `asl` 日志，循环推进仿真；`eval` 离线消费日志生成指标与视频。

架构图与详细说明见：[docs/DESIGN.md](docs/DESIGN.md)

---

## 关键模块与职责

- [src/runtime](src/runtime/README.md)：
  - 仿真主循环、事件时钟与零延迟（Zero-delay）模式校验。
  - gRPC 客户端，统一与各微服务交互，并产出 `asl`/评估输入。
- [src/controller](src/controller/README.md)：
  - `SystemManager`/`System` 管理车辆状态与 MPC 控制（基于 `do_mpc`/`casadi`）。
  - 接收规划轨迹与状态，计算转向与加速度，推进车辆动力学。
- [src/physics](src/physics/README.md)：
  - 基于环境地面网格施加地面约束（不做碰撞与完整动力学）。
- [src/driver](src/driver/README.md)：
  - 驾驶策略服务（VaVAM 为默认；支持 Alpamayo-R1 与 Transfuser 示例）。
- [src/grpc](src/grpc/README.md)：
  - gRPC 接口与 protobuf 定义；需先在该目录编译 proto 以便 Python 使用。
- [src/eval](src/eval/README.md)：
  - 评估指标（Scorers）与视频生成；消费 `asl` 日志聚合出安全/性能指标。
- [src/wizard](src/wizard)：
  - 基于 Hydra 的向导工具，生成配置、拉起微服务（本仓库主要提供 OSS 本地部署配置）。
- [tools](src/tools) / [utils](src/utils)：
  - 通用工具与脚本（地图相关、运行在特定集群上的脚本等）。

---

## 运行流程与数据产物

执行一次仿真（例如本地 `docker compose`）后，在 `wizard.log_dir` 下会产生：
- `asl/`：每个 rollout 的二进制日志（size-delimited protobuf），记录会话元数据、Actor 位姿、跨服务请求/返回等。
- `eval/`：单次 rollout 的原始指标（`metrics_unprocessed.parquet`）与视频。
- `aggregate/`：跨所有 rollout 的聚合指标（文本与图表）与按违规类型组织的视频。
- `metrics/`：性能画像（Prometheus 导出 + 自动生成的 `metrics_plot.png`）。
- `txt-logs/`：各服务的文本日志用于排障。
- 若干已解析/派生的配置快照（如 `wizard-config.yaml`、`generated-*.yaml`）。

详见教程的“结果结构”与运维指南：[docs/TUTORIAL.md](docs/TUTORIAL.md)、[docs/OPERATIONS.md](docs/OPERATIONS.md)

ASL 日志格式与读取示例详见：[docs/DATA_PIPELINE.md](docs/DATA_PIPELINE.md) 与 [src/utils/alpasim_utils/logs.py](src/utils/alpasim_utils/logs.py)

---

## 开发与运行环境

必要依赖（简）：
- 需具备 Hugging Face 读取权限（环境变量 `HF_TOKEN`），用于下载场景/模型资产。
- `uv`（Python 环境/工具链管理）、Docker + Compose、CUDA 12.6+、NVIDIA Container Toolkit。
- 详见：[docs/ONBOARDING.md](docs/ONBOARDING.md)

### Proto 编译
在 [src/grpc](src/grpc/README.md) 目录：
```bash
uv run compile-protos
```
（如修改定义可重复该命令；清理使用 `uv run clean-protos`）

---

## 快速上手（本地）

1) 环境准备（参考 Onboarding）。
2) 初始化本地环境：
```bash
source setup_local_env.sh
```
3) 使用 Wizard 生成配置并运行（默认 VaVAM 策略）：
```bash
uv run alpasim_wizard +deploy=local \
  wizard.log_dir=$PWD/tutorial
```
4) 结果查看：`tutorial/` 下的 `asl/`、`eval/`、`aggregate/`、`metrics/` 等。

更多用法示例（选择场景、替换策略、视频布局等）参见：[docs/TUTORIAL.md](docs/TUTORIAL.md)

---

## 配置与扩展

- Wizard 配置基于 Hydra；参考示例：[src/wizard/configs](src/wizard/configs)
- 运行时关键参数（对齐各“时钟”）：`control_timestep_us`、`egopose_interval_us`、相机 `frame_interval_us`、`time_start_offset_us`。零延迟模式下需保持数学整齐（严格对齐），`assert_zero_decision_delay=true` 可在运行时做一致性校验。详见：[src/runtime/README.md](src/runtime/README.md)
- 驾驶策略：默认 VaVAM，可切换至 Alpamayo-R1/Transfuser 或自定义镜像（需暴露兼容的 gRPC 接口），见：[docs/TUTORIAL.md](docs/TUTORIAL.md)、[docs/OPERATIONS.md](docs/OPERATIONS.md)
- 代码改动：仓库源码会以挂载方式注入容器，新增依赖需重建镜像；见教程“Code changes”。

示例：将推理频率改为 5Hz（200ms）并对齐相机与姿态更新：
```bash
uv run alpasim_wizard +deploy=local_oss \
  wizard.log_dir=runs/{DATETIME} \
  runtime.default_scenario_parameters.control_timestep_us=200000 \
  runtime.default_scenario_parameters.egopose_interval_us=200000 \
  runtime.default_scenario_parameters.time_start_offset_us=600000 \
  runtime.default_scenario_parameters.cameras.0.frame_interval_us=200000
```
更多频率组合与校验说明见：[docs/OPERATIONS.md](docs/OPERATIONS.md)

---

## 性能与运维要点

- 副本与 GPU 分配：在 `src/wizard/configs/deploy/local_oss.yaml` 配置 `replicas_per_container` 与 `gpus`，总能力约等于 `nr_gpus * replicas_per_container * n_concurrent_rollouts`。各服务需容量匹配，避免瓶颈。详见：[docs/OPERATIONS.md](docs/OPERATIONS.md)
- 自动性能画像：每次运行生成 `metrics/metrics_plot.png`，包含 RPC、时序、CPU/GPU 利用率与内存等。利用队列深度、RPC 时长与 GPU 利用率判断瓶颈。
- 排障：优先查看控制台与各服务 `txt-logs/`；若 `asl/` 未生成，按时间顺序定位首次错误的服务。

---

## 测试与质量

- 单元测试分散在各模块 `tests/`。示例：控制器模块测试运行方式见：[src/controller/README.md](src/controller/README.md)
- 修改 gRPC 定义或协议相关逻辑后，需重新编译 proto 并（如有）更新对应客户端/服务端实现。

---

## 目录结构要点（摘）

- 数据与场景：
  - 资产下载脚本与说明：[data/README.md](data/README.md)，场景清单：[data/scenes/sim_scenes.csv](data/scenes/sim_scenes.csv)、套件：[data/scenes/sim_suites.csv](data/scenes/sim_suites.csv)
- 运行时 Notebook（日志回放）：[src/runtime/notebooks/replay_logs_alpamodel.ipynb](src/runtime/notebooks/replay_logs_alpamodel.ipynb)
- gRPC 接口与示例：见 [src/grpc](src/grpc/README.md) 以及 `alpasim_grpc/v0/*`

---

## 文档索引（常用）

- 总览与设计：[README.md](README.md)、[docs/DESIGN.md](docs/DESIGN.md)
- 教程与入门：[docs/TUTORIAL.md](docs/TUTORIAL.md)、[docs/ONBOARDING.md](docs/ONBOARDING.md)
- 运维与性能：[docs/OPERATIONS.md](docs/OPERATIONS.md)
- 数据与日志：[docs/DATA_PIPELINE.md](docs/DATA_PIPELINE.md)
- 运行时与零延迟：[src/runtime/README.md](src/runtime/README.md)
- 控制器与 MPC：[src/controller/README.md](src/controller/README.md)
- 评估与视频生成：[src/eval/README.md](src/eval/README.md)
- gRPC API：[src/grpc/README.md](src/grpc/README.md)

---

## 建议的下一步

- 使用 Wizard 跑一次默认场景，核验环境与资产下载是否顺利，查看 `aggregate/metrics_results.*` 与视频。
- 根据研究目标调整频率、相机与场景套件，并在 `metrics_plot.png` 中识别瓶颈后做容量匹配与扩展。
- 如需替换策略或调试：参考教程 Level 3 的“断点调试”，单独拉起某服务并使用生成的 Compose 配置与端口对齐。

如需我进一步补充模块间调用时序图、接口字段表或生成更细的代码索引，请告诉我你的重点方向。