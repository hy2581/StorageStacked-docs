# llm/result：输出文件说明

这里是输出说明索引，不复制各次历史仿真的日志、二进制文件和大体积波形。
原项目运行时创建 `result/运行目录/`；以下文件名均相对于其中一次运行目录。
根目录构建还会使用 `result/build/`，回归测试可能再按测试名分层。
文件是否存在取决于已完成的阶段；失败或较早格式的运行可能只有其中一部分。

## 最短阅读顺序

1. 打开 report.md 看运行结论和输出位置。
2. 检查 summary.json 的完整状态，再看 completion.json 的设备退出原因。
3. 查看 llm_summary.json 的生成 token、模型数值比较和 KV/中间结果检查。
4. 需要追踪访存时，从 transactions.csv 查看地址，再关联 AXI、UCIe 与 MEMSIM 记录。

单个日志出现成功字样、程序退出码为 0、或 completion.json 通过，都不能替代最后的完整校验。

## 共有文件

| 文件或目录 | 看什么 |
| --- | --- |
| `llm_summary.json` | 实际生成 token、模型输入、独立数值核对与推理检查汇总。 |
| `input.json` | 本次用户输入的固定快照；先确认自己实际跑了哪个配置。 |
| `resolved.json` | 运行所需的 architecture、程序路径、模型信息等解析结果。 |
| `environment.json` | 本次运行使用的环境与工具信息。 |
| `build/` | 用户程序、生成头文件及运行器选择记录；具体文件由平台编译流程决定。 |
| `build.log` | 平台检查、用户编译及解析阶段日志；编译失败先看这里。 |
| `run.log` | 设备/主机程序输出、仿真器信息和异常；运行失败先看这里。 |
| `validation.log`、`verification.log` | 结果与链路校验过程日志；数值或协议失败时查看。 |
| `completion.json` | 计算是否正常结束、退出原因和时刻；只是完整验收的一部分。 |
| `summary.json`、`report.md` | 运行入口汇总的完整状态及面向读者的报告。 |
| `config.json`、`ucie_config.json`、`memsim_config.json` | 仿真器、编译链路、存储模型的实际配置记录；与用户输入快照互相核对。 |
| `transactions.csv` | 桥接事务从接收、AXI 完成到上游响应完成的生命周期，包含地址、字节数和分段。 |
| `axi_events.csv`、`axi256_handshakes.csv` | AXI 五通道事件与握手明细，用于核对 VALID/READY 和返回。 |
| `axi_first_200ns_cycles.csv`、`axi_flit_path.csv` | 前段周期视图、AXI 到 flit 路径的关联记录。 |
| `axi_wave.vcd`、`axi_view.gtkw`、`axi_overview.svg` | 信号波形、GTKWave 查看配置和总览图。 |
| `ucie_flits.csv`、`ucie_soc.csv`、`ucie_mem.csv`、`flit_pairs.csv` | 链路 flit 与两端收发、对应关系。 |
| `aou_events.csv`、`aou_messages.csv`、`aou_summary.json` | AXI over UCIe 消息、事件与汇总。 |
| `aou_check_summary.json`、`link_check_summary.json`、`check_summary.json`、`protocol_summary.json` | 不同检查器的协议与链路结果，需结合最终 summary 阅读。 |
| `memsim_bridge.csv`、`memsim_bridge_summary.json`、`memsim_check.json` | 公共后端与存储模型之间的请求/响应及检查。 |
| `memsim_commands.csv`、`memsim_core.json`、`memsim_stats.txt` | 存储命令、核心配置/状态与统计。 |
| `memsim_dfi.csv`、`memsim_dfi_signals.csv`、`memsim_image.csv`、`memsim_journeys.json` | DFI 交互、实际内存镜像与请求轨迹，用于追踪数据去向。 |
| `memsim_view.html`、`memsim_data/` | 存储过程的可视化页面与配套数据。 |
| `trace_view.html`、`trace_flits_data/`、`trace_paths_data/`、`view_store.js` | 链路浏览页面和分片数据；阅读时需要保留配套目录。 |
| `wave_audit/` | 波形审计的中间结果与检查记录。 |

## CoralNPU 的补充文件

| 文件或目录 | 看什么 |
| --- | --- |
| `npu_requests.csv` | NPU 原生请求与响应，含序号、原生 ID、AXI ID、数据和掩码。 |
| `build/program.elf` | 由 CoralNPU 加载的 RV32 程序。 |
| `build/simulator.path` | 运行入口选中的 coralnpu_sim 路径。 |

## 本例重点看什么

先确认 input.json 中 prompt 保留了末尾空格、generated_tokens 和 kv_cache 符合本次选择。
再核对返回权重、KV、中间 trace、生成 token 和 forward 计数。
权重、KV、trace、token、report 的设备地址从 0x90000000 起分区，具体见对应计算源码的解释版。
GPU 版本的设备地址还会经过 BAR 映射，关联 AXI 地址时要使用转换后的地址。

```mermaid
flowchart LR
  A["输入快照"] --> B["构建与运行日志"] --> C["计算完成记录"] --> D["返回数据 + 协议检查"] --> E["summary.json / report.md"]
```

## 用本次运行目录实际读一次

假设刚从原项目 user 目录运行：

```bash
./run.sh llm --output result/reading-01
cat llm/result/reading-01/input.json
cat llm/result/reading-01/report.md
cat llm/result/reading-01/summary.json
```

先看 input，防止误读另一份配置或旧结果。report 适合阅读，
summary 给出最终结论。运行失败时，文件可能尚未生成齐，先看失败阶段再决定打开哪个日志。

## 看一项数据时，记录三个信息

1. **字段在哪个文件？** 同名 passed 在 completion 与 summary 中覆盖的范围不同。
2. **单位是什么？** tick_fs 是 fs，周期数要配合该设备的频率。
3. **统计的是哪一层？** 源码操作、上游事务、AXI burst、Flit 和内存子请求数不同。

LLM 先读输入 token、生成 token、forward_calls、各阶段数值比较。
默认 `red ` 生成 `blu`；开关 KV 的对照应得到相同文本，
但也必须核对中间值和链路。token 一样不自动证明所有计算正确。

## 怎样沿原始记录找一次访问

CoralNPU 从 npu_requests.csv 的 sequence、wire_id 和地址开始。
transactions.csv 的 substream 保存原生 sequence，id 是传输用 AXI ID。

再去 axi_events.csv 找对应地址通道与 R/B 返回，
需要看停顿时，打开同一时间区间的 axi_wave.vcd。
用 UCIe 与 mem_sim 的关联记录继续向后追，不要仅用两份 CSV 的行号相等来配对。

请求 ID 在完成后可能复用，因此关联时同时使用方向、时间范围和上游序号。
数据以字节串、十进制整条总线值或浮点数显示时，先统一地址与字节顺序再比较。

## 如果目录里只有部分文件

| 最后完成的阶段 | 可能已有 | 下一步 |
|---|---|---|
| 配置快照 | input.json、初始报告 | 看终端和 build.log |
| 编译 | build 中的程序 | 看运行阶段是否启动、run.log 是否报错 |
| 设备结束 | completion 和原始日志 | 等待或排查独立校验 |
| 完整校验 | summary、任务摘要与报告 | 检查最终 passed 及各项检查 |

可视化网页依赖同目录下的数据分片和 JS；复制分享时保留它们的相对位置。
结果里出现的历史计数、延迟属于那一份配置和程序，不能直接套到另一份实验。

本页是结果读法，不是本轮新仿真的通过报告。
