---
title: OpenAI 聊天补全客户端
---

源代码: [vllm-project/vllm](https://raw.githubusercontent.com/vllm-project/vllm/main/examples/openai_chat_completion_client.py)

```python
from openai import OpenAI

# Modify OpenAI's API key and API base to use vLLM's API server.
# 设置 OpenAI 的 API key，和 API base 以使用 vLLM's API 服务

openai_api_key = "EMPTY"
openai_api_base = "http://localhost:8000/v1"

client = OpenAI(
    # defaults to os.environ.get("OPENAI_API_KEY")
    # 默认为 defaults to os.environ.get("OPENAI_API_KEY")

    api_key=openai_api_key,
    base_url=openai_api_base,
)

models = client.models.list()
model = models.data[0].id

chat_completion = client.chat.completions.create(
    messages=[{
        "role": "system",
        "content": "You are a helpful assistant."
    }, {
        "role": "user",
        "content": "Who won the world series in 2020?"
    }, {
        "role":
        "assistant",
        "content":
        "The Los Angeles Dodgers won the World Series in 2020."
    }, {
        "role": "user",
        "content": "Where was it played?"
    }],
    model=model,
)

print("Chat completion results:")
print(chat_completion)
```
