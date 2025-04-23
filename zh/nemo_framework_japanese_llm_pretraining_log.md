# NeMo Framework 日语LLM继续预训练操作记录

## 操作概述

本文档记录使用NVIDIA NeMo Framework对Llama-3.1-8B模型进行日语继续预训练的操作流程、可能遇到的问题及解决方案。参考的是NVIDIA官方博客[《NeMo Framework で実践する継続事前学習 – 日本語 LLM 編 –》](https://developer.nvidia.com/ja-jp/blog/how-to-use-continual-pre-training-with-japanese-language-on-nemo-framework/)。

## NeMo Framework 简介

NeMo Framework 是NVIDIA提供的一个云原生框架，用于构建和自定义LLM等生成式AI模型。该框架通过NGC平台提供容器，可以立即开始使用。

NeMo Framework可以免费使用，但作为NVIDIA AI Enterprise的组件，企业用户如需支持服务可以考虑购买NVIDIA AI Enterprise许可证。

### LLM 开发工作流程

LLM的开发涉及以下任务：

- 大规模数据准备（用于预训练）
- 利用分布式学习进行LLM预训练
- 通过微调、对齐和提示工程来自定义LLM
- 优化模型以加速推理
- 充分利用GPU进行LLM服务
- 通过RAG（检索增强生成）以较低成本为LLM提供最新信息
- 为防止LLM应用出现意外行为而设置护栏

### NeMo Framework 组件

NeMo Framework容器包含从数据准备到LLM训练和自定义所需的多个模块，这些模块包括：

- **NeMo Curator**：用于下载、提取、清洗和过滤LLM训练所需的大规模数据集的可扩展工具包
- **NeMo**：用于构建LLM、多模态和语音等生成式AI模型的可扩展框架
- **NeMo Framework Launcher**：用于从云端或本地集群启动任务的工具包
- **Megatron-LM**：研究Transformer模型大规模训练的项目
- **Transformer Engine**：以FP8为核心加速Transformer模型的工具包
- **NeMo-Aligner**：使用RLHF（基于人类反馈的强化学习）、DPO、SteerLM等高效对齐LLM的工具包

这些库在GitHub上以开源形式发布，但建议通过依赖关系已解决的NeMo Framework容器使用。在容器中，这些模块位于`/opt`目录下。

## 继续预训练（Continual Pre-training）

继续预训练是指在现有预训练模型的基础上，使用特定领域或特定语言的数据进行进一步的预训练，以使模型更好地适应特定的应用场景。

在本例中，我们将使用日语维基百科数据对Llama-3.1-8B模型进行继续预训练，以提高其在日语处理任务上的表现。虽然本教程使用单节点和较小规模的日语维基百科数据，但通过增加数据量和使用多节点计算环境，可以轻松扩展为大规模训练。

## 环境准备

### 硬件配置

我的实际环境配置：

- 硬件：
  - CPU: Intel(R) Xeon(R) Silver 4310 CPU @ 2.10GHz
  - GPU: 
    - NVIDIA RTX 6000 Ada Generation (48GB)
    - NVIDIA T600 (4GB)
  - 系统内存: 251GB

- 软件：
  - OS: Ubuntu 20.04.6 LTS
  - 容器: `nvcr.io/nvidia/nemo:25.02.01`

### 原始博客验证环境

原始博客的验证环境为：

- 硬件：
  - DGX H100
  - GPU: 8 x NVIDIA H100 80GB GPUs (驱动版本: 550.90.7)
  - CPU: Intel(R) Xeon(R) Platinum 8480C
  - 系统内存: 2 TB

- 软件：
  - OS: Ubuntu 22.04.5 LTS
  - 容器: `nvcr.io/nvidia/nemo:24.09`

## 实际操作过程

### 1. 创建工作目录

按照教程，首先创建工作目录并进入该目录：

```bash
# 创建工作目录
mkdir cp-example
cd cp-example
```

执行结果：
```
tut-server-11% mkdir -p cp-example && cd cp-example && pwd
/home/cho/workspace/cp-example
```

当前工作目录已成功设置为 `/home/cho/workspace/cp-example`。

### 2. 启动Docker容器

执行以下命令启动Docker容器：

```bash
docker run --rm -it --gpus all --shm-size=16g --ulimit memlock=-1 --network=host -v ${PWD}:/workspace -w /workspace nvcr.io/nvidia/nemo:25.02.01 bash
```

此命令的参数说明：
- `--rm`：容器停止运行后自动删除容器
- `-it`：交互式终端
- `--gpus all`：使用所有可用的GPU
- `--shm-size=16g`：设置共享内存大小为16GB
- `--ulimit memlock=-1`：取消内存锁定限制
- `--network=host`：使用主机网络
- `-v ${PWD}:/workspace`：将当前目录挂载到容器的/workspace目录
- `-w /workspace`：设置容器的工作目录为/workspace

执行结果：
```
Unable to find image 'nvcr.io/nvidia/nemo:25.02.01' locally
25.02.01: Pulling from nvidia/nemo
de44b265507a: Pulling fs layer 
75f58769314d: Pulling fs layer 
...
Digest: sha256:be271767724ac6b03c22c2c1d7b1de08e82771eacc0b97a825a8a50bef70e1c9
Status: Downloaded newer image for nvcr.io/nvidia/nemo:25.02.01

====================
== NeMo Framework ==
====================

NVIDIA Release  (build 134983853)
Container image Copyright (c) 2025, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
...
NOTE: CUDA Forward Compatibility mode ENABLED.
  Using CUDA 12.8 driver version 570.86.10 with kernel driver version 535.183.01.
  See https://docs.nvidia.com/deploy/cuda-compatibility/ for details.

root@tut-server-11:/workspace# 
```

容器已成功启动，当前处于容器内的bash环境，工作目录为`/workspace`，这是主机上`/home/cho/workspace/cp-example`目录的映射位置。

### 3. 从Hugging Face下载预训练模型

登录Hugging Face（需要获得meta-llama/Llama-3.1-8B的访问权限）：

```bash
huggingface-cli login
```

执行此命令后，系统会提示输入Hugging Face令牌。这个令牌可以通过以下步骤获取：
1. 登录到 [Hugging Face官网](https://huggingface.co/)
2. 访问 [设置页面](https://huggingface.co/settings/tokens)
3. 创建新令牌，给予读取权限
4. 复制生成的令牌

另外，使用Llama模型需要先申请访问权限：
1. 访问 [meta-llama/Llama-3.1-8B模型页面](https://huggingface.co/meta-llama/Llama-3.1-8B)
2. 点击"Access"按钮申请访问权限
3. 填写相关表单并等待批准

执行结果：
```
    _|    _|  _|    _|    _|_|_|    _|_|_|  _|_|_|  _|      _|    _|_|_|      _|_|_|_|    _|_|      _|_|_|  _|_|_|_|
    _|    _|  _|    _|  _|        _|          _|    _|_|    _|  _|            _|        _|    _|  _|        _|
    _|_|_|_|  _|    _|  _|  _|_|  _|  _|_|    _|    _|  _|  _|  _|  _|_|      _|_|_|    _|_|_|_|  _|        _|_|_|
    _|    _|  _|    _|  _|    _|  _|    _|    _|    _|    _|_|  _|    _|      _|        _|    _|  _|        _|
    _|    _|    _|_|      _|_|_|    _|_|_|  _|_|_|  _|      _|    _|_|_|      _|        _|    _|    _|_|_|  _|_|_|_|

    To log in, `huggingface_hub` requires a token generated from https://huggingface.co/settings/tokens .
Enter your token (input will not be visible): 
Add token as git credential? (Y/n) y
Token is valid (permission: fineGrained).
The token has been saved to /root/.cache/huggingface/stored_tokens
Cannot authenticate through git-credential as no helper is defined on your machine.
You might have to re-authenticate when pushing to the Hugging Face Hub.
Run the following command in your terminal in case you want to set the 'store' credential helper as default.

git config --global credential.helper store

Read https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage for more details.
Token has not been saved to git credential helper.
Your token has been saved to /root/.cache/huggingface/token
Login successful.
```

Hugging Face登录成功完成，令牌已保存在容器中。现在可以继续下载模型。

下一步，创建一个Python脚本来下载Llama-3.1-8B模型。在`src`目录中创建`download_model.py`文件：

```bash
mkdir -p src
```

创建以下内容的Python脚本：

```python
# src/download_model.py
import os
from huggingface_hub import snapshot_download
 
MODEL_DIR = "./models"
os.makedirs(MODEL_DIR, exist_ok=True)

snapshot_download(
    repo_id="meta-llama/Llama-3.1-8B",
    local_dir=f"{MODEL_DIR}/Llama-3.1-8B",
)
```

执行脚本时遇到了以下权限错误：

```
huggingface_hub.errors.GatedRepoError: 403 Client Error.
Cannot access gated repo for url https://huggingface.co/meta-llama/Llama-3.1-8B/...
Access to model meta-llama/Llama-3.1-8B is restricted and you are not in the authorized list. 
Visit https://huggingface.co/meta-llama/Llama-3.1-8B to ask for access.
```

这个错误表明虽然我们已经登录了Hugging Face账号，但账号没有访问meta-llama/Llama-3.1-8B模型的权限。需要完成以下步骤：

1. 访问[meta-llama/Llama-3.1-8B模型页面](https://huggingface.co/meta-llama/Llama-3.1-8B)
2. 点击"Access"按钮申请访问权限
3. 填写相关申请表单（包括使用目的等）
4. 等待Meta AI批准访问请求

批准通常不是即时的，可能需要等待一段时间。获得访问权限后，需要重新登录并再次执行下载脚本。

在等待权限批准的过程中，为了继续教程的其他部分，我们可以考虑使用以下替代方案：
1. 使用已经有公开访问权限的其他模型（例如Mistral、Falcon等）
2. 使用已经下载好的模型（如果有）

开始从Hugging Face下载Llama-3.1-8B模型的所有文件。根据网络状况，这个过程可能需要几分钟到几十分钟不等。

### 4. 数据准备

使用llm-jp-corpus-v3中的日语维基百科数据：

```bash
cd /workspace/
mkdir -p data/ja_wiki
wget -O data/ja_wiki/train_0.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_0.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_1.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_1.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_2.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_2.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_3.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_3.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_4.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_4.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_5.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_5.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_6.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_6.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_7.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_7.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_8.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_8.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_9.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_9.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_10.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_10.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_11.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_11.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_12.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_12.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/train_13.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/train_13.jsonl.gz?ref_type=heads
wget -O data/ja_wiki/validation_0.jsonl.gz --no-check-certificate https://gitlab.llm-jp.nii.ac.jp/datasets/llm-jp-corpus-v3/-/raw/main/ja/ja_wiki/validation_0.jsonl.gz?ref_type=heads
gunzip data/ja_wiki/*
```

### 5. 数据预处理

创建数据处理目录并进行预处理：

```bash
mkdir ds

# 训练数据处理
python /opt/NeMo/scripts/nlp_language_modeling/preprocess_data_for_megatron.py --input="/workspace/data/ja_wiki" --json-keys=text --tokenizer-library=huggingface --tokenizer-type="meta-llama/Llama-3.1-8B" --dataset-impl mmap --append-eod --output-prefix="/workspace/ds/train" --workers=24 --files-filter '**/train_*.json*' --preproc-folder --log-interval 10000

# 验证数据处理
python /opt/NeMo/scripts/nlp_language_modeling/preprocess_data_for_megatron.py --input="/workspace/data/ja_wiki" --json-keys=text --tokenizer-library=huggingface --tokenizer-type="meta-llama/Llama-3.1-8B" --dataset-impl mmap --append-eod --output-prefix="/workspace/ds/validation" --workers=24 --files-filter '**/validation_*.json*' --preproc-folder --log-interval 1000
```

预处理后将生成以下文件：
```
train_text_document.bin
train_text_document.idx
validation_text_document.bin
validation_text_document.idx
```

### 6. 模型格式转换

为了将Hugging Face格式的模型转换为NeMo格式，需要执行以下步骤：

首先，对NeMo容器应用一些补丁（在Llama-3.1模型上需要）：

```bash
cd /opt/NeMo/
curl -L https://github.com/NVIDIA/NeMo/pull/11548.diff | git apply
curl -L https://github.com/NVIDIA/NeMo/pull/11580.diff | git apply
```

然后执行转换脚本：

```bash
export INPUT="/workspace/models/Llama-3.1-8B"
export OUTPUT="/workspace/models/Llama-3.1-8B.nemo"
export PREC="bf16"

python /opt/NeMo/scripts/checkpoint_converters/convert_llama_hf_to_nemo.py --input_name_or_path ${INPUT} --output_path ${OUTPUT} --precision ${PREC} --llama31 True
```

生成的`Llama-3.1-8B.nemo`文件使用了distributed checkpoint，可以在不更改checkpoint的情况下，使用任意的Tensor Parallel (TP)或Pipeline Parallel (PP)等模型并行的组合加载。

### 7. 执行继续预训练

继续预训练可以使用NeMo的`/opt/NeMo/examples/nlp/language_modeling/megatron_gpt_pretraining.py`执行。可以通过以下方式配置参数：

```bash
export NEMO_MODEL="/workspace/models/Llama-3.1-8B.nemo"
export TOKENIZER="/workspace/models/Llama-3.1-8B/tokenizer.json"
export TP_SIZE=2
export SP=False
export PP_SIZE=1
export EP_SIZE=1

TRAIN_DATA_PATH="{train:[1.0,/workspace/ds/train_text_document],validation:[/workspace/ds/validation_text_document],test:[/workspace/ds/validation_text_document]}"
```

[注意：实际执行命令将在操作过程中补充]

## 评估方法

### Nejumi リーダーボード 3 评估

Nejumi リーダーボード 3是一个用于多方面评估LLM日语能力的基准测试。通过运行基准测试脚本，可以将自己的模型与各种模型进行比较。

根据博客作者的评估，在MT-bench（160个样本）上比较原始的meta-llama/Llama-3.1-8B和经过日语继续预训练的模型时发现：

- 对于日语指令，原始模型约有30-40%的回复是英语
- 经过日语继续预训练的模型，英语回复减少到了极少数情况

这表明通过日语继续预训练，可以显著提高模型对日语指令的理解和响应能力。

## 可能遇到的问题及解决方案

1. **Hugging Face权限问题**:
   - 问题：无法访问meta-llama/Llama-3.1-8B模型
   - 解决方案：确保已在Hugging Face网站上申请并获得访问权限

2. **GPU内存不足**:
   - 问题：如果使用的不是H100 GPU，可能会遇到内存不足问题
   - 解决方案：减小批量大小或使用梯度累积

3. **数据下载失败**:
   - 问题：日语Wikipedia数据下载失败
   - 解决方案：确保网络连接稳定，或使用镜像站点

## 参考资料

- [NeMo Framework で実践する継続事前学習 – 日本語 LLM 編 –](https://developer.nvidia.com/ja-jp/blog/how-to-use-continual-pre-training-with-japanese-language-on-nemo-framework/)
- NVIDIA NeMo Framework 用户指南
- NeMo Framework 单节点预训练文档
- [Training Localized Multilingual LLMs with NVIDIA NeMo, Part 1](https://developer.nvidia.com/blog/training-localized-multilingual-llms-with-nvidia-nemo-part-1/)
- [Training Localized Multilingual LLMs with NVIDIA NeMo, Part 2](https://developer.nvidia.com/blog/training-localized-multilingual-llms-with-nvidia-nemo-part-2/)
- [NeMo Curator を使った日本語データのキュレーション](https://developer.nvidia.com/ja-jp/blog/using-nemo-curator-for-japanese-data-curation/)
- [NeMo Framework で日本語 LLM をファインチューニング – SFT 編 –](https://developer.nvidia.com/ja-jp/blog/fine-tuning-japanese-llm-with-nemo-framework-sft/)
- [NeMo Framework で日本語 LLM をファインチューニング – PEFT 編 –](https://developer.nvidia.com/ja-jp/blog/fine-tuning-japanese-llm-with-nemo-framework-peft/)
- [NeMo Framework で日本語 LLM をファインチューニング – DPO 編 –](https://developer.nvidia.com/ja-jp/blog/fine-tuning-japanese-llm-with-nemo-framework-dpo/)
- [NVIDIA NIM でファインチューニングされた AI モデルのデプロイ](https://developer.nvidia.com/ja-jp/blog/deploying-fine-tuned-ai-models-with-nvidia-nim/) 