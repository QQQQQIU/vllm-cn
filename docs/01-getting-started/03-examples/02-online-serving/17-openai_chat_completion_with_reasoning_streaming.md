---
title: 基于推理流的 OpenAI 聊天完成
---

[\*在线运行 vLLM 入门教程：零基础分步指南](https://openbayes.com/console/public/tutorials/rXxb5fZFr29?utm_source=vLLM-CNdoc&utm_medium=vLLM-CNdoc-V1&utm_campaign=vLLM-CNdoc-V1-25ap)

源码 [examples/online_serving/openai_chat_completion_with_reasoning_streaming.py](https://github.com/vllm-project/vllm/blob/main/examples/online_serving/openai_chat_completion_with_reasoning_streaming.py)

````python
# SPDX-License-Identifier: Apache-2.0
"""
An example shows how to generate chat completions from reasoning models
like DeepSeekR1.

To run this example, you need to start the vLLM server with the reasoning
parser:

```bash
vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B \
--enable-reasoning --reasoning-parser deepseek_r1
````

Unlike openai_chat_completion_with_reasoning.py, this example demonstrates the
streaming chat completions feature.

The streaming chat completions feature allows you to receive chat completions
in real-time as they are generated by the model. This is useful for scenarios
where you want to display chat completions to the user as they are generated
by the model.

Remember to check content and reasoning_content exist in `ChatCompletionChunk`,
content may not exist leading to errors if you try to access it.
"""
"""
本示例演示如何从推理模型如 DeepSeekR1 生成聊天完成。

要运行此示例，您需要启动 vLLM 服务器，并启用推理解析器：

```bash
vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B \
--enable-reasoning --reasoning-parser deepseek_r1
与 openai_chat_completion_with_reasoning.py 不同，本示例演示了流式聊天完成特性。
```

流式聊天完成特性允许您在模型生成聊天内容时，实时接收聊天完成。这对于您希望在用户界面上实时显示模型生成的聊天内容的场景非常有用。

记住，检查 ChatCompletionChunk 中是否存在 content 和 reasoning_content，content 可能不存在，如果您尝试访问它会导致错误。
"""

from openai import OpenAI

# Modify OpenAI's API key and API base to use vLLM's API server.

# 修改 OpenAI 的 API key 和 API base, 使用 vLLM 的 API 服务器。

openai_api_key = "EMPTY"
openai_api_base = "http://localhost:8000/v1"

client = OpenAI(
api_key=openai_api_key,
base_url=openai_api_base,
)

models = client.models.list()
model = models.data[0].id

messages = [{"role": "user", "content": "9.11 and 9.8, which is greater?"}]

# For granite, add: `extra_body={"chat_template_kwargs": {"thinking": True}}`

# 对于 granite，添加:`extra_body = { "chat_template_kwargs":{ "thinky":true}}'

stream = client.chat.completions.create(model=model,
messages=messages,
stream=True)

print("client: Start streaming chat completions...")
printed_reasoning_content = False
printed_content = False

for chunk in stream:
reasoning_content = None
content = None # Check the content is reasoning_content or content # 检查 content 是 chinconing_content 还是 content
if hasattr(chunk.choices[0].delta, "reasoning_content"):
reasoning_content = chunk.choices[0].delta.reasoning_content
elif hasattr(chunk.choices[0].delta, "content"):
content = chunk.choices[0].delta.content

    if reasoning_content is not None:
        if not printed_reasoning_content:
            printed_reasoning_content = True
            print("reasoning_content:", end="", flush=True)
        print(reasoning_content, end="", flush=True)
    elif content is not None:
        if not printed_content:
            printed_content = True
            print("\ncontent:", end="", flush=True)
        # Extract and print the content
        # 提取并打印内容
        print(content, end="", flush=True)

```

```
