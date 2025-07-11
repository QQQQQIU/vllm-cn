---
title: 带前缀的离线推理
---

源代码: [vllm-project/vllm](https://raw.githubusercontent.com/vllm-project/vllm/main/examples/offline_inference_with_prefix.py)

```python
from time import time

from vllm import LLM, SamplingParams

# Common prefix.
# 通用前缀

prefix = (
    "You are an expert school principal, skilled in effectively managing "
    "faculty and staff. Draft 10-15 questions for a potential first grade "
    "Head Teacher for my K-12, all-girls', independent school that emphasizes "
    "community, joyful discovery, and life-long learning. The candidate is "
    "coming in for a first-round panel interview for a 8th grade Math "
    "teaching role. They have 5 years of previous teaching experience "
    "as an assistant teacher at a co-ed, public school with experience "
    "in middle school math teaching. Based on these information, fulfill "
    "the following paragraph: ")

# Sample prompts.
# 提示示例

prompts = [
    "Hello, my name is",
    "The president of the United States is",
    "The capital of France is",
    "The future of AI is",
]

generating_prompts = [prefix + prompt for prompt in prompts]

# Create a sampling params object.
# 创建 sampling params 对象
sampling_params = SamplingParams(temperature=0.0)

# Create an LLM.
# 创建一个 LLM
regular_llm = LLM(model="facebook/opt-125m", gpu_memory_utilization=0.4)

prefix_cached_llm = LLM(model="facebook/opt-125m",
                        enable_prefix_caching=True,
                        gpu_memory_utilization=0.4)
print("Results without `enable_prefix_caching`")

# Generate texts from the prompts. The output is a list of RequestOutput objects
# that contain the prompt, generated text, and other information.
# 从提示中生成文本。输出是一个 RequestOutput 列表，包含提示、生成文本和其他信息

start_time_regular = time()
outputs = regular_llm.generate(generating_prompts, sampling_params)
duration_regular = time() - start_time_regular

regular_generated_texts = []
# Print the outputs.
# 打印输出
for output in outputs:
    prompt = output.prompt
    generated_text = output.outputs[0].text
    regular_generated_texts.append(generated_text)
    print(f"Prompt: {prompt!r}, Generated text: {generated_text!r}")

print("-" * 80)

# Warmup so that the shared prompt's KV cache is computed.
# 预热，以便计算共享提示的 KV 缓存。
prefix_cached_llm.generate(generating_prompts[0], sampling_params)

# Generate with prefix caching.
# 使用前缀缓存生成
start_time_cached = time()
outputs = prefix_cached_llm.generate(generating_prompts, sampling_params)
duration_cached = time() - start_time_cached

print("Results with `enable_prefix_caching`")

cached_generated_texts = []
# Print the outputs. You should see the same outputs as before.
# 打印输出。你应该会看到与之前相同的输出
for output in outputs:
    prompt = output.prompt
    generated_text = output.outputs[0].text
    cached_generated_texts.append(generated_text)
    print(f"Prompt: {prompt!r}, Generated text: {generated_text!r}")

print("-" * 80)

# Compare the results and display the speedup
# 比较结果并显示加速情况
generated_same = all([
    regular_generated_texts[i] == cached_generated_texts[i]
    for i in range(len(prompts))
])
print(f"Generated answers are the same: {generated_same}")

speedup = round(duration_regular / duration_cached, 2)
print(f"Speed up of cached generation compared to the regular is: {speedup}")
```
