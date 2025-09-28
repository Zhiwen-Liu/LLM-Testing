# AISBench数据集准备
## 支持数据集类型
AISBench Benchmark当前支持的数据集类型如下：
1. 开源数据集，涵盖通用语言理解（如 ARC、SuperGLUE_BoolQ、MMLU）、数学推理（如 GSM8K、AIME2024、Math）、代码生成（如 HumanEval、MBPP、LiveCodeBench）、文本摘要（如 XSum、LCSTS）以及多模态任务（如 TextVQA、VideoBench、VocalSound）等多个方向，满足对语言模型在多任务、多模态、多语言等能力的全面评估需求。
2. 随机合成数据集，支持指定输入输出序列长度和请求数目，适用于对于序列分布场景和数据规模存在要求的性能测试场景。
3. 自定义数据集，支持将用户自定义的数据内容转换成固定格式的数据进行测评，适用于定制化精度和性能测试场景。

便于用户快速进行标准化测试，数据集的详细介绍和获取方式请查看：

[AISBench数据集准备指南](https://gitee.com/aisbench/benchmark/blob/master/doc/users_guide/datasets.md)