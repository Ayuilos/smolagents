<!--Copyright 2024 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.

-->
# 构建好用的agent

[[open-in-colab]]

能良好工作的agent和不能工作的agent之间，有天壤之别。
我们怎么样才能构建出属于前者的agent呢？
在本指南中，我们将看到构建agent的最佳实践。

> [!TIP]
> 如果你是agent构建的新手，请确保首先阅读[agent介绍](../conceptual_guides/intro_agents)和[smolagents导览](../guided_tour)。

### 最好的agent系统是最简单的：尽可能简化工作流

在你的工作流中赋予LLM一些自主权，会引入一些错误风险。

经过良好编程的agent系统，通常具有良好的错误日志记录和重试机制，因此LLM引擎有机会自我纠错。但为了最大限度地降低LLM错误的风险，你应该简化你的工作流！

让我们回顾一下[agent介绍](../conceptual_guides/intro_agents)中的例子：一个为冲浪旅行公司回答用户咨询的机器人。
与其让agent每次被问及新的冲浪地点时，都分别调用"旅行距离API"和"天气API"，你可以只创建一个统一的工具"return_spot_information"，一个同时调用这两个API，并返回它们连接输出的函数。

这可以降低成本、延迟和错误风险！

主要的指导原则是：尽可能减少LLM调用的次数。

这可以带来一些启发：
- 尽可能把两个工具合并为一个，就像我们两个API的例子。
- 尽可能基于确定性函数，而不是agent决策，来实现逻辑。

### 改善流向LLM引擎的信息流

记住，你的LLM引擎就像一个~智能~机器人，被关在一个房间里，与外界唯一的交流方式是通过门缝传递的纸条。

如果你没有明确地将信息放入其提示中，它将不知道发生的任何事情。

所以首先要让你的任务非常清晰！
由于agent由LLM驱动，任务表述的微小变化可能会产生完全不同的结果。

然后，改善工具使用中流向agent的信息流。

需要遵循的具体指南：
- 每个工具都应该记录（只需在工具的`forward`方法中使用`print`语句）对LLM引擎可能有用的所有信息。
  - 特别是，记录工具执行错误的详细信息会很有帮助！

例如，这里有一个根据位置和日期时间检索天气数据的工具：

首先，这是一个糟糕的版本：
```python
import datetime
from smolagents import tool

def get_weather_report_at_coordinates(coordinates, date_time):
    # 虚拟函数，返回[温度（°C），降雨风险（0-1），浪高（m）]
    return [28.0, 0.35, 0.85]

def get_coordinates_from_location(location):
    # 返回虚拟坐标
    return [3.3, -42.0]

@tool
def get_weather_api(location: str, date_time: str) -> str:
    """
    Returns the weather report.

    Args:
        location: the name of the place that you want the weather for.
        date_time: the date and time for which you want the report.
    """
    lon, lat = convert_location_to_coordinates(location)
    date_time = datetime.strptime(date_time)
    return str(get_weather_report_at_coordinates((lon, lat), date_time))
```

为什么它不好？
- 没有说明`date_time`应该使用的格式
- 没有说明位置应该如何指定
- 没有记录机制来处理明确的报错情况，如位置格式不正确或date_time格式不正确
- 输出格式难以理解

如果工具调用失败，内存中记录的错误跟踪，可以帮助LLM逆向工程工具来修复错误。但为什么要让它做这么多繁重的工作呢？

构建这个工具的更好方式如下：
```python
@tool
def get_weather_api(location: str, date_time: str) -> str:
    """
    Returns the weather report.

    Args:
        location: the name of the place that you want the weather for. Should be a place name, followed by possibly a city name, then a country, like "Anchor Point, Taghazout, Morocco".
        date_time: the date and time for which you want the report, formatted as '%m/%d/%y %H:%M:%S'.
    """
    lon, lat = convert_location_to_coordinates(location)
    try:
        date_time = datetime.strptime(date_time)
    except Exception as e:
        raise ValueError("Conversion of `date_time` to datetime format failed, make sure to provide a string in format '%m/%d/%y %H:%M:%S'. Full trace:" + str(e))
    temperature_celsius, risk_of_rain, wave_height = get_weather_report_at_coordinates((lon, lat), date_time)
    return f"Weather report for {location}, {date_time}: Temperature will be {temperature_celsius}°C, risk of rain is {risk_of_rain*100:.0f}%, wave height is {wave_height}m."
```

一般来说，为了减轻LLM的负担，要问自己的好问题是："如果我是一个第一次使用这个工具的傻瓜，使用这个工具编程并纠正自己的错误有多容易？"。

### 给agent更多参数

除了简单的任务描述字符串外，你还可以使用`additional_args`参数传递任何类型的对象：

```py
from smolagents import CodeAgent, HfApiModel

model_id = "meta-llama/Llama-3.3-70B-Instruct"

agent = CodeAgent(tools=[], model=HfApiModel(model_id=model_id), add_base_tools=True)

agent.run(
    "Why does Mike not know many people in New York?",
    additional_args={"mp3_sound_file_url":'https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/recording.mp3'}
)
```
例如，你可以使用这个`additional_args`参数传递你希望agent利用的图像或字符串。


## 如何调试你的agent

### 1. 使用更强大的LLM

在agent工作流中，有些错误是实际错误，有些则是你的LLM引擎没有正确推理的结果。
例如，参考这个我要求创建一个汽车图片的`CodeAgent`的运行记录：
```text
==================================================================================================== New task ====================================================================================================
Make me a cool car picture
──────────────────────────────────────────────────────────────────────────────────────────────────── New step ────────────────────────────────────────────────────────────────────────────────────────────────────
Agent is executing the code below: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
image_generator(prompt="A cool, futuristic sports car with LED headlights, aerodynamic design, and vibrant color, high-res, photorealistic")
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Last output from code snippet: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png
Step 1:

- Time taken: 16.35 seconds
- Input tokens: 1,383
- Output tokens: 77
──────────────────────────────────────────────────────────────────────────────────────────────────── New step ────────────────────────────────────────────────────────────────────────────────────────────────────
Agent is executing the code below: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
final_answer("/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png")
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
Print outputs:

Last output from code snippet: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png
Final answer:
/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png
```
用户看到的是返回了一个路径，而不是图像。
这看起来像是系统的错误，但实际上agent系统并没有导致错误：只是LLM大脑犯了一个错误，没有把图像输出，保存到变量中。
因此，它无法再次访问图像，只能利用保存图像时记录的路径，所以它返回的是路径，而不是图像。

调试agent的第一步是"使用更强大的LLM"。像`Qwen2.5-72B-Instruct`这样的替代方案不会犯这种错误。

### 2. 提供更多指导/更多信息

你也可以使用不太强大的模型，只要你更有效地指导它们。

站在模型的角度思考：如果你是模型在解决任务，你会因为系统提示+任务表述+工具描述中提供的信息而挣扎吗？

你需要一些额外的说明吗？

为了提供额外信息，我们不建议立即更改系统提示：默认系统提示有许多调整，除非你非常了解提示，否则你很容易翻车。
更好的指导LLM引擎的方法是：
- 如果是关于要解决的任务：把所有细节添加到任务中。任务可以有几百页长。
- 如果是关于如何使用工具：你的工具的description属性。


### 3. 更改系统提示（通常不建议）

如果上述说明不够，你可以更改系统提示。

让我们看看它是如何工作的。例如，让我们检查[`CodeAgent`]的默认系统提示（下面的版本通过跳过零样本示例进行了缩短）。

```python
print(agent.system_prompt_template)
```
你会得到：
```text
You are an expert assistant who can solve any task using code blobs. You will be given a task to solve as best you can.
To do so, you have been given access to a list of tools: these tools are basically Python functions which you can call with code.
To solve the task, you must plan forward to proceed in a series of steps, in a cycle of 'Thought:', 'Code:', and 'Observation:' sequences.

At each step, in the 'Thought:' sequence, you should first explain your reasoning towards solving the task and the tools that you want to use.
Then in the 'Code:' sequence, you should write the code in simple Python. The code sequence must end with '<end_code>' sequence.
During each intermediate step, you can use 'print()' to save whatever important information you will then need.
These print outputs will then appear in the 'Observation:' field, which will be available as input for the next step.
In the end you have to return a final answer using the `final_answer` tool.

Here are a few examples using notional tools:
---
{examples}

Above example were using notional tools that might not exist for you. On top of performing computations in the Python code snippets that you create, you only have access to these tools:

{{tool_descriptions}}

{{managed_agents_descriptions}}

Here are the rules you should always follow to solve your task:
1. Always provide a 'Thought:' sequence, and a 'Code:\n```py' sequence ending with '```<end_code>' sequence, else you will fail.
2. Use only variables that you have defined!
3. Always use the right arguments for the tools. DO NOT pass the arguments as a dict as in 'answer = wiki({'query': "What is the place where James Bond lives?"})', but use the arguments directly as in 'answer = wiki(query="What is the place where James Bond lives?")'.
4. Take care to not chain too many sequential tool calls in the same code block, especially when the output format is unpredictable. For instance, a call to search has an unpredictable return format, so do not have another tool call that depends on its output in the same block: rather output results with print() to use them in the next block.
5. Call a tool only when needed, and never re-do a tool call that you previously did with the exact same parameters.
6. Don't name any new variable with the same name as a tool: for instance don't name a variable 'final_answer'.
7. Never create any notional variables in our code, as having these in your logs might derail you from the true variables.
8. You can use imports in your code, but only from the following list of modules: {{authorized_imports}}
9. The state persists between code executions: so if in one step you've created variables or imported modules, these will all persist.
10. Don't give up! You're in charge of solving the task, not providing directions to solve it.

Now Begin! If you solve the task correctly, you will receive a reward of $1,000,000.
```

如你所见，有一些占位符，如`"{{tool_descriptions}}"`：这些将在agent初始化时用于插入某些自动生成的工具或管理agent的描述。

因此，虽然你可以通过将自定义提示作为参数传递给`system_prompt`参数来覆盖此系统提示模板，但你的新系统提示必须包含以下占位符：
- `"{{tool_descriptions}}"` 用于插入工具描述。
- `"{{managed_agents_description}}"` 用于插入managed agent的描述（如果有）。
- 仅限`CodeAgent`：`"{{authorized_imports}}"` 用于插入授权导入列表。

然后你可以根据如下，更改系统提示：

```py
from smolagents.prompts import CODE_SYSTEM_PROMPT

modified_system_prompt = CODE_SYSTEM_PROMPT + "\nHere you go!" # 在此更改系统提示

agent = CodeAgent(
    tools=[], 
    model=HfApiModel(), 
    system_prompt=modified_system_prompt
)
```

这也适用于[`ToolCallingAgent`]。


### 4. 额外规划

我们提供了一个用于补充规划步骤的模型，agent可以在正常操作步骤之间定期运行。在此步骤中，没有工具调用，LLM只是被要求更新它知道的事实列表，并根据这些事实反推它应该采取的下一步。

```py
from smolagents import load_tool, CodeAgent, HfApiModel, DuckDuckGoSearchTool
from dotenv import load_dotenv

load_dotenv()

# 从Hub导入工具
image_generation_tool = load_tool("m-ric/text-to-image", trust_remote_code=True)

search_tool = DuckDuckGoSearchTool()

agent = CodeAgent(
    tools=[search_tool],
    model=HfApiModel("Qwen/Qwen2.5-72B-Instruct"),
    planning_interval=3 # 这是你激活规划的地方！
)

# 运行它！
result = agent.run(
    "How long would a cheetah at full speed take to run the length of Pont Alexandre III?",
)
```