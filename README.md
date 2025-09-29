<div align="center">

  # **LLM-Testing**
  #### 大模型服务化性能及精度测试指南
  <!-- 用分隔线替代背景 -->
  ---
</div>

## 1 工具安装
本项目主要使用AISBench工具进行大模型服务化的性能和精度测试，旨在全面评估大语言模型在不同场景下的表现，为模型的优化和调参提供可靠的数据支持。
AISBench工具请参照测试前准备[AISBench安装与卸载](./测评前准备/AISBench安装与卸载.md)章节进行。

## 2 测评数据准备
AISBench Benchmark当前支持的数据集类型如下：
1. 开源数据集，涵盖通用语言理解（如 ARC、SuperGLUE_BoolQ、MMLU）、数学推理（如 GSM8K、AIME2024、Math）、代码生成（如 HumanEval、MBPP、LiveCodeBench）、文本摘要（如 XSum、LCSTS）以及多模态任务（如 TextVQA、VideoBench、VocalSound）等多个方向，满足对语言模型在多任务、多模态、多语言等能力的全面评估需求。
2. 随机合成数据集，支持指定输入输出序列长度和请求数目，适用于对于序列分布场景和数据规模存在要求的性能测试场景。
3. 自定义数据集，支持将用户自定义的数据内容转换成固定格式的数据进行测评，适用于定制化精度和性能测试场景。
   
便于用户快速进行标准化测试，数据集的详细介绍和获取方式请查看：[AISBench数据集准备指南](https://gitee.com/aisbench/benchmark/blob/master/doc/users_guide/datasets.md)，请按照指南中的步骤进行性能测试和精度测试数据集的准备。

## 3 大模型服务化测评
### 3.1 服务化性能测评
MindIE推理引擎启动的大模型服务支持vllm兼容OpenAI接口，因此以下测评步骤以**MindIE推理引擎**大模型服务化模型性能测评为例，使用vllm流式对话任务：vllm_api_stream_chat。
- 功能描述：在真实部署环境中评估服务模型的运行效率（吞吐、延迟）；
- 要求：模型推理服务已使用MindIE部署且需支持**流式接口**方式访问；
- 注意：性能测评所占用的缓存大小与请求的上下文长度以及请求的数量成正比，因此通常与测评时长呈正相关增长。

**3.1.1 运行命令前置准备**

- 使用MindIE推理引擎部署大模型服务；
- 准备性能测评数据集，以gsm8k数据集为例，可以下载 [opencompass
提供的gsm8k数据集压缩包](http://opencompass.oss-cn-shanghai.aliyuncs.com/datasets/data/gsm8k.zip)。将解压后的`gsm8k/`文件夹部署到AISBench评测工具根路径下的`ais_bench/datasets`文件夹下。

**3.1.2 修改配置文件**

每个模型测评任务、数据集任务和结果呈现任务都对应一个配置文件，运行命令前需要修改这些配置文件的内容。这些配置文件路径可以通过在原有AISBench命令基础上加上`--search`来查询，例如：
```shell
# 注意search的命令中是否加 "--mode perf" 不影响搜索结果
ais_bench --models vllm_api_stream_chat --datasets demo_gsm8k_gen_4_shot_cot_chat_prompt --mode perf --search
```
> ⚠️ **注意**： 执行带search命令会打印出任务对应的配置文件的绝对路径。

执行查询命令可以得到如下查询结果：
```shell
06/28 11:52:25 - AISBench - INFO - Searching configs...
╒══════════════╤═══════════════════════════════════════╤════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╕
│ Task Type    │ Task Name                             │ Config File Path                                                                                                               │
╞══════════════╪═══════════════════════════════════════╪════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
│ --models     │ vllm_api_stream_chat                  │ /your_workspace/benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_general_chat.py                                 │
├──────────────┼───────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ --datasets   │ demo_gsm8k_gen_4_shot_cot_chat_prompt │ /your_workspace/benchmark/ais_bench/benchmark/configs/datasets/demo/demo_gsm8k_gen_4_shot_cot_chat_prompt.py                   │
╘══════════════╧═══════════════════════════════════════╧════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╛

```

模型配置文件`vllm_api_stream_chat.py`中包含了模型运行相关的配置内容，是需要依据实际情况修改的。**此项目示例测评任务需要修改的内容已用注释标明。**
```python
from ais_bench.benchmark.models import VLLMCustomAPIChatStream

models = [
    dict(
        attr="service",
        type=VLLMCustomAPIChatStream,
        abbr='vllm-api-stream-chat',
        path="",                    # 指定模型序列化词表文件绝对路径，一般来说就是模型权重文件夹路径
        model="DeepSeek-R1",        # 指定服务端已加载模型名称，依据实际MindIE推理服务拉取的模型名称配置（配置成空字符串会自动获取）
        request_rate = 0,           # 请求发送频率，每1/request_rate秒发送1个请求给服务端，小于0.1则一次性发送所有请求
        retry = 2,
        host_ip = "localhost",      # 指定推理服务的IP
        host_port = 8080,           # 指定推理服务的端口
        max_out_len = 512,          # 推理服务输出的token的最大数量
        batch_size=1,               # 请求发送的最大并发数
        generation_kwargs = dict(
            temperature = 0.5,
            top_k = 10,
            top_p = 0.95,
            seed = None,
            repetition_penalty = 1.03,
            ignore_eos = True,      # 推理服务输出忽略eos（输出长度一定会达到max_out_len）
        )
    )
]
```
此项目测评示例的数据集任务配置文件`demo_gsm8k_gen_4_shot_cot_chat_prompt.py`已默认配置好，不需要做额外修改。

**3.1.3 执行服务化性能测评命令**

修改好配置文件后，执行命令启动服务化性能评测（⚠️ 第一次执行建议加上`--debug`，可以将具体日志打屏，如果有请求推理服务过程中的报错更方便处理）：
```bash
# 命令行加上--debug
ais_bench --models vllm_api_stream_chat --datasets demo_gsm8k_gen_4_shot_cot_chat_prompt -m perf --debug
```
其中：
- `--models`指定了模型任务，即`vllm_api_stream_chat`模型任务。

- `--datasets`指定了数据集任务，即`demo_gsm8k_gen_4_shot_cot_chat_prompt`数据集任务。

- `-m`指定了评测场景，即`perf`性能评测场景。

- `--summarizer`指定了结果呈现任务，即`default_perf`结果呈现任务(不指定`--summarizer`精度评测场景默认使用`default_perf`任务)，一般使用默认，不需要在命令行中指定，后续命令不指定。

**3.1.4 查看性能结果**

性能结果打屏示例如下：
```bash
06/05 20:22:24 - AISBench - INFO - Performance Results of task: vllm-api-stream-chat/gsm8kdataset:

╒══════════════════════════╤═════════╤══════════════════╤══════════════════╤══════════════════╤══════════════════╤══════════════════╤══════════════════╤══════════════════╤══════╕
│ Performance Parameters   │ Stage   │ Average          │ Min              │ Max              │ Median           │ P75              │ P90              │ P99              │  N   │
╞══════════════════════════╪═════════╪══════════════════╪══════════════════╪══════════════════╪══════════════════╪══════════════════╪══════════════════╪══════════════════╪══════╡
│ E2EL                     │ total   │ 2048.2945  ms    │ 1729.7498 ms     │ 3450.96 ms       │ 2491.8789 ms     │ 2750.85 ms       │ 3184.9186 ms     │ 3424.4354 ms     │ 8    │
├──────────────────────────┼─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────┤
│ TTFT                     │ total   │ 50.332 ms        │ 50.6244 ms       │ 52.0585 ms       │ 50.3237 ms       │ 50.5872 ms       │ 50.7566 ms       │ 50 .0551 ms      │ 8    │
├──────────────────────────┼─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────┤
│ TPOT                     │ total   │ 10.6965 ms       │ 10.061 ms        │ 10.8805 ms       │ 10.7495 ms       │ 10.7818 ms       │ 10.808 ms        │ 10.8582 ms       │ 8    │
├──────────────────────────┼─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────┤
│ ITL                      │ total   │ 10.6965 ms       │ 7.3583 ms        │ 13.7707 ms       │ 10.7513 ms       │ 10.8009 ms       │ 10.8358 ms       │ 10.9322 ms       │ 8    │
├──────────────────────────┼─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────┤
│ InputTokens              │ total   │ 1512.5           │ 1481.0           │ 1566.0           │ 1511.5           │ 1520.25          │ 1536.6           │ 1563.06          │ 8    │
├──────────────────────────┼─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────┤
│ OutputTokens             │ total   │ 287.375          │ 200.0            │ 407.0            │ 280.0            │ 322.75           │ 374.8            │ 403.78           │ 8    │
├──────────────────────────┼─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┼──────┤
│ OutputTokenThroughput    │ total   │ 115.9216 token/s │ 107.6555 token/s │ 116.5352 token/s │ 117.6448 token/s │ 118.2426 token/s │ 118.3765 token/s │ 118.6388 token/s │ 8    │
╘══════════════════════════╧═════════╧══════════════════╧══════════════════╧══════════════════╧══════════════════╧══════════════════╧══════════════════╧══════════════════╧══════╛
╒══════════════════════════╤═════════╤════════════════════╕
│ Common Metric            │ Stage   │ Value              │
╞══════════════════════════╪═════════╪════════════════════╡
│ Benchmark Duration       │ total   │ 19897.8505 ms      │
├──────────────────────────┼─────────┼────────────────────┤
│ Total Requests           │ total   │ 8                  │
├──────────────────────────┼─────────┼────────────────────┤
│ Failed Requests          │ total   │ 0                  │
├──────────────────────────┼─────────┼────────────────────┤
│ Success Requests         │ total   │ 8                  │
├──────────────────────────┼─────────┼────────────────────┤
│ Concurrency              │ total   │ 0.9972             │
├──────────────────────────┼─────────┼────────────────────┤
│ Max Concurrency          │ total   │ 1                  │
├──────────────────────────┼─────────┼────────────────────┤
│ Request Throughput       │ total   │ 0.4021 req/s       │
├──────────────────────────┼─────────┼────────────────────┤
│ Total Input Tokens       │ total   │ 12100              │
├──────────────────────────┼─────────┼────────────────────┤
│ Prefill Token Throughput │ total   │ 17014.3123 token/s │
├──────────────────────────┼─────────┼────────────────────┤
│ Total generated tokens   │ total   │ 2299               │
├──────────────────────────┼─────────┼────────────────────┤
│ Input Token Throughput   │ total   │ 608.7438 token/s   │
├──────────────────────────┼─────────┼────────────────────┤
│ Output Token Throughput  │ total   │ 115.7835 token/s   │
├──────────────────────────┼─────────┼────────────────────┤
│ Total Token Throughput   │ total   │ 723.5273 token/s   │
╘══════════════════════════╧═════════╧════════════════════╛

06/05 20:22:24 - AISBench - INFO - Performance Result files locate in outputs/default/20250605_202220/performances/vllm-api-stream-chat.

```

**💡 具体性能参数的含义请参考下表**
|Performance Parameters|Stage|Average|Max|Min|Median|P75|P90|P99|N|
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
|E2EL|统计此参数的阶段|平均请求时延|最大请求时延|最小请求时延|请求时延中位数|请求时延75分位值|请求时延90分位值|请求时延99分位值|测试数据量，来源于输入参数|
|TTFT|统计此参数的阶段|首个token平均时延|首个token最大时延|首个token最小时延|首个token中位数时延|首个token75分位时延|首个token90分位时延|首个token99分位时延|测试数据量，来源于输入参数|
|TPOT|统计此参数的阶段|Decode阶段平均时延|最大Decode阶段时延|最小Decode阶段时延|Decode阶段中位数时延|75分位Decode阶段时延|90分位每条请求Decode阶段平均时延|99分位Decode阶段时延|测试数据量，来源于输入参数|
|ITL|统计此参数的阶段|token间平均时延|token间最大时延|token间最小时延|token间中位数时延|token间75分位时延|token间90分位时延|token间99分位时延|测试数据量，来源于输入参数|
|InputTokens|统计此参数的阶段|输入token平均长度|最大输入token长度|最小输入token长度|输入token中位数长度|75分位输入token长度|90分位输入token长度|99分位输入token长度|测试数据量，来源于输入参数|
|OutputTokens|统计此参数的阶段|输出token平均长度|最大输出token长度|最小输出token长度|输出token中位数长度|75分位输出token长度|90分位输出token长度|99分位输出token长度|测试数据量，来源于输入参数|
|OutputTokenThroughput|统计此参数的阶段|平均输出吞吐|最大输出吞吐|最小输出吞吐|中位数输出吞吐|输出吞吐75分位|输出吞吐90分位|输出吞吐99分位|测试数据量，来源于输入参数|

| 参数                           | 说明                    |
| ---------------------------- | ---------------------  |
| **Benchmark Duration**       | 测试任务的总执行时间            |
| **Total Requests**           | 请求总数量                 |
| **Failed Requests**          | 请求失败数量（包含无响应或响应为空）    |
| **Success Requests**         | 成功返回的请求数量（包括空响应与非空响应） |
| **Concurrency**              | 实际平均并发数               |
| **Max Concurrency**          | 配置的最大并发数              |
| **Request Throughput**       | 请求级吞吐率（请求数/秒）         |
| **Total Input Tokens**       | 所有请求的总输入 Token 数      |
| **Prefill Token Throughput** | Prefill 阶段的 Token 吞吐率 |
| **Total Output Tokens**      | 所有请求生成的总输出 Token 数    |
| **Input Token Throughput**   | 输入 Token 吞吐率          |
| **Output Token Throughput**  | 输出 Token 吞吐率          |
| **Total Token Throughput**   | 总 Token 吞吐率（输入 + 输出）  |

### 3.2 服务化精度测评
MindIE推理引擎启动的大模型服务支持vllm兼容OpenAI接口，因此以下测评步骤以**MindIE推理引擎**大模型服务化模型精度测评为例，使用vllm流式对话任务：vllm_api_stream_chat。
- 功能描述：评估部署为服务形式的模型在特定数据集上的预测准确率;
- 要求：模型推理服务已使用MindIE部署，需测试其实际服务能力;
  
**3.2.1 运行命令前置准备**

- 使用MindIE推理引擎部署大模型服务；
- 准备精度测评数据集，以gsm8k数据集为例，可以下载 [opencompass
提供的gsm8k数据集压缩包](http://opencompass.oss-cn-shanghai.aliyuncs.com/datasets/data/gsm8k.zip)。将解压后的`gsm8k/`文件夹部署到AISBench评测工具根路径下的`ais_bench/datasets`文件夹下。

**3.2.2 修改配置文件**

每个模型测评任务、数据集任务和结果呈现任务都对应一个配置文件，运行命令前需要修改这些配置文件的内容。这些配置文件路径可以通过在原有AISBench命令基础上加上`--search`来查询，例如：
```shell
# 注意search的命令中是否加 "--mode all" 不影响搜索结果
ais_bench --models vllm_api_stream_chat --datasets demo_gsm8k_gen_4_shot_cot_chat_prompt --mode all --search
```
> ⚠️ **注意**： 执行带search命令会打印出任务对应的配置文件的绝对路径。

执行查询命令可以得到如下查询结果：
```shell
06/28 11:52:25 - AISBench - INFO - Searching configs...
╒══════════════╤═══════════════════════════════════════╤════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╕
│ Task Type    │ Task Name                             │ Config File Path                                                                                                               │
╞══════════════╪═══════════════════════════════════════╪════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
│ --models     │ vllm_api_stream_chat                  │ /your_workspace/benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_general_chat.py                                 │
├──────────────┼───────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ --datasets   │ demo_gsm8k_gen_4_shot_cot_chat_prompt │ /your_workspace/benchmark/ais_bench/benchmark/configs/datasets/demo/demo_gsm8k_gen_4_shot_cot_chat_prompt.py                   │
╘══════════════╧═══════════════════════════════════════╧════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╛

```

模型配置文件`vllm_api_stream_chat.py`中包含了模型运行相关的配置内容，是需要依据实际情况修改的。**此项目示例测评任务需要修改的内容已用注释标明。**
```python
from ais_bench.benchmark.models import VLLMCustomAPIChat

models = [
    dict(
        attr="service",
        type=VLLMCustomAPIChat,
        abbr='vllm-api-general-chat',
        path="",
        model="DeepSeek-R1",        # 指定服务端已加载模型名称，依据实际MindIE推理服务拉取的模型名称配置（配置成空字符串会自动获取）
        request_rate = 0,
        retry = 2,
        host_ip = "localhost",      # 指定推理服务的IP
        host_port = 8080,           # 指定推理服务的端口
        max_out_len = 512,
        batch_size=1,
        generation_kwargs = dict(
            temperature = 0.5,
            top_k = 10,
            top_p = 0.95,
            seed = None,
            repetition_penalty = 1.03,
        )
    )
]
```
此项目测评示例的数据集任务配置文件`demo_gsm8k_gen_4_shot_cot_chat_prompt.py`已默认配置好，不需要做额外修改。

**3.2.3 执行服务化精度测评命令**

修改好配置文件后，执行命令启动服务化精度评测（⚠️ 第一次执行建议加上`--debug`，可以将具体日志打屏，如果有请求推理服务过程中的报错更方便处理）：
```bash
# 命令行加上--debug
ais_bench --models vllm_api_stream_chat --datasets demo_gsm8k_gen_4_shot_cot_chat_prompt -m all --debug
```
其中：
- `--models`指定了模型任务，即`vllm_api_stream_chat`模型任务。

- `--datasets`指定了数据集任务，即`demo_gsm8k_gen_4_shot_cot_chat_prompt`数据集任务。

- `-m`指定了评测场景，即`all`精度评测场景。

- `--summarizer`指定了结果呈现任务，即`default_perf`结果呈现任务(不指定`--summarizer`精度评测场景默认使用`default_perf`任务)，一般使用默认，不需要在命令行中指定，后续命令不指定。

**3.2.4 查看精度结果**

因为只有8条数据，会很快跑出结果，结果显示的示例如下：
```bash
dataset                 version  metric   mode  vllm_api_general_chat
----------------------- -------- -------- ----- ----------------------
demo_gsm8k              401e4c   accuracy gen                   62.50
```
-  `metric`表示测评指标，`accuracy`指采用正确率进行精度计算。
-  `mode`表示测评模式，`gen`指生成模式。
-  `vllm_api_general_chat`表示模型测评任务名称，`62.50`表示在`vllm_api_stream_chat`模型测评任务下，该数据集上的正确率。