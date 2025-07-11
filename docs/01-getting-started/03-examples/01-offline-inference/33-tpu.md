---
title: Tpu
---

[\*在线运行 vLLM 入门教程：零基础分步指南](https://openbayes.com/console/public/tutorials/rXxb5fZFr29?utm_source=vLLM-CNdoc&utm_medium=vLLM-CNdoc-V1&utm_campaign=vLLM-CNdoc-V1-25ap)

源码 [examples/offline_inference/tpu.py](https://github.com/vllm-project/vllm/blob/main/examples/offline_inference/tpu.py)

```python
# SPDX-License-Identifier: Apache-2.0

from vllm import LLM, SamplingParams

prompts = [
    "A robot may not injure a human being",
    "It is only with the heart that one can see rightly;",
    "The greatest glory in living lies not in never falling,",
]
answers = [
    " or, through inaction, allow a human being to come to harm.",
    " what is essential is invisible to the eye.",
    " but in rising every time we fall.",
]
N = 1
# Currently, top-p sampling is disabled. `top_p` should be 1.0.
# 当前，TOP-P 采样被禁用。 `top_p` 应为 1.0。
sampling_params = SamplingParams(temperature=0, top_p=1.0, n=N, max_tokens=16)

# Set `enforce_eager=True` to avoid ahead-of-time compilation.
# In real workloads, `enforace_eager` should be `False`.
# 设置 `enforce_eager = true`，避免提前汇编。
# 在实际的工作负载中，`enforace_eager` 应该是 False'。
llm = LLM(model="Qwen/Qwen2-1.5B-Instruct",
          max_num_batched_tokens=64,
          max_num_seqs=4)
outputs = llm.generate(prompts, sampling_params)
for output, answer in zip(outputs, answers):
    prompt = output.prompt
    generated_text = output.outputs[0].text
    print(f"Prompt: {prompt!r}, Generated text: {generated_text!r}")
    assert generated_text.startswith(answer)

```
