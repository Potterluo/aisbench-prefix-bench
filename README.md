# aisbench_auto_tools_prefix

A streamlined tool for **prefix cache performance testing** using AISBench.

## 核心特性

- **纯随机数据集生成**：基于 tokenizer 词表随机采样 token，不依赖 GSM8K，多轮测试间无 prefix collision
- **灵活配置**：config.py 可手动填写默认值，命令行参数优先级更高可覆盖，方便手动执行和 CI 集成
- **多轮测试支持**：`--rounds` 参数支持一次运行多组不同配置，每组可覆盖 input_len、concurrency、seed 等
- **命中率写入结果表格**：分 per-DP 和汇总，分 HBM 和 external，写入 CSV 和 JSONL
- **两阶段前缀测试**：Phase 1 warmup 前缀 → Phase 2 全量测试

## 适用场景

1. vLLM prefix cache 性能测试
2. PD 分离场景前缀命中率测试
3. 多轮不同配置的批量性能测试（CI 集成）

## 使用方法

进入带 [AISBench](https://github.com/AISBench/benchmark) 的环境 → 修改 `config.py`（可选）→ 运行 `python prefix_bench.py` 命令

## 配置优先级

**CLI 命令行参数 > config.py 配置文件 > 硬编码默认值**

- 手动执行时：编辑 `config.py` 填写常用值（IP、端口、模型路径等），命令行只需传变化的参数
- CI 集成时：全部通过命令行参数传递，忽略 config.py
- 多轮测试时：`--rounds` 中每轮的 override 字段优先级高于 CLI 基值

例如 config.py 中 `HOST_IP="10.0.0.1"`，运行时 `--host_ip 10.0.0.2` 则使用 10.0.0.2。

## 参数详解

`python prefix_bench.py --help` 查看所有参数

### 服务配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --host_ip | str | localhost | 目标服务 IP（需为合法 IPv4/IPv6） |
| --host_port | int | 8000 | 目标服务端口 |
| --url | str | "" | 完整 URL，覆盖 host_ip/host_port（Docker 场景必填） |
| --model_name | str | "" | 模型名称（空则 AISBench 自动检测） |
| --model_path | str | **必填** | tokenizer/模型权重路径 |
| --npu_num | int | 1 | NPU 卡数（用于单卡吞吐计算） |

### 测试配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --input_len | int | 3500 | 输入 token 长度 |
| --output_len | int | 1500 | 输出 token 长度 |
| --data_num | int | 8192 | 数据集条数 |
| --concurrency | int | 2048 | 全量测试阶段最大并发 |
| --request_rate | int | 0 | 请求频率，0 为突发 |
| --test_type | str | stream | API 类型：stream 或 text |

### 前缀配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --prefix_num | int | 1 | 前缀种类数 |
| --repeat_rate | float | 0.5 | 前缀重复率，支持 0.5 或 "50%"（--rounds 中可传字符串） |
| --dp | int | 1 | DP 域数量 |
| --seed | int | 1 | 随机种子；0 表示纯随机（不可复现） |

### 变长配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --length_mean | int | None | 高斯均值 |
| --length_std | float | None | 高斯标准差 |
| --length_min | int | None | 最小长度 |
| --length_max | int | None | 最大长度 |

### AISBench 配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --work_path | str | **必填** | AISBench 安装路径 |
| --dataset_path | str | ./datasets | 数据集存储目录 |
| --output_dir | str | ./outputs/default | AISBench 输出目录 |
| --summarizer | str | default_perf | AISBench summarizer（default_perf 或 stable_stage） |
| --enable_think | flag | False | DeepSeek V3.1 思考模式 |

### 结果与命中率

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --result_csv | str | prefix_bench_result.csv | 结果 CSV 径 |
| --result_jsonl | str | prefix_bench_result.jsonl | 结果 JSONL 径 |
| --pod_info | str[] | None | Pod 地址（ip:port），默认用 host_ip:host_port |

### 数据集覆盖

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --dataset | str | None | 使用已有数据集文件，跳过生成 |

### 多轮测试配置

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| --rounds | str | None | 多轮测试配置：JSON 文件路径或内联 JSON 字符串 |

**`--rounds` 格式**：每轮是一个 dict，可覆盖以下字段（缺省字段继承 CLI/config.py 默认值）：

- `input_len`、`output_len`、`data_num`、`concurrency`、`request_rate`
- `prefix_num`、`repeat_rate`、`dp`、`seed`

不可覆盖字段（全局共享）：`host_ip`、`host_port`、`url`、`model_name`、`model_path` 等。

**示例 JSON**：
```json
[
  {"input_len": 3500, "output_len": 1500, "seed": 1},
  {"input_len": 8500, "seed": 2},
  {"input_len": 12000, "concurrency": 4096, "seed": 0}
]
```

- `seed=0` 表示纯随机（使用系统熵，不可复现）
- `repeat_rate` 在 JSON 中可传 `"50%"` 或 `0.5`

## 数据集生成逻辑

完全基于 tokenizer 词表的随机采样，参考 `token_counter.py` 的 `get_some_tokens()` 方法：

- **前缀**：调用 `get_some_tokens(prefix_len, seed=seed+i)` 随机采样，每个 seed 偏移确保不同前缀
- **分隔**：每行 3 个随机 token，`get_some_tokens(3, seed=seed+prefix_num+i)`
- **后缀**：`get_some_tokens(suffix_len, seed=seed+prefix_num+data_num+i)`
- **拼接**：`prefix + separator + suffix`

**关键改进**：
- ✅ 不依赖 GSM8K.jsonl，纯随机生成
- ✅ 不使用 picked_ids.txt，无全局状态累积
- ✅ Seed 完全控制所有文本生成，多轮测试间无 prefix collision
- ✅ 使用 local RNG (random.Random(seed))，不污染全局 random 状态
- ✅ seed=0 时使用系统熵（纯随机，不可复现）

## 命令示例

### 1. 基本前缀测试（dp=2）

```bash
python prefix_bench.py \
    --model_path /home/weights/model \
    --work_path /home/benchmark \
    --host_ip 141.xx.xx.xx --host_port 8004 \
    --input_len 8192 --output_len 1 --data_num 32 \
    --concurrency 8 --request_rate 0 \
    --repeat_rate 0.5 --prefix_num 1 --seed 1 --dp 2
```

### 2. 多轮不同配置测试（--rounds JSON）

```bash
python prefix_bench.py \
    --model_path /home/weights/model \
    --work_path /home/benchmark \
    --host_ip 141.xx.xx.xx --host_port 8004 \
    --rounds '[{"input_len":3500,"output_len":1500,"concurrency":8,"seed":1},
               {"input_len":8500,"output_len":1500,"concurrency":8,"seed":2},
               {"input_len":8192,"output_len":1,"concurrency":16,"seed":0}]'
```

或使用 JSON 文件：
```bash
python prefix_bench.py \
    --model_path /home/weights/model \
    --work_path /home/benchmark \
    --host_ip 141.xx.xx.xx --host_port 8004 \
    --rounds rounds_config.json
```

所有结果追加到同一个 CSV，每行带 `round` 列标识轮次。

### 3. 传统 shell 脚本多轮测试（仍然有效）

```bash
for cc in 8 16 32 48 56; do
    python prefix_bench.py \
        --model_path /home/weights/model \
        --work_path /home/benchmark \
        --host_ip 141.xx.xx.xx --host_port 8004 \
        --input_len 8192 --output_len 1 --data_num $(($cc * 4)) \
        --concurrency $cc --request_rate 0 \
        --repeat_rate 0.5 --prefix_num 1 --seed $cc --dp 2 \
        --pod_info "141.xx.xx.11:8000" "141.xx.xx.12:8000" \
        --result_csv prefix_bench_result.csv
done
```

### 4. 变长测试（8k~128k，均值32k）

```bash
python prefix_bench.py \
    --model_path /home/weights/model \
    --work_path /home/benchmark \
    --host_ip 141.xx.xx.xx --host_port 8004 \
    --input_len 32768 --output_len 300 --data_num 32 \
    --concurrency 8 --request_rate 0 \
    --repeat_rate 0.9 --prefix_num 1 --seed 1 --dp 2 \
    --length_mean 32768 --length_std 49152 \
    --length_min 8192 --length_max 131072
```

### 5. Docker 内运行（AISBench 镜像）

Docker 容器中访问宿主机服务时，`host.docker.internal` 不是合法 IPv4，AISBench 校验会失败。使用 `--url` 参数绕过：

```bash
docker run --rm \
  --add-host=host.docker.internal:host-gateway \
  -v /path/to/tokenizer:/tokenizer:ro \
  -v /path/to/datasets:/benchmark/ais_bench/datasets/gsm8k \
  -v /path/to/model_config:/benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_chat_temp.py \
  -v /path/to/output:/output \
  aisbench_image \
  ais_bench --models vllm_api_chat_temp --datasets gsm8k_gen_0_shot_cot_str_perf \
  --mode perf --summarizer default_perf --work-dir /output --debug --num-warmups 0
```

配置 `--url http://host.docker.internal:8080 --host_ip localhost` 即可让 AISBench 使用 URL 直连宿主机，同时 `host_ip` 满足校验。

### 6. PD 分离场景命中率采集

```bash
python prefix_bench.py \
    --model_path /home/weights/model \
    --work_path /home/benchmark \
    --host_ip 141.xx.xx.xx --host_port 8004 \
    --input_len 8192 --output_len 1 --data_num 32 \
    --concurrency 8 --dp 2 --repeat_rate 0.5 \
    --pod_info "141.xx.xx.11:8000" "141.xx.xx.12:8000"
```

## 结果获取

### CSV 文件
`prefix_bench_result.csv` — 包含所有性能指标 + 命中率

列包括：
- 时间与配置：timestamp, round, phase, input_len, output_len, concurrency, dp, repeat_rate, seed 等
- 性能指标：total_requests, ttft_avg_ms, tpot_avg_ms, throughput 等
- 命中率（per-DP）：hbm_hit_rate_dp0, hbm_queries_dp0, hbm_hits_dp0, ext_hit_rate_dp0 等
- 命中率（汇总）：hbm_hit_rate_total, hbm_queries_total, hbm_hits_total, ext_hit_rate_total 等

### JSONL 文件
`prefix_bench_result.jsonl` — 每行一个 JSON 对象，方便程序化读取

### 日志
- `aisbench.log` — 当前 AISBench 测试日志
- `aisbench_all.log` — 所有测试日志的汇总
- `outputs/default/时间戳/` — AISBench 原始输出目录

## 命中率说明

命中率数据来自 vLLM `/metrics` 端点，区分：
- **HBM（片上缓存）**：`vllm:prefix_cache_queries_total` / `vllm:prefix_cache_hits_total`
- **External（外部缓存）**：`external_prefix_cache_queries_total` / `external_prefix_cache_hits_total`

计算公式：
```
hit_rate = (hits_after - hits_before) / (queries_after - queries_before)
```

Per-DP 数据按 `engine_id` 分组，汇总为所有 DP 的总和。

## FAQ

### 1、加载 tokenizer 报错

检查 transformers 版本是否适配模型，如 GLM5 需更新 mindie/vllm 镜像内 transformers 版本。

### 2、ais_bench: error: unrecognized arguments: --num-warmups

修改 prefix_bench.py 中 `generate_aisbench_command()`，删除 `--num-warmups 0`。详见 [AISBench GitHub](https://github.com/AISBench/benchmark)。

### 3、打屏不显示命中率信息

检查 --pod_info 是否配置正确。也可手动 curl 获取 metrics：
```bash
unset http_proxy && unset https_proxy
curl -s http://{ip}:{port}/metrics | grep prefix
```

### 4、Docker 场景 host_ip 校验失败

AISBench 校验 `host_ip` 必须为合法 IPv4/IPv6 地址，Docker 中 `host.docker.internal` 不合法。使用 `--url` 参数：

```bash
python prefix_bench.py --host_ip localhost --url http://host.docker.internal:8080 ...
```

`--url` 非空时 AISBench 直接使用 URL 连接服务，忽略 host_ip/host_port，但 host_ip 仍需合法以满足校验。

### 5、PD 分离场景 pod_info 配置

- 单开 prefix cache：配置 P 节点 IP + 对应 DP 域的 port
- 开启池化：配置每个节点的 IP + 对应 DP 域的 port

### 6、seed=0 纯随机说明

- `seed=0`：每次调用 `get_some_tokens()` 使用系统熵（`random.Random()`），结果不可复现
- `seed=非零值`：使用固定种子（`random.Random(seed)`），结果完全可复现
- 多轮测试中不同轮次可使用不同 seed，确保数据集内容不同

