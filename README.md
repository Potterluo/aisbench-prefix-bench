# AISBench Prefix Cache 性能测试工具

本工具用于测试 **vLLM prefix cache** 的性能和命中率，基于 [AISBench](https://github.com/AISBench/benchmark) 框架运行。

主要测量指标：

- **吞吐量**：整体 token/s、请求/s
- **延迟**：TTFT（首 Token 延迟）、TPOT（每 Token 生成延迟）
- **Prefix Cache 命中率**：HBM（片上缓存）和 External（外部缓存），per-DP + 汇总
- **并发稳定性**：多 DP 域、多并发下的命中率与吞吐

---

# 方式一：本地运行（AISBench 环境）

适合：已有 AISBench 安装环境的机器，直接 `python3 prefix_bench.py` 运行。

## 1. 下载代码

```bash
git clone https://github.com/Potterluo/aisbench-prefix-bench.git
cd aisbench-prefix-bench
```

> 也可以在 GitHub 页面点击 `Code → Download ZIP` 下载解压。

## 2. 修改服务配置

打开项目根目录下的 `config.py`，修改以下 **必改** 项：

```python
# ===== Service configuration =====
HOST_IP = "10.0.0.1"                            # 【必改】推理服务 IP
HOST_PORT = 8000                                 # 【必改】推理服务端口
URL = ""                                         # Docker 场景填完整 URL
MODEL_NAME = ""                                  # 空 = AISBench 自动检测
MODEL_PATH = "/home/weights/model_weights"        # 【必改】tokenizer/模型权重路径

# ===== AISBench configuration =====
WORK_PATH = "/benchmark"                         # AISBench 安装目录（容器内固定路径为 /benchmark）
```

| 参数 | 填什么 | 示例 |
| --- | --- | --- |
| `HOST_IP` | 推理服务 IP（需为合法 IPv4/IPv6） | `10.0.0.1` |
| `HOST_PORT` | 推理服务端口 | `8000` |
| `MODEL_PATH` | tokenizer 所在目录路径 | `/mnt/model/Qwen3-32B` |
| `WORK_PATH` | AISBench 安装目录 | `/benchmark` |

> **tokenizer 怎么找？** 就是模型目录下包含 `tokenizer.json` / `tokenizer_config.json` 的那一层目录，通常就是模型权重所在目录。

## 3. 修改测试参数

`config.py` 中还有测试参数，按需修改：

```python
INPUT_LEN = 3500          # 输入 token 长度
OUTPUT_LEN = 1500         # 输出 token 长度
DATA_NUM = 8192           # 数据集条数
CONCURRENCY = 2048        # 全量测试并发数
REQUEST_RATE = 0          # 请求频率（0 = 突发）
REPEAT_RATE = 0.5         # 前缀重复率（0.5 或 "50%"）
PREFIX_NUM = 1            # 前缀种类数
DP = 1                    # 数据并行度
SEED = 1                  # 随机种子（0 = 纯随机）
```

**示例：测一组 input=8192 / concurrency=2048 / repeat_rate=0.5 / dp=2**

```python
INPUT_LEN = 8192
CONCURRENCY = 2048
REPEAT_RATE = 0.5
DP = 2
SEED = 1
```

**示例：测多组不同配置**（使用 `--rounds` 参数，见 [多轮测试](#多轮测试) 章节）

## 4. 运行测试

```bash
python3 prefix_bench.py
```

或指定部分参数覆盖 `config.py`：

```bash
python3 prefix_bench.py \
    --host_ip 10.0.0.1 --host_port 8004 \
    --input_len 8192 --concurrency 2048 \
    --repeat_rate 0.5 --dp 2 --seed 1
```

## 5. 查看结果

结果默认保存在项目根目录下：

| 文件 | 内容 |
| --- | --- |
| `prefix_bench_result.csv` | 所有测试记录，方便 Excel 打开 |
| `prefix_bench_result.jsonl` | 同内容 JSONL 格式，方便程序化读取 |

各字段含义见文末 [测试结果字段说明](#测试结果字段说明)。

---

# 方式二：Docker 镜像运行

适合：需要在容器内运行 AISBench 的场景（如 K8s Pod、Docker Desktop 等）。镜像里**不含**代码和模型，需要通过挂载引入。

## 1. 准备 AISBench 镜像

已有 AISBench Docker 镜像即可，无需额外操作。如需构建：

```bash
# 参考 AISBench 官方文档获取镜像
```

## 2. 下载代码

```bash
git clone https://github.com/Potterluo/aisbench-prefix-bench.git
cd aisbench-prefix-bench
```

## 3. 理解挂载路径与配置的关系

Docker 运行时，容器看不到宿主机文件，必须用 `-v 宿主机路径:容器路径` 把目录"映射"进去。配置文件里填的路径必须是**容器内能看到的路径**，不是宿主机路径。

假设宿主机上：

- 代码在：`/home/user/aisbench-prefix-bench`
- 模型在：`/mnt/model/Qwen3-32B`

> AISBench 容器内路径为 `/benchmark`（内置），无需额外挂载。

| 挂载什么 | 命令片段 | 容器内对应路径 | 作用 |
| --- | --- | --- | --- |
| 代码目录 | `-v /home/user/aisbench-prefix-bench:/workspace` | `/workspace` | 读代码、写结果 |
| 模型目录 | `-v /mnt/model:/mnt/model` | `/mnt/model` | 读 tokenizer |

> **关键技巧**：模型目录挂载时，让容器内路径和宿主机路径保持一致（都写 `/mnt/model`），这样 `config.py` 里 `MODEL_PATH` 直接写 `/mnt/model/Qwen3-32B` 就行，不用换算路径。

## 4. 修改配置文件

在**宿主机的代码目录**里改 `config.py`（改完容器会直接读到，因为是挂载的）：

```python
HOST_IP = "localhost"                      # 填 localhost（满足校验）
HOST_PORT = 8000
URL = "http://host.docker.internal:8080"   # 【必改】Docker 场景填完整 URL
MODEL_PATH = "/mnt/model/Qwen3-32B"        # 容器内路径
WORK_PATH = "/benchmark"                   # AISBench 容器内固定路径
```

> **Docker 网络说明**：下方运行命令用了 `--network=host`，容器直接共享宿主机网络。如果你的 Docker Desktop / WSL2 不支持 `--network=host`，需改用 `--add-host=host.docker.internal:host-gateway` 并把 `URL` 填 `http://host.docker.internal:8080`。
>
> **为什么需要 `--url`？** AISBench 校验 `host_ip` 必须为合法 IPv4/IPv6 地址，Docker 中 `host.docker.internal` 不合法。`--url` 非空时 AISBench 直接使用 URL 连接服务，忽略 host_ip/host_port，但 host_ip 仍需合法以满足校验。

## 5. 运行测试

```bash
docker run --rm \
  --network=host \
  --add-host=host.docker.internal:host-gateway \
  -v /home/user/aisbench-prefix-bench:/workspace \
  -v /mnt/model:/mnt/model \
  -w /workspace \
  aisbench_image \
  python3 prefix_bench.py
```

逐行解释：

| 命令片段 | 含义 |
| --- | --- |
| `--rm` | 运行完自动删除容器 |
| `--network=host` | 容器直接用宿主机网络 |
| `--add-host=host.docker.internal:host-gateway` | Docker Desktop 场景访问宿主机 |
| `-v ...:/workspace` | 宿主机代码 → 容器工作目录 |
| `-v ...:/mnt/model` | 宿主机模型 → 容器内 tokenizer |
| `-w /workspace` | 容器内工作目录 |
| `python3 prefix_bench.py` | 执行测试（AISBench 容器内需用 python3） |

## 6. 查看结果

结果写在容器内 `/workspace/` 下，因为代码目录是挂载的，**宿主机代码目录里直接就能看到**：

| 文件 | 内容 |
| --- | --- |
| `prefix_bench_result.csv` | 所有测试记录 CSV |
| `prefix_bench_result.jsonl` | 同内容 JSONL |

---

# 配置优先级

**CLI 命令行参数 > config.py 配置文件 > 硬编码默认值**

- 手动执行时：编辑 `config.py` 填写常用值（IP、端口、模型路径等），命令行只需传变化的参数
- CI 集成时：全部通过命令行参数传递，忽略 `config.py`
- 多轮测试时：`--rounds` 中每轮的 override 字段优先级高于 CLI 基值

例如 `config.py` 中 `HOST_IP="10.0.0.1"`，运行时 `--host_ip 10.0.0.2` 则使用 `10.0.0.2`。

---

# 参数详解

`python3 prefix_bench.py --help` 查看所有参数。

## 服务配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--host_ip` | str | localhost | 目标服务 IP（需为合法 IPv4/IPv6） |
| `--host_port` | int | 8000 | 目标服务端口 |
| `--url` | str | "" | 完整 URL，覆盖 host_ip/host_port（Docker 场景必填） |
| `--model_name` | str | "" | 模型名称（空 = AISBench 自动检测） |
| `--model_path` | str | **必填** | tokenizer/模型权重路径 |
| `--npu_num` | int | 1 | NPU 卡数（用于单卡吞吐计算） |

## 测试配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--test_name` | str | "" | 可选名称标签（写入结果 CSV） |
| `--input_len` | int | 3500 | 输入 token 长度 |
| `--output_len` | int | 1500 | 输出 token 长度 |
| `--data_num` | int | 8192 | 数据集条数 |
| `--concurrency` | int | 2048 | 全量测试阶段最大并发 |
| `--request_rate` | int | 0 | 请求频率（0 = 突发） |
| `--test_type` | str | stream | API 类型：stream 或 text |
| `--enable_think` | flag | False | DeepSeek V3.1 思考模式 |

## 前缀配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--prefix_num` | int | 1 | 前缀种类数 |
| `--repeat_rate` | float | 0.5 | 前缀重复率（0.5 或 "50%"） |
| `--dp` | int | 1 | 数据并行度（影响 warmup 并发数） |
| `--seed` | int | 1 | 随机种子；0 = 纯随机（不可复现） |

## 变长配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--length_mean` | int | None | 高斯均值 |
| `--length_std` | float | None | 高斯标准差 |
| `--length_min` | int | None | 最小长度 |
| `--length_max` | int | None | 最大长度 |

> `length_mean` 和 `length_std` 必须同时提供或同时省略；`length_min` 和 `length_max` 同理。

## AISBench 配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--work_path` | str | **必填** | AISBench 安装目录 |
| `--dataset_path` | str | /home/dataset | 数据集存储目录（容器内可挂载） |
| `--output_dir` | str | ./outputs/default | AISBench 输出目录 |
| `--summarizer` | str | default_perf | AISBench summarizer |

## 结果与命中率

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--result_csv` | str | prefix_bench_result.csv | 结果 CSV 路径 |
| `--result_jsonl` | str | prefix_bench_result.jsonl | 结果 JSONL 路径 |
| `--pod_info` | str[] | None | Pod 地址（ip:port），默认用 host_ip:host_port |

## 数据集覆盖

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `--dataset` | str | None | 使用已有数据集文件，跳过生成 |

---

# 多轮测试

使用 `--rounds` 参数可以在一次运行中测试多组不同配置，每组独立执行完整的两阶段测试（warmup → full），结果追加到同一个 CSV。

## 格式说明

`--rounds` 接受 JSON 数组，每项是一个 dict 覆盖本轮配置：

**可覆盖字段**（缺省字段继承 CLI/config.py 默认值）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `input_len` | int | 输入 token 长度 |
| `output_len` | int | 输出 token 长度 |
| `data_num` | int | 数据集条数 |
| `concurrency` | int | 全量测试并发数 |
| `request_rate` | int | 请求频率 |
| `prefix_num` | int | 前缀种类数 |
| `repeat_rate` | float/str | 前缀重复率（0.5 或 "50%"） |
| `seed` | int | 随机种子（0 = 纯随机） |
| `test_name` | str | 可选名称标签 |

**不可覆盖字段**（全局共享）：`host_ip`、`host_port`、`url`、`model_name`、`model_path`、`work_path` 等。

## 示例

### 内联 JSON

```bash
python3 prefix_bench.py \
    --model_path /mnt/model/Qwen3-32B \
    --work_path /benchmark \
    --host_ip 10.0.0.1 --host_port 8004 \
    --rounds '[{"input_len":3500,"output_len":1500,"concurrency":2048,"seed":1},
               {"input_len":8500,"output_len":1500,"concurrency":2048,"seed":2},
               {"input_len":8192,"concurrency":4096,"seed":0}]'
```

### JSON 文件

将配置写入 `rounds_config.json`：

```json
[
  {
    "input_len": 3500,
    "output_len": 1500,
    "data_num": 8192,
    "concurrency": 2048,
    "request_rate": 0,
    "prefix_num": 1,
    "repeat_rate": 0.5,
    "seed": 1
  },
  {
    "test_name": "long_input",
    "input_len": 8500,
    "output_len": 1500,
    "concurrency": 4096,
    "repeat_rate": "50%",
    "seed": 2
  }
]
```

```bash
python3 prefix_bench.py \
    --model_path /mnt/model/Qwen3-32B \
    --work_path /benchmark \
    --host_ip 10.0.0.1 --host_port 8004 \
    --rounds rounds_config.json
```

### 传统 shell 循环（仍然有效）

```bash
for cc in 8 16 32; do
    python3 prefix_bench.py \
        --model_path /mnt/model/Qwen3-32B \
        --work_path /benchmark \
        --host_ip 10.0.0.1 --host_port 8004 \
        --concurrency $cc --data_num $(($cc * 4)) \
        --repeat_rate 0.5 --dp 2 --seed $cc \
        --result_csv prefix_bench_result.csv
done
```

---

# 测试流程说明

工具采用**两阶段前缀测试**：

| 阶段 | 并发数 | 输出长度 | 目的 |
| --- | --- | --- | --- |
| Phase 1（warmup） | = dp | 1 | 用 dp 并发发送仅前缀的请求，预热 KV Cache |
| Phase 2（full） | = concurrency | = output_len | 全量并发发送完整数据集请求，测量吞吐和命中率 |

> **为什么先 warmup？** prefix cache 只有在 KV Cache 已有前缀内容时才能命中。warmup 阶段将前缀注入缓存，full 阶段才能测到真实的命中率。

---

# 数据集生成逻辑

完全基于 tokenizer 词表的随机采样，不依赖 GSM8K：

- **前缀**：调用 `get_some_tokens(prefix_len, seed=seed+i)` 随机采样
- **分隔符**：每行 3 个随机 token，`get_some_tokens(3, seed=seed+prefix_num+i)`
- **后缀**：`get_some_tokens(suffix_len, seed=seed+prefix_num+data_num+i)`
- **拼接**：`prefix + separator + suffix`

**关键特性**：

- ✅ 不依赖 GSM8K.jsonl，纯随机生成
- ✅ 不使用 picked_ids.txt，无全局状态累积
- ✅ Seed 完全控制所有文本生成，多轮测试间无 prefix collision
- ✅ 使用 local RNG（`random.Random(seed)`），不污染全局 random 状态
- ✅ `seed=0` 时使用系统熵（纯随机，不可复现）

---

# 命中率说明

命中率数据来自 vLLM `/metrics` 端点，区分两种缓存：

| 类型 | 指标前缀 | 说明 |
| --- | --- | --- |
| **HBM（片上缓存）** | `vllm:prefix_cache_*` | GPU 显存中的 KV Cache 命中 |
| **External（外部缓存）** | `external_prefix_cache_*` | 外部存储（如 UCM 磁盘）中的命中 |

计算公式：

```
hit_rate = (hits_after - hits_before) / (queries_after - queries_before)
```

Per-DP 数据按 `engine_id` 分组，汇总为所有 DP 的总和。

---

# PD 分离场景配置

PD 分离场景中，`--pod_info` 需要配置对应节点的 IP 和 DP 域端口：

- **单开 prefix cache**：配置 P 节点 IP + 每个 DP 域的 port
- **开启池化**：配置每个节点的 IP + 每个 DP 域的 port

```bash
python3 prefix_bench.py \
    --model_path /mnt/model/Qwen3-32B \
    --work_path /benchmark \
    --host_ip 10.0.0.1 --host_port 8004 \
    --dp 2 --repeat_rate 0.5 \
    --pod_info "141.xx.xx.11:8000" "141.xx.xx.12:8000"
```

> 当 `--pod_info` 未指定时，自动从 `--url` 中提取 host:port（Docker 场景），或使用 `--host_ip:host_port` 作为默认值。

---

# 测试结果字段说明

| 字段 | 含义 |
| --- | --- |
| `timestamp` | 测试执行时间 |
| `round` | 多轮测试中的轮次编号（单轮 = 1） |
| `test_name` | 可选名称标签 |
| `phase` | 测试阶段（`warmup` 或 `full`） |
| `url` | 推理服务地址 |
| `input_len` | 输入 token 长度 |
| `output_len` | 输出 token 长度 |
| `concurrency` | 并发数 |
| `data_num` | 数据集条数 |
| `dp` | 数据并行度 |
| `repeat_rate` | 前缀重复率 |
| `seed` | 随机种子 |
| `total_requests` | 总请求数 |
| `ttft_avg_ms` | 首 Token 平均延迟（ms） |
| `tpot_avg_ms` | 每 Token 平均生成延迟（ms） |
| `throughput` | 整体吞吐（token/s） |
| `hbm_hit_rate_dp0` | DP0 HBM 命中率 |
| `hbm_queries_dp0` | DP0 HBM 查询数 |
| `hbm_hits_dp0` | DP0 HBM 命中数 |
| `hbm_hit_rate_total` | 所有 DP HBM 命中率汇总 |
| `ext_hit_rate_dp0` | DP0 External 命中率 |
| `ext_hit_rate_total` | 所有 DP External 命中率汇总 |

---

# FAQ

### Q1：加载 tokenizer 报错

检查 transformers 版本是否适配模型。GLM-5 系列需要 `transformers >= 5.0`，其他模型（Qwen、Llama、DeepSeek 等）需要 `transformers 4.x`。

```bash
# 非 GLM-5 模型
pip install transformers==4.57.6
# GLM-5 系列模型
pip install transformers==5.2.0
```

### Q2：ais_bench: error: unrecognized arguments: --num-warmups

部分 AISBench 版本不支持 `--num-warmups` 参数。修改 `prefix_bench.py` 中 `generate_aisbench_command()`，删除 `--num-warmups 0`。

### Q3：打屏不显示命中率信息

检查 `--pod_info` 是否配置正确。也可手动 curl 获取 metrics：

```bash
unset http_proxy && unset https_proxy
curl -s http://{ip}:{port}/metrics | grep prefix_cache
```

### Q4：Docker 场景 host_ip 校验失败

AISBench 校验 `host_ip` 必须为合法 IPv4/IPv6 地址，`host.docker.internal` 不合法。使用 `--url` 参数绕过：

```bash
python3 prefix_bench.py --host_ip localhost --url http://host.docker.internal:8080 ...
```

`--url` 非空时 AISBench 直接使用 URL 连接服务，忽略 host_ip/host_port，但 host_ip 仍需合法以满足校验。

### Q5：PD 分离场景 pod_info 怎么填？

- 单开 prefix cache：填 P 节点 IP + 每个 DP 域的 port
- 开启池化：填每个节点的 IP + 每个 DP 域的 port

### Q6：seed=0 纯随机说明

- `seed=0`：每次调用 `get_some_tokens()` 使用系统熵（`random.Random()`），结果不可复现
- `seed=非零值`：使用固定种子（`random.Random(seed)`），结果完全可复现
- 多轮测试中不同轮次可使用不同 seed，确保数据集内容不同

### Q7：多轮测试结果如何区分？

CSV/JSONL 中每行带 `round` 列标识轮次编号（1, 2, 3...），配合 `phase` 列区分 `warmup` / `full`。所有轮次结果追加到同一个文件。

### Q8：变长测试怎么用？

需要同时提供 `--length_mean` 和 `--length_std`（高斯分布参数），以及可选的 `--length_min` / `--length_max` 限制范围。此时每条数据的实际长度是围绕 `input_len`（作为均值）的高斯分布：

```bash
python3 prefix_bench.py \
    --input_len 32768 --output_len 300 --data_num 32 \
    --length_mean 32768 --length_std 49152 \
    --length_min 8192 --length_max 131072 \
    --repeat_rate 0.9 --dp 2 --seed 1
```
