# cot_agent_runner.py 详细讲解

**1. 文件定位**：实现 **Chain-of-Thought / ReAct 策略**的 Agent 运行器基类。当 `AgentEntity.Strategy` 为 `CHAIN_OF_THOUGHT` 时使用此类的子类。它通过 Prompt 模板引导 LLM 按照 `Thought -> Action -> Observation` 的循环进行推理。

**2. 代码结构**：包含 2 个类和多个方法

| 类/方法名 | 作用 |
|---|---|
| `ActionDict` (TypedDict) | Action 字典的类型定义 |
| `CotAgentRunner` | CoT/ReAct 策略基类（ABC） |
| `CotAgentRunner.run()` | ReAct 主循环 |
| `CotAgentRunner._handle_invoke_action()` | 执行工具调用 |
| `CotAgentRunner._convert_dict_to_action()` | 字典转 Action 对象 |
| `CotAgentRunner._fill_in_inputs_from_external_data_tools()` | 填充模板变量 |
| `CotAgentRunner._init_react_state()` | 初始化 ReAct 状态 |
| `CotAgentRunner._organize_prompt_messages()` | 抽象方法，子类实现 |
| `CotAgentRunner._format_assistant_message()` | 格式化 scratchpad 为文本 |
| `CotAgentRunner._organize_historic_prompt_messages()` | 组织历史 Prompt 消息 |

---

**3. 执行流程详解**：

### `CotAgentRunner` 类属性

```python
class CotAgentRunner(BaseAgentRunner, ABC):
    _is_first_iteration = True
    _ignore_observation_providers = ["wenxin"]
    _historic_prompt_messages: list[PromptMessage]
    _agent_scratchpad: list[AgentScratchpadUnit]
    _instruction: str
    _query: str
    _prompt_messages_tools: Sequence[PromptMessageTool]
```

- `_is_first_iteration` -- 是否是第一次迭代的标志
- `_ignore_observation_providers` -- 不需要自动添加 "Observation" stop 词的模型提供方列表（文心等模型有自己的处理方式）
- `_historic_prompt_messages` -- 历史消息列表
- `_agent_scratchpad` -- 当前会话的 scratchpad 列表，记录每轮的 Thought/Action/Observation
- `_instruction` -- 填充变量后的指令文本
- `_query` -- 当前用户查询
- `_prompt_messages_tools` -- 工具描述列表

---

### `run()`

```python
def run(self, message: Message, query: str, inputs: Mapping[str, str]) -> Generator:
```

- **作用**：ReAct 策略的主执行循环
- **入参**：
  - `message: Message` -- 当前消息对象
  - `query: str` -- 用户查询
  - `inputs: Mapping[str, str]` -- 外部数据工具的输入变量

- **内部执行步骤**：

  1. **重新打包应用实体**：`self._repack_app_generate_entity(app_generate_entity)`

  2. **初始化 ReAct 状态**：`self._init_react_state(query)`
     - 设置 `_query`
     - 清空 `_agent_scratchpad`
     - 组织历史消息

  3. **添加 stop 词**：
     ```python
     if "Observation" not in app_generate_entity.model_conf.stop:
         if app_generate_entity.model_conf.provider not in self._ignore_observation_providers:
             app_generate_entity.model_conf.stop.append("Observation")
     ```
     - 在 stop 列表中添加 "Observation"，让 LLM 在输出 Observation 后停止，等待工具执行结果
     - 文心等提供方被排除，因为它们有自己的处理逻辑

  4. **初始化指令**：
     ```python
     instruction = app_config.prompt_template.simple_prompt_template or ""
     self._instruction = self._fill_in_inputs_from_external_data_tools(instruction, inputs)
     ```
     - 获取 Prompt 模板
     - 用外部数据工具的输入变量替换模板中的 `{{key}}` 占位符

  5. **初始化工具和迭代参数**（与 FC 类似）

  6. **主循环 `while function_call_state and iteration_step <= max_iteration_steps:`**：

     a. **重置标志**，最后一轮移除工具

     b. **创建思维记录**

     c. **第二轮及以后发布思维事件**

     d. **组装 Prompt 并调用 LLM**：
        ```python
        prompt_messages = self._organize_prompt_messages()
        self.recalc_llm_max_tokens(self.model_config, prompt_messages)
        chunks = model_instance.invoke_llm(
            prompt_messages=prompt_messages,
            model_parameters=app_generate_entity.model_conf.parameters,
            tools=[],   # <-- 关键区别：不传 tools！
            stop=app_generate_entity.model_conf.stop,
            stream=True,
            callbacks=[],
        )
        ```
        - **与 FC 的关键区别**：`tools=[]`，不传工具描述给 LLM，因为 CoT 模式通过 Prompt 文本引导

     e. **使用 CotAgentOutputParser 解析流式输出**：
        ```python
        react_chunks = CotAgentOutputParser.handle_react_stream_output(chunks, usage_dict)
        ```
        - 解析器将 LLM 的文本输出逐字符分析，提取 Thought 和 Action

     f. **遍历解析结果**：
        - 如果是 `AgentScratchpadUnit.Action` 类型：记录到 scratchpad 的 action 字段
        - 如果是字符串类型：追加到 scratchpad 的 thought 和 agent_response 字段，同时 `yield` 给上层

     g. **处理 thought 为空的情况**：
        ```python
        scratchpad.thought = scratchpad.thought.strip() or "I am thinking about how to help you"
        ```
        - 如果模型没有输出 Thought，使用默认文本

     h. **将 scratchpad 添加到列表**：`self._agent_scratchpad.append(scratchpad)`

     i. **检查最大迭代**：如果到达上限且 Action 不是 "Final Answer"，抛出 `AgentMaxIterationError`

     j. **保存思维记录**

     k. **判断是否到达最终答案**：
        - 如果 `scratchpad.action` 为 None：没有解析到 Action，设置 `final_answer = ""`
        - 如果 `action_name` 是 "Final Answer"：提取 `action_input` 作为最终答案
        - 否则：设置 `function_call_state = True`，调用 `_handle_invoke_action()` 执行工具
        - 工具执行后，将结果保存到 scratchpad 的 observation 字段

     l. **更新工具参数 schema**

     m. **递增迭代步数**

  7. **输出最终答案**：`yield LLMResultChunk(...)`

  8. **保存最终思维记录**

  9. **发布结束事件**

---

### `_handle_invoke_action()`

```python
def _handle_invoke_action(
    self,
    action: AgentScratchpadUnit.Action,
    tool_instances: Mapping[str, Tool],
    message_file_ids: list[str],
    trace_manager: TraceQueueManager | None = None,
) -> tuple[str, ToolInvokeMeta]:
```

- **作用**：执行工具调用并返回结果
- **入参**：
  - `action` -- 要执行的动作（含工具名和参数）
  - `tool_instances` -- 工具实例字典
  - `message_file_ids` -- 消息文件 ID 列表（会被追加）
  - `trace_manager` -- 追踪管理器
- **出参**：`tuple[str, ToolInvokeMeta]` -- (工具响应文本, 调用元数据)
- **内部执行步骤**：
  1. 从 `tool_instances` 中获取工具实例
  2. 如果工具不存在，返回错误消息和错误元数据
  3. 如果 `action_input` 是字符串，尝试解析为 JSON
  4. 调用 `ToolEngine.agent_invoke()` 执行工具
  5. 发布文件事件
  6. 返回工具响应和元数据

---

### `_fill_in_inputs_from_external_data_tools()`

```python
def _fill_in_inputs_from_external_data_tools(self, instruction: str, inputs: Mapping[str, Any]) -> str:
    for key, value in inputs.items():
        try:
            instruction = instruction.replace(f"{{{{{key}}}}}", str(value))
        except Exception:
            continue
    return instruction
```

- **作用**：将输入变量替换到指令模板中
- **替换格式**：`{{key}}` -> `value`
- **异常处理**：如果替换失败（如值无法转为字符串），跳过继续

---

### `_init_react_state()`

```python
def _init_react_state(self, query):
    self._query = query
    self._agent_scratchpad = []
    self._historic_prompt_messages = self._organize_historic_prompt_messages()
```

- **作用**：初始化 ReAct 循环的状态变量

---

### `_organize_prompt_messages()` (抽象方法)

- **作用**：由子类实现，定义如何组装 Prompt 消息列表
- **设计意图**：Chat 模式和 Completion 模式的消息组织方式完全不同，通过抽象方法实现多态

---

### `_format_assistant_message()`

```python
def _format_assistant_message(self, agent_scratchpad: list[AgentScratchpadUnit]) -> str:
    message = ""
    for scratchpad in agent_scratchpad:
        if scratchpad.is_final():
            message += f"Final Answer: {scratchpad.agent_response}"
        else:
            message += f"Thought: {scratchpad.thought}\n\n"
            if scratchpad.action_str:
                message += f"Action: {scratchpad.action_str}\n\n"
            if scratchpad.observation:
                message += f"Observation: {scratchpad.observation}\n\n"
    return message
```

- **作用**：将 scratchpad 列表格式化为 ReAct 文本格式
- **输出格式**：
  ```
  Thought: 我需要搜索相关信息
  Action: {"action": "web_search", "action_input": {"query": "..."}}
  Observation: 搜索结果...
  Thought: 我现在知道答案了
  Final Answer: 答案内容
  ```

---

### `_organize_historic_prompt_messages()`

```python
def _organize_historic_prompt_messages(
    self, current_session_messages: list[PromptMessage] | None = None
) -> list[PromptMessage]:
```

- **作用**：将历史消息中的 Agent 思维过程重新格式化为 ReAct 文本格式
- **内部执行步骤**：
  1. 遍历 `self.history_prompt_messages`
  2. 对于 `AssistantPromptMessage`：
     - 创建新的 `AgentScratchpadUnit`
     - 如果有 `tool_calls`，解析为 Action
  3. 对于 `ToolPromptMessage`：设置 observation
  4. 对于 `UserPromptMessage`：
     - 如果有累积的 scratchpad，先格式化并添加到结果
     - 然后添加用户消息
  5. 处理剩余的 scratchpad
  6. 使用 `AgentHistoryPromptTransform` 处理 token 预算

---

**4. 补充说明**：
- CoT 策略与 FC 策略的核心区别：CoT 不传 `tools` 给 LLM，而是通过 Prompt 文本描述工具
- CoT 使用 `CotAgentOutputParser` 解析 LLM 的文本输出，FC 直接读取 `tool_calls` 结构
- CoT 的 stop 词包含 "Observation"，确保 LLM 在输出 Observation 后暂停等待工具结果
- `_organize_prompt_messages()` 是唯一的抽象方法，体现了模板方法设计模式
