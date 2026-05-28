# Dify Agent 模块 — 架构总览

## 1. 核心业务定位

`api/core/agent/` 模块是 Dify 平台的**智能体（Agent）运行时引擎**。

它解决的核心问题是：**让大语言模型（LLM）能够自主推理、规划，并通过调用外部工具来完成复杂任务**。

具体而言，该模块实现了两种主流的 Agent 策略范式：
- **Function Calling**：利用 LLM 原生的函数调用能力（如 OpenAI 的 tool_calls），让模型直接输出结构化的工具调用请求
- **Chain-of-Thought / ReAct**：通过精心设计的 Prompt 模板，引导 LLM 按照 `Thought -> Action -> Observation` 的循环进行推理和行动

此外，该模块还提供了**插件化策略扩展机制**，允许第三方插件注册自定义的 Agent 策略。

---

## 2. 技术栈 / 依赖

| 维度 | 技术 |
|---|---|
| 语言 | Python 3.10+（使用 `str \| None`、`list[str]` 等现代类型语法） |
| 数据验证 | Pydantic v2（BaseModel、Field、field_validator） |
| 数据库 | SQLAlchemy 2.0（Mapped、mapped_column、select） |
| LLM 运行时 | `graphon.model_runtime`（ModelInstance、PromptMessage 体系） |
| 工具系统 | `core.tools`（ToolManager、ToolEngine、DatasetRetrieverTool） |
| 队列系统 | `core.app.apps.base_app_queue_manager`（AppQueueManager） |
| 记忆系统 | `core.memory.token_buffer_memory`（TokenBufferMemory） |
| 设计模式 | 模板方法模式、策略模式 |

---

## 3. 文件清单 + 每个文件的独立功能

```
api/core/agent/
├── __init__.py                    # 空文件，模块入口
├── entities.py                    # 核心数据实体：AgentEntity、AgentScratchpadUnit、AgentToolEntity 等
├── errors.py                      # 异常定义：AgentMaxIterationError
├── base_agent_runner.py           # Agent 运行基类：工具初始化、思维记录、历史对话组织
├── fc_agent_runner.py             # Function Calling 策略：利用 LLM 原生 tool_calls 的循环执行
├── cot_agent_runner.py            # CoT/ReAct 策略基类：Thought/Action/Observation 循环
├── cot_chat_agent_runner.py       # CoT Chat 模式：System + User + Assistant 多消息结构
├── cot_completion_agent_runner.py # CoT Completion 模式：单条 UserPromptMessage 拼接
├── plugin_entities.py             # 插件策略实体：AgentStrategyEntity、AgentStrategyParameter 等
├── output_parser/
│   └── cot_output_parser.py       # ReAct 输出流解析器：逐字符解析 Thought/Action/JSON
├── prompt/
│   └── template.py                # ReAct Prompt 模板：Chat 和 Completion 两种模板
└── strategy/
    ├── base.py                    # 策略抽象基类：BaseAgentStrategy
    └── plugin.py                  # 插件策略实现：PluginAgentStrategy
```

---

## 4. 模块整体调用流程

```
外部调用方（agent_chat app）
        |
        v
  BaseAgentRunner.__init__()     <-- 初始化上下文、工具、历史、模型特性检测
        |
        +-- 策略选择 ------------------------------------------------+
        |                                                           |
        v                                                           v
FunctionCallAgentRunner.run()                       CotAgentRunner.run()
  |                                                   |
  | 循环:                                             | 循环:
  |  (1) _organize_prompt_messages()                  |  (1) _organize_prompt_messages()  [抽象，子类实现]
  |  (2) invoke_llm(tools=prompt_messages_tools)      |  (2) invoke_llm(tools=[])         [不传 tools]
  |  (3) check_tool_calls / extract_tool_calls        |  (3) CotAgentOutputParser.handle_react_stream_output()
  |  (4) ToolEngine.agent_invoke()                    |  (4) 解析出 AgentScratchpadUnit.Action
  |  (5) ToolPromptMessage -> _current_thoughts       |  (5) 若非 Final Answer -> _handle_invoke_action()
  |  (6) 直到无 tool_calls                            |  (6) 直到 action_name == "Final Answer"
  |                                                   |
  +-------------------+-------------------------------+
                      |
                      v
     QueueMessageEndEvent 发布到队列
```

**子类 Prompt 组织差异**：

```
CotChatAgentRunner._organize_prompt_messages():
  [SystemPromptMessage] + [historic_messages] + [UserPromptMessage]
  + [AssistantPromptMessage(scratchpad)] + [UserPromptMessage("continue")]

CotCompletionAgentRunner._organize_prompt_messages():
  [UserPromptMessage(整个 prompt 拼接字符串)]
  其中 prompt = system_prompt + historic_messages + agent_scratchpad + query
```

**插件策略调用流程**：

```
PluginAgentStrategy._invoke()
  -> initialize_parameters()        <-- 初始化前端参数
  -> convert_parameters_to_plugin_format()  <-- 转换为插件格式
  -> PluginAgentClient.invoke()     <-- 远程调用插件 Agent
  -> yield AgentInvokeMessage       <-- 流式返回结果
```

---

## 5. 类继承体系

```
AppRunner (core.app.apps.base_app_runner)
  |
  +-- BaseAgentRunner (base_agent_runner.py)
        |
        +-- FunctionCallAgentRunner (fc_agent_runner.py)
        |
        +-- CotAgentRunner (cot_agent_runner.py) [ABC]
              |
              +-- CotChatAgentRunner (cot_chat_agent_runner.py)
              |
              +-- CotCompletionAgentRunner (cot_completion_agent_runner.py)

BaseAgentStrategy (strategy/base.py) [ABC]
  |
  +-- PluginAgentStrategy (strategy/plugin.py)
```

---

## 6. 两种策略的核心对比

| 维度 | Function Calling | Chain-of-Thought / ReAct |
|---|---|---|
| 工具传递方式 | 通过 `tools` 参数传给 LLM | 通过 Prompt 文本描述工具 |
| LLM 输出格式 | 结构化的 `tool_calls` JSON | 自由文本 `Thought/Action/Observation` |
| 输出解析方式 | 直接读取 `tool_calls` 字段 | `CotAgentOutputParser` 逐字符解析 |
| 适用模型 | 支持 function calling 的模型（GPT-4、Claude 等） | 所有支持文本补全的模型 |
| 可靠性 | 高（结构化输出） | 中（依赖 Prompt 引导和文本解析） |
| 灵活性 | 受限于模型原生能力 | 高（可自定义 Prompt 模板） |
| stop 词 | 不需要 | 需要 "Observation" |

---

## 7. 详细讲解文档索引

每个文件的详细讲解保存在同目录下的 `docs/` 文件夹中：

- [entities.py 详解](./docs/01-entities.md)
- [errors.py 详解](./docs/02-errors.md)
- [base_agent_runner.py 详解](./docs/03-base_agent_runner.md)
- [fc_agent_runner.py 详解](./docs/04-fc_agent_runner.md)
- [cot_agent_runner.py 详解](./docs/05-cot_agent_runner.md)
- [cot_chat_agent_runner.py 详解](./docs/06-cot_chat_agent_runner.md)
- [cot_completion_agent_runner.py 详解](./docs/07-cot_completion_agent_runner.md)
- [cot_output_parser.py 详解](./docs/08-cot_output_parser.md)
- [template.py 详解](./docs/09-template.md)
- [plugin_entities.py 详解](./docs/10-plugin_entities.md)
- [strategy/base.py 详解](./docs/11-strategy_base.md)
- [strategy/plugin.py 详解](./docs/12-strategy_plugin.md)
