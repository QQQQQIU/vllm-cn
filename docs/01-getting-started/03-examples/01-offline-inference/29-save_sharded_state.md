---
title: Save Sharded State
---

[\*在线运行 vLLM 入门教程：零基础分步指南](https://openbayes.com/console/public/tutorials/rXxb5fZFr29?utm_source=vLLM-CNdoc&utm_medium=vLLM-CNdoc-V1&utm_campaign=vLLM-CNdoc-V1-25ap)

源码 [examples/offline_inference/save_sharded_state.py](https://github.com/vllm-project/vllm/blob/main/examples/offline_inference/save_sharded_state.py)

```python
# SPDX-License-Identifier: Apache-2.0
"""
将每个工作进程(worker)的模型状态字典直接保存到检查点，
这为大型张量并行模型提供了快速加载路径 - 每个工作进程只需读取自己的分片，
而无需读取整个检查点。

示例用法：
python save_sharded_state.py \
--model /path/to/load \
--quantization deepspeedfp \
--tensor-parallel-size 8 \
--output /path/to/save
Then, the model can be loaded with
llm = LLM(
model="/path/to/save",
load_format="sharded_state",
quantization="deepspeedfp",
tensor_parallel_size=8,
)
"""
import dataclasses
import os
import shutil
from pathlib import Path

from vllm import LLM, EngineArgs
from vllm.utils import FlexibleArgumentParser

parser = FlexibleArgumentParser()
EngineArgs.add_cli_args(parser)
parser.add_argument("--output",
                    "-o",
                    required=True,
                    type=str,
                    help="path to output checkpoint")
parser.add_argument("--file-pattern",
                    type=str,
                    help="string pattern of saved filenames")
parser.add_argument("--max-file-size",
                    type=str,
                    default=5 * 1024**3,
                    help="max size (in bytes) of each safetensors file")


def main(args):
    engine_args = EngineArgs.from_cli_args(args)
    if engine_args.enable_lora:
        raise ValueError("Saving with enable_lora=True is not supported!")
    model_path = engine_args.model
    if not Path(model_path).is_dir():
        raise ValueError("model path must be a local directory")
    # Create LLM instance from arguments
    # 从参数创建 LLM 实例
    llm = LLM(**dataclasses.asdict(engine_args))
    # Prepare output directory
    # 准备输出目录
    Path(args.output).mkdir(exist_ok=True)
    # Dump worker states to output directory
    # 转储工作进程状态到输出目录
    model_executor = llm.llm_engine.model_executor
    model_executor.save_sharded_state(path=args.output,
                                      pattern=args.file_pattern,
                                      max_size=args.max_file_size)
    # Copy metadata files to output directory
    # 将元数据文件复制到输出目录
    for file in os.listdir(model_path):
        if os.path.splitext(file)[1] not in (".bin", ".pt", ".safetensors"):
            if os.path.isdir(os.path.join(model_path, file)):
                shutil.copytree(os.path.join(model_path, file),
                                os.path.join(args.output, file))
            else:
                shutil.copy(os.path.join(model_path, file), args.output)


if __name__ == "__main__":
    args = parser.parse_args()
    main(args)

```
