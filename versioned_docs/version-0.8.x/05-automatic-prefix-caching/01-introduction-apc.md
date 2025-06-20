---
title: 简介
---

## 什么是自动前缀缓存 (Automatic Prefix Caching)

自动前缀缓存（Automatic Prefix Caching，简称 APC）会缓存现有查询的 KV 缓存，这样以来，如果新查询与现有查询之一共享相同的前缀，就可以直接重用 KV 缓存，从而允许新查询跳过现有查询的计算共享部分。

**注意：**

有关 vLLM 如何实现 APC 的技术详细信息将在下一页介绍。

## 在 vLLM 中启用 APC

如需启用 APC，请在 vLLM 引擎中设置 `enable_prefix_caching=True` 。下面是一个例子：

```python
import time
from vllm import LLM, SamplingParams




# A prompt containing a large markdown table. The table is randomly generated by GPT-4.
# 一个包含大型 Markdown 表格的提示。该表由 GPT-4 随机生成。


LONG_PROMPT = "You are a helpful assistant in recognizes the content of tables in markdown format. Here is a table as follows.\n# Table\n" + """
LONG_PROMPT = "你是识别Markdown格式表格内容的得力助手，表格如下。\n# Table\n" + """


| ID  | Name          | Age | Occupation    | Country       | Email                  | Phone Number   | Address                       |
|-----|---------------|-----|---------------|---------------|------------------------|----------------|------------------------------|
| 1   | John Doe      | 29  | Engineer      | USA           | john.doe@example.com   | 555-1234       | 123 Elm St, Springfield, IL  |
| 2   | Jane Smith    | 34  | Doctor        | Canada        | jane.smith@example.com | 555-5678       | 456 Oak St, Toronto, ON      |
| 3   | Alice Johnson | 27  | Teacher       | UK            | alice.j@example.com    | 555-8765       | 789 Pine St, London, UK      |
| 4   | Bob Brown     | 45  | Artist        | Australia     | bob.b@example.com      | 555-4321       | 321 Maple St, Sydney, NSW    |
| 5   | Carol White   | 31  | Scientist     | New Zealand   | carol.w@example.com    | 555-6789       | 654 Birch St, Wellington, NZ |
| 6   | Dave Green    | 28  | Lawyer        | Ireland       | dave.g@example.com     | 555-3456       | 987 Cedar St, Dublin, IE     |
| 7   | Emma Black    | 40  | Musician      | USA           | emma.b@example.com     | 555-1111       | 246 Ash St, New York, NY     |
| 8   | Frank Blue    | 37  | Chef          | Canada        | frank.b@example.com    | 555-2222       | 135 Spruce St, Vancouver, BC |
| 9   | Grace Yellow  | 50  | Engineer      | UK            | grace.y@example.com    | 555-3333       | 864 Fir St, Manchester, UK   |
| 10  | Henry Violet  | 32  | Artist        | Australia     | henry.v@example.com    | 555-4444       | 753 Willow St, Melbourne, VIC|
| 11  | Irene Orange  | 26  | Scientist     | New Zealand   | irene.o@example.com    | 555-5555       | 912 Poplar St, Auckland, NZ  |
| 12  | Jack Indigo   | 38  | Teacher       | Ireland       | jack.i@example.com     | 555-6666       | 159 Elm St, Cork, IE         |
| 13  | Karen Red     | 41  | Lawyer        | USA           | karen.r@example.com    | 555-7777       | 357 Cedar St, Boston, MA     |
| 14  | Leo Brown     | 30  | Chef          | Canada        | leo.b@example.com      | 555-8888       | 246 Oak St, Calgary, AB      |
| 15  | Mia Green     | 33  | Musician      | UK            | mia.g@example.com      | 555-9999       | 975 Pine St, Edinburgh, UK   |
| 16  | Noah Yellow   | 29  | Doctor        | Australia     | noah.y@example.com     | 555-0000       | 864 Birch St, Brisbane, QLD  |
| 17  | Olivia Blue   | 35  | Engineer      | New Zealand   | olivia.b@example.com   | 555-1212       | 753 Maple St, Hamilton, NZ   |
| 18  | Peter Black   | 42  | Artist        | Ireland       | peter.b@example.com    | 555-3434       | 912 Fir St, Limerick, IE     |
| 19  | Quinn White   | 28  | Scientist     | USA           | quinn.w@example.com    | 555-5656       | 159 Willow St, Seattle, WA   |
| 20  | Rachel Red    | 31  | Teacher       | Canada        | rachel.r@example.com   | 555-7878       | 357 Poplar St, Ottawa, ON    |
| 21  | Steve Green   | 44  | Lawyer        | UK            | steve.g@example.com    | 555-9090       | 753 Elm St, Birmingham, UK   |
| 22  | Tina Blue     | 36  | Musician      | Australia     | tina.b@example.com     | 555-1213       | 864 Cedar St, Perth, WA      |
| 23  | Umar Black    | 39  | Chef          | New Zealand   | umar.b@example.com     | 555-3435       | 975 Spruce St, Christchurch, NZ|
| 24  | Victor Yellow | 43  | Engineer      | Ireland       | victor.y@example.com   | 555-5657       | 246 Willow St, Galway, IE    |
| 25  | Wendy Orange  | 27  | Artist        | USA           | wendy.o@example.com    | 555-7879       | 135 Elm St, Denver, CO       |
| 26  | Xavier Green  | 34  | Scientist     | Canada        | xavier.g@example.com   | 555-9091       | 357 Oak St, Montreal, QC     |
| 27  | Yara Red      | 41  | Teacher       | UK            | yara.r@example.com     | 555-1214       | 975 Pine St, Leeds, UK       |
| 28  | Zack Blue     | 30  | Lawyer        | Australia     | zack.b@example.com     | 555-3436       | 135 Birch St, Adelaide, SA   |
| 29  | Amy White     | 33  | Musician      | New Zealand   | amy.w@example.com      | 555-5658       | 159 Maple St, Wellington, NZ |
| 30  | Ben Black     | 38  | Chef          | Ireland       | ben.b@example.com      | 555-7870       | 246 Fir St, Waterford, IE    |
"""




def get_generation_time(llm, sampling_params, prompts):
    # time the generation
    # 生成时间


    start_time = time.time()
    output = llm.generate(prompts, sampling_params=sampling_params)
    end_time = time.time()
    # print the output and generation time
    # 打印输出和生成时间


    print(f"Output: {output[0].outputs[0].text}")
    print(f"Generation time: {end_time - start_time} seconds.")




# set enable_prefix_caching=True to enable APC
# 设置 enable_prefix_caching=True 启用APC


llm = LLM(
    model='lmsys/longchat-13b-16k',
    enable_prefix_caching=True
)


sampling_params = SamplingParams(temperature=0, max_tokens=100)


# Querying the age of John Doe
# 查询John Doe的年龄


get_generation_time(
    llm,
    sampling_params,
    LONG_PROMPT + "Question: what is the age of John Doe? Your answer: The age of John Doe is ",
)


# Querying the age of Zack Blue
# 查询Zack Blue的年龄


# This query will be faster since vllm avoids computing the KV cache of LONG_PROMPT again.
# 这个查询会更快，因为 vllm 避免了再次计算 LONG_PROMPT 的 KV 缓存。


get_generation_time(
    llm,
    sampling_params,
    LONG_PROMPT + "Question: what is the age of Zack Blue? Your answer: The age of Zack Blue is ",
)
```

## 工作负载示例

我们描述了 2 个示例工作负载，其中 APC 可以提供巨大的性能提升：

- 长文档查询，用户多次使用不同的查询内容对同一长文档（例如软件手册或年度报告）进行查询。在这种情况下，APC 允许 vLLM *仅处理一次*这个长文档，而不是一次又一次地处理这个长文档，并且所有后续请求都可以通过重用其 KV 缓存来避免重新计算这个长文档。这使得 vLLM 能够以更高的吞吐量和更低的延迟来服务未来的请求。
- 多轮对话，用户可以在同一个聊天会话中与应用程序多次进行对话。在这种情况下，APC 允许 vLLM 在所有后续的对话轮次中重用聊天历史的处理结果，而无需一次又一次地处理整个聊天历史。从而允许 vLLM 以更高的吞吐量和更低的延迟来服务未来的请求。

## 局限性

通常情况下，APC 一般不会降低 vLLM 的性能。话虽如此，APC 只是减少了处理查询的时间 （预填充阶段），并没有减少生成新 token 所需的时间 （预填充阶段）。因此，当 vLLM 花费大部分时间生成查询答案时（例如，当答案的长度较长时），或者新查询不与任何现有查询共享相同的前缀（以致于计算不能重复使用），APC 并不会带来性能提升。
