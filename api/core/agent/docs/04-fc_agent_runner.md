# fc_agent_runner.py 详细讲解

**1. 文件定位**：实现 **Function Calling 策略**的 Agent 运行器。当 `AgentEntity.Strategy` 为 `FUNCTION_CALLING` 时使用此类。它利用 LLM 原生的 `tool_calls` 能力，让模型直接输出结构化的工具调用请求。

**2. 代码结构**：包含 1 个类 `FunctionCallAgentRunner`，继承自 `BaseAgentRunner`，内含多个方法

| 方法名 | 作用 |
|---|---|
| `run()` | FC 策略的主循环 |
| `check_tool_calls()` | 检测流式响应中是否有工具调用 |
| `check_blocking_tool_calls()` | 检测阻塞式响应中是否有工具调用 |
| `extract_tool_calls()` | 从流式响应中提取工具调用 |
| `extract_blocking_tool_calls()` | 从阻塞式响应中提取工具调用 |
| `_init_system_message()` | 初始化系统消息 |
| `_organize_user_query()` | 组织用户查询消息 |
| `_clear_user_prompt_image_messages()` | 清除用户消息中的图片内容 |
| `_organize_prompt_messages()` | 组装完整的 Prompt 消息列表 |

---

**3. 执行流程详解**：

### `run()`

```python
def run(self, message: Message, query: str, **kwargs: Any) -> Generator[LLMResultChunk, None, None]:
```

- **作用**：Function Calling 策略的主执行循环
- **入参**：
  - `message: Message` -- 当前消息对象
  - `query: str` -- 用户查询文本
- **出参**：`Generator[LLMResultChunk, None, None]` -- 流式输出的 LLM 结果块

- **内部执行步骤**：

  1. **初始化**：
     ```python
     self.query = query
     app_generate_entity = self.application_generate_entity
     app_config = self.app_config
     ```
     保存查询和配置引用

  2. **断言配置完整性**：
     ```python
     assert app_config is not None, "app_config is required"
     assert app_config.agent is not None, "app_config.agent is required"
     ```

  3. **初始化工具**：
     ```python
     tool_instances, prompt_messages_tools = self._init_prompt_tools()
     ```
     调用基类方法，获取工具实例字典和 LLM 工具描述列表

  4. **设置迭代参数**：
     ```python
     iteration_step = 1
     max_iteration_steps = min(app_config.agent.max_iteration, 99) + 1
     function_call_state = True
     llm_usage: dict[str, LLMUsage | None] = {"usage": None}
     final_answer = ""
     ```
     - `max_iteration_steps`：最大迭代步数，上限 99，+1 是因为第一次迭代也算
     - `function_call_state`：是否继续循环的标志，初始为 True
     - `llm_usage`：累积的 LLM 使用量
     - `final_answer`：最终答案累积

  5. **定义内部函数 `increase_usage()`**：用于累积 LLM 使用量（token 数和价格）

  6. **主循环 `while function_call_state and iteration_step <= max_iteration_steps:`**：

     a. **重置循环标志**：`function_call_state = False`

     b. **最后一轮移除工具**：
        ```python
        if iteration_step == max_iteration_steps:
            prompt_messages_tools = []
        ```
        最后一轮不传工具，强制模型给出最终答案

     c. **创建思维记录**：调用 `create_agent_thought()`

     d. **组装 Prompt 并调用 LLM**：
        ```python
        prompt_messages = self._organize_prompt_messages()
        self.recalc_llm_max_tokens(self.model_config, prompt_messages)
        chunks = model_instance.invoke_llm(
            prompt_messages=prompt_messages,
            model_parameters=app_generate_entity.model_conf.parameters,
            tools=prompt_messages_tools,
            stop=app_generate_entity.model_conf.stop,
            stream=self.stream_tool_call,
            callbacks=[],
        )
        ```
        - 重新计算 max_tokens（根据 prompt 长度动态调整）
        - 传入 `tools` 参数，让 LLM 知道可用的工具
        - `stream` 参数取决于模型是否支持流式工具调用

     e. **处理 LLM 响应**（分两种情况）：

        **情况一：流式响应（Generator）**
        - 遍历每个 chunk
        - 第一个 chunk 时发布 `QueueAgentThoughtEvent`
        - 检测是否有 tool_calls：`check_tool_calls(chunk)`
        - 如果有，提取工具调用：`extract_tool_calls(chunk)`
        - 累积响应文本
        - 累积使用量
        - `yield chunk` 向上层流式输出

        **情况二：阻塞式响应（LLMResult）**
        - 检测是否有 tool_calls：`check_blocking_tool_calls(result)`
        - 如果有，提取工具调用：`extract_blocking_tool_calls(result)`
        - 累积响应文本和使用量
        - 包装为 `LLMResultChunk` 并 `yield`

     f. **构造 Assistant 消息**：
        ```python
        assistant_message = AssistantPromptMessage(content=response, tool_calls=[])
        if tool_calls:
            assistant_message.tool_calls = [...]
        self._current_thoughts.append(assistant_message)
        ```
        将 LLM 的响应（含工具调用信息）添加到当前思考列表

     g. **保存思维记录**：调用 `save_agent_thought()`

     h. **检查最大迭代**：
        ```python
        if iteration_step == max_iteration_steps and tool_calls:
            raise AgentMaxIterationError(app_config.agent.max_iteration)
        ```

     i. **执行工具调用**：
        - 遍历每个 `tool_call`
        - 从 `tool_instances` 中获取工具实例
        - 如果工具不存在，返回错误消息
        - 如果工具存在，调用 `ToolEngine.agent_invoke()` 执行
        - 发布文件事件
        - 将工具响应构造为 `ToolPromptMessage`，添加到 `_current_thoughts`

     j. **保存工具调用结果**：调用 `save_agent_thought()`，记录 observation 和 tool_invoke_meta

     k. **更新工具参数 schema**：遍历所有 prompt_tool，调用 `update_prompt_message_tool()`

     l. **递增迭代步数**：`iteration_step += 1`

  7. **发布结束事件**：
     ```python
     self.queue_manager.publish(
         QueueMessageEndEvent(
             llm_result=LLMResult(
                 model=model_instance.model_name,
                 prompt_messages=prompt_messages,
                 message=AssistantPromptMessage(content=final_answer),
                 usage=llm_usage["usage"] or LLMUsage.empty_usage(),
                 system_fingerprint="",
             )
         ),
         PublishFrom.APPLICATION_MANAGER,
     )
     ```

---

### `check_tool_calls()` / `check_blocking_tool_calls()`

- **作用**：检测 LLM 响应中是否包含工具调用
- **流式版本**：检查 `llm_result_chunk.delta.message.tool_calls` 是否非空
- **阻塞版本**：检查 `llm_result.message.tool_calls` 是否非空

---

### `extract_tool_calls()` / `extract_blocking_tool_calls()`

- **作用**：从 LLM 响应中提取工具调用的详细信息
- **出参**：`list[tuple[str, str, dict[str, Any]]]` -- `[(tool_call_id, tool_call_name, tool_call_args)]`
- **内部逻辑**：
  - 遍历每个 tool_call
  - 解析 `function.arguments` 为 JSON 字典（如果非空）
  - 返回三元组列表

---

### `_init_system_message()`

```python
def _init_system_message(self, prompt_template: str, prompt_messages: list[PromptMessage]) -> list[PromptMessage]:
    if not prompt_messages and prompt_template:
        return [SystemPromptMessage(content=prompt_template)]
    if prompt_messages and not isinstance(prompt_messages[0], SystemPromptMessage) and prompt_template:
        prompt_messages.insert(0, SystemPromptMessage(content=prompt_template))
    return prompt_messages or []
```

- **作用**：确保 Prompt 消息列表以 SystemPromptMessage 开头
- **三种情况**：
  1. 没有消息但有模板：创建新的 SystemPromptMessage
  2. 有消息但第一条不是 SystemPromptMessage 且有模板：在开头插入 SystemPromptMessage
  3. 其他情况：原样返回

---

### `_organize_user_query()`

- **作用**：组织用户查询消息，支持多模态内容
- **逻辑**：如果有文件，将文件和查询文本组合为多内容消息；否则创建纯文本消息

---

### `_clear_user_prompt_image_messages()`

```python
def _clear_user_prompt_image_messages(self, prompt_messages: list[PromptMessage]) -> list[PromptMessage]:
    prompt_messages = deepcopy(prompt_messages)
    for prompt_message in prompt_messages:
        if isinstance(prompt_message, UserPromptMessage):
            if isinstance(prompt_message.content, list):
                prompt_message.content = "\n".join([
                    content.data if content.type == PromptMessageContentType.TEXT
                    else "[image]" if content.type == PromptMessageContentType.IMAGE
                    else "[file]"
                    for content in prompt_message.content
                ])
    return prompt_messages
```

- **作用**：在第一次迭代之后，将用户消息中的图片和文件替换为文本占位符
- **设计意图**：GPT 等模型在第一次迭代时同时支持 FC 和视觉，但后续迭代中图片内容可能导致 token 超限，因此需要清除
- **使用深拷贝**：避免修改原始消息列表

---

### `_organize_prompt_messages()`

```python
def _organize_prompt_messages(self):
    prompt_template = self.app_config.prompt_template.simple_prompt_template or ""
    self.history_prompt_messages = self._init_system_message(prompt_template, self.history_prompt_messages)
    query_prompt_messages = self._organize_user_query(self.query or "", [])

    self.history_prompt_messages = AgentHistoryPromptTransform(
        model_config=self.model_config,
        prompt_messages=[*query_prompt_messages, *self._current_thoughts],
        history_messages=self.history_prompt_messages,
        memory=self.memory,
    ).get_prompt()

    prompt_messages = [*self.history_prompt_messages, *query_prompt_messages, *self._current_thoughts]
    if len(self._current_thoughts) != 0:
        prompt_messages = self._clear_user_prompt_image_messages(prompt_messages)
    return prompt_messages
```

- **作用**：组装完整的 Prompt 消息列表
- **内部执行步骤**：
  1. 获取 Prompt 模板，确保历史消息以 SystemPromptMessage 开头
  2. 组织当前用户查询消息
  3. 使用 `AgentHistoryPromptTransform` 处理历史消息（考虑 token 预算和记忆窗口）
  4. 拼接：历史消息 + 用户查询 + 当前思考
  5. 如果不是第一次迭代（`_current_thoughts` 非空），清除图片内容

---

**4. 补充说明**：
- FC 策略的核心优势是利用 LLM 原生的函数调用能力，不需要通过 Prompt 引导，输出更可靠
- FC 策略将 `tools` 参数传给 `invoke_llm()`，而 CoT 策略传空列表
- FC 策略的流式和阻塞两种响应处理路径保证了兼容性
- 最后一轮移除工具列表是强制模型给出最终答案的关键设计
