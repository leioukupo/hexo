---
abbrlink: ''
categories: []
date: '2026-03-07T16:30:18.239567+08:00'
tags: []
title: title
updated: '2026-03-07T16:30:41.760+08:00'
---
之前的文章 [逆向api通过外挂解锁特殊功能 | leioukupo的博客](https://blog.leioukupo.top/2025/06/08/%E9%80%86%E5%90%91api%E9%80%9A%E8%BF%87%E5%A4%96%E6%8C%82%E8%A7%A3%E9%94%81%E7%89%B9%E6%AE%8A%E5%8A%9F%E8%83%BD/)
直接构建一个带tool\_calls的completion发送没太大问题，但是不能流式
后续写流式tool调用时一直不行，直到和ai一起看cherry studio的源码关于tool调用这一部分时
才发现我一直套用的chat的流式生成函数，其中的finish\_reason一直是None，所以各个客户端才不响应我的工具请求，让ai根据我之前的chat流式生成器写了一个tool流式生成器

```python
async def generate_tool_call_chunks(
        tool_calls: list,
        answer_id: str,
        created: int,
        model: str,
        system_fingerprint: Optional[str] = None,
        finish_reason: str = "tool_calls"
):
    """
    异步生成器，模拟流式发送工具调用数据
    Args:
        tool_calls: 工具调用列表，每个元素应包含：
                    {
                        "id": "call_xxx",
                        "function": {
                            "name": "function_name",
                            "arguments": '{"param1": "value1"}'  # JSON字符串
                        }
                    }
        answer_id: 回答ID
        created: 创建时间戳
        model: 模型名称
        system_fingerprint: 系统指纹
        finish_reason: 完成原因，默认为 "tool_calls"
    """
    # 第一步：流式发送每个工具调用的初始信息和参数
    for idx, tool_call in enumerate(tool_calls):
        tool_id = tool_call.get("id", f"call_{idx}")
        function_name = tool_call["function"]["name"]
        arguments_str = tool_call["function"]["arguments"]

        # 如果 arguments 是 dict，转换为 JSON 字符串
        if isinstance(arguments_str, dict):
            arguments_str = json.dumps(arguments_str, ensure_ascii=False)

        # 发送工具调用的开始 chunk（包含 id 和 function name）
        chunk_start = {
            "id": answer_id,
            "object": "chat.completion.chunk",
            "created": created,
            "model": model,
            "system_fingerprint": system_fingerprint,
            "choices": [{
                "index": 0,
                "delta": {
                    "tool_calls": [{
                        "index": idx,
                        "id": tool_id,
                        "type": "function",
                        "function": {
                            "name": function_name,
                            "arguments": ""  # 开始为空
                        }
                    }]
                },
                "logprobs": None,
                "finish_reason": None
            }]
        }
        yield f"data: {json.dumps(chunk_start, ensure_ascii=False)}\n\n"
        await asyncio.sleep(random.uniform(0.01, 0.03))

        # 第二步：逐个字符流式发送参数
        for char in arguments_str:
            chunk_arg = {
                "id": answer_id,
                "object": "chat.completion.chunk",
                "created": created,
                "model": model,
                "system_fingerprint": system_fingerprint,
                "choices": [{
                    "index": 0,
                    "delta": {
                        "tool_calls": [{
                            "index": idx,
                            "function": {
                                "arguments": char
                            }
                        }]
                    },
                    "logprobs": None,
                    "finish_reason": None
                }]
            }
            yield f"data: {json.dumps(chunk_arg, ensure_ascii=False)}\n\n"
            await asyncio.sleep(random.uniform(0.005, 0.015))

    # 第三步：发送 finish_reason 为 tool_calls 的完成 chunk
    completion_data = {
        "id": answer_id,
        "object": "chat.completion.chunk",
        "created": created,
        "model": model,
        "system_fingerprint": system_fingerprint,
        "choices": [{
            "index": 0,
            "delta": {},
            "logprobs": None,
            "finish_reason": finish_reason  # "tool_calls"
        }]
    }
    yield f"data: {json.dumps(completion_data, ensure_ascii=False)}\n\n"

    # 第四步：发送流结束标记
    yield "data: [DONE]\n\n"
```

使用这个新的tool流式生成器果然测试成功
使用的是miloco和自带的米家设备控制的mcp测试成功

[![image](https://linux.do/uploads/default/optimized/4X/e/a/c/eac9f74ca1673cf530076377464bea975741f851_2_690x133.png)](https://linux.do/uploads/default/original/4X/e/a/c/eac9f74ca1673cf530076377464bea975741f851.png "image")

重新捋一遍2api要模拟fc以及mcp，各位佬友可以把它当作一个模板

```python
# 首先需要一些必须的全局变量标识状态和一个异步的定时器(长时间得不到tool响应，就重置tools_use)
tools_use = False  # 标识有没有使用过工具，true标识已经调用过工具了
tools_answer = 'no' # 这是单独对tools和问题进行判断的回答， no说明不需要调用工具，对其进行处理

question = None # 不一定需要，最好有一个，方便全局标记用户的问题

async def start_timer():
    """启动定时器"""
    global tools_use
    await asyncio.sleep(30)
    if tools_use:  # 60秒后如果还是True
        tools_use = False
        logger.debug("tools_use 超时未重置，已自动重置为 False")

# 首先要接收存储请求中的tools和tool_choice  根据tool_choice进行处理
tools_json = data.get("tools", None)
tool_choice = data.get("tool_choice", 'Auto')
# 工具选择 我根据openai api文档总结的
    # 默认情况下，模型将决定何时以及使用多少工具。您可以使用 tool_choice 参数强制特定行为。
    #
    # 自动：（默认）调用零个、一个或多个函数。 tool_choice: "auto"
    # 需要：调用一个或多个函数。 tool_choice: "required"
    # 强制函数：精确调用一个特定函数。 tool_choice: {"type": "function", "name": "get_weather"}
    # 允许的工具：将模型可以调用的工具限制为模型可用工具的子集。
    # "tool_choice": {
    #     "type": "allowed_tools",
    #     "mode": "auto",
    #     "tools": [
    #         {"type": "function", "name": "get_weather"},
    #         {"type": "function", "name": "search_docs"}
    #     ]
    # }
    # }
    # None不传递函数
tools_prompt=xxxxx  #还是之前的系统提示词，这提示词偶尔给的json不正常,
                    # 不知道是我的模型问题还是参数或者提示词本身问题
if tools_use is False and tools_json is not None:
    #只有还没调用tool或者有tool情况下才处理
    finish_reason = "tool_calls" # tool的completion里的finish_reason一定要是这个
        if isinstance(tool_choice, dict):
            # 根据tool_choice的不同分别处理
            # 设置tools_json里的工具
            logger.debug('tool_choice is force or allowed_tools')
            if "type" in tool_choice:
                if tool_choice["type"] == "function":
                    # logger.debug('tool_choice is force')
                    force_tools_json = tool_choice
                    force_tools_dict = [force_tools_json]
                    # 强制调用，先调用存起来
                    question = messages[-1]["content"]
                    tools_question = str(force_tools_dict) + messages[-1]["content"]
                    tools_answer, prompt_tokens, completion_tokens, total_tokens, answer_id, system_fingerprint = await tools_chat(
                        tools_prompt, tools_question) # tools_chat根据自己的2api去写,就是用前面的提示词加tool和问题的组合去别的模型
                    if tools_answer != 'no':
                        logger.debug(f"tool_answer {tools_answer}")
                        tools_answer = re.sub(r"^\`\`\`json|^\`\`\`", "", tools_answer) # 偶尔出错,所以我加个正则增强稳定性
                        tool_call = json.loads(tools_answer)
                        tools_list.append(tool_call[0])
                elif tool_choice["type"] == "allowed_tools":
                    logger.debug('tool_choice is allowed_tools')
                    tools_json = tool_choice.get("tools", [])
                else:
                    logger.debug('tool_choice is auto or required')
            else:
                logger.debug(f'tool_choice is {tool_choice}')

# 这样写，会把所有的tool都拿去问，比较浪费时间，后续继续优化，做到和官网api一样按需调用模型
        # 接着处理tool_choice为auto或者require的情况，两者差别很小，更细分等后续优化
        # 转换为OpenAI标准格式  注意一定要是这样的标准格式，前面的提示词生成的是claude风格，不太会改，就写了个转换的
          openai_tool = {
              "id": f"call_{call_id}",
              "type": "function",
              "function": {
                  "name": item.get("name", ""),
                  "arguments": item.get("arguments", "{}")
              }
          }
        tools_list = openai_tools_list
        # 然后发送给客户端，不管是流式还是非流，一定记得finish_reason要是tool_calls
        completion = {
            "id": f"{answer_id}",
            "object": "chat.completion",
            "created": created,
            "model": model,
            "system_fingerprint": system_fingerprint,
            "choices": [{
                "index": 0,
                "message": {
                    "role": "assistant",
                    "tool_calls": tools_list
                },
                "logprobs": None,
                "finish_reason": finish_reason
            }],
            "usage": {
                "prompt_tokens": prompt_tokens,
                "completion_tokens": completion_tokens,
                "total_tokens": total_tokens
            }
        }
# 记得重置tools_use的状态为下一次的tool调用做准备
# 第二次请求的是tools_use是真了
if tools_use is True:
    # 把tool的响应和用户的问题一起打包去问大模型，处理工具响应的时候要注意可能会是文件  图片 音频以及文字这几种
# 最后一种就是和工具没有一点关系的纯文字chat了
# 整体代码结构就是
if tools_use is False and tools_json is not None:
if tools_use is True:
else:
```

目前的话，还是有几个点需要去进一步优化
1.当tool的数量很多的时候全部调用一次时间会很长
需要优化，提前判断要调用哪些tool， 这需要提示词专家重新编写一个提示词让大模型去判断并返回tool\_json
2.还有参数不够的情况, 怎么去处理
演示里有一种情况就是一个tool需要三个参数，第一次只给了一个参数，然后需要大模型去问用户其他的参数，这个暂时没想到解决办法去模拟
3.工具的返回值可能是图片 文件 音频或者文字
目前只考虑了文字，因为其他的我不知客户端发过来的内容是啥样的，图片可能是base64，但文件或者音频就不知道了
首要的就是tools\_prompt的改进
