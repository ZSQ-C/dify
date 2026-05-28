# base_agent_runner.py 详细讲解

**1. 文件定位**：Agent 模块的**核心基类**，所有 Agent 策略运行器（FunctionCallAgentRunner、CotAgentRunner）都继承自此类。它负责：工具初始化与转换、Agent 思维记录的创建与保存、历史对话的组织与重建、模型特性检测。被 `fc_agent_runner.py` 和 `cot_agent_runner.py` 直接继承。

**2. 代码结构**：包含 1 个类 `BaseAgentRunner`，内含多个方法

| 方法名 | 作用 |
|---|---|
| `__init__()` | 初始化 Agent 运行上下文 |
| `_repack_app_generate_entity()` | 重新打包应用生成实体 |
| `_convert_tool_to_prompt_message_tool()` | 将 AgentToolEntity 转换为 LLM 可识别的 PromptMessageTool |
| `_convert_dataset_retriever_tool_to_prompt_message_tool()` | 将数据集检索工具转换为 PromptMessageTool |
| `_init_prompt_tools()` | 初始化所有工具（Agent 工具 + 数据集工具） |
| `update_prompt_message_tool()` | 更新工具的参数 schema |
| `create_agent_thought()` | 创建一条 Agent 思维记录到数据库 |
| `save_agent_thought()` | 保存/更新 Agent 思维记录到数据库 |
| `organize_agent_history()` | 从数据库重建历史对话消息列表 |
| `organize_agent_user_prompt()` | 组织用户消息（支持多模态文件） |

---

**3. 执行流程详解**：

### `__init__()`

```python
def __init__(
    self,
    *,
    tenant_id: str,
    application_generate_entity: AgentChatAppGenerateEntity,
    conversation: Conversation,
    app_config: AgentChatAppConfig,
    model_config: ModelConfigWithCredentialsEntity,
    config: AgentEntity,
    queue_manager: AppQueueManager,
    message: Message,
    user_id: str,
    model_instance: ModelInstance,
    memory: TokenBufferMemory | None = None,
    prompt_messages: list[PromptMessage] | None = None,
):
```

- **作用**：初始化 Agent 运行所需的全部上下文
- **入参讲解**：
  - `tenant_id: str` -- 租户 ID，用于多租户隔离
  - `application_generate_entity: AgentChatAppGenerateEntity` -- 应用生成实体，包含完整的调用参数
  - `conversation: Conversation` -- 当前对话的数据库模型对象
  - `app_config: AgentChatAppConfig` -- Agent Chat 应用的配置管理器
  - `model_config: ModelConfigWithCredentialsEntity` -- 模型配置（含凭证）
  - `config: AgentEntity` -- Agent 配置实体（策略、工具、迭代上限等）
  - `queue_manager: AppQueueManager` -- 队列管理器，用于发布事件
  - `message: Message` -- 当前消息的数据库模型对象
  - `user_id: str` -- 用户 ID
  - `model_instance: ModelInstance` -- 模型实例，用于调用 LLM
  - `memory: TokenBufferMemory | None = None` -- 记忆管理器，用于管理对话历史的 token 预算
  - `prompt_messages: list[PromptMessage] | None = None` -- 初始的 prompt 消息列表

- **内部执行步骤**：

  1. **保存基本属性**：将所有入参保存为实例属性
     ```python
     self.tenant_id = tenant_id
     self.application_generate_entity = application_generate_entity
     self.conversation = conversation
     self.app_config = app_config
     self.model_config = model_config
     self.config = config
     self.queue_manager = queue_manager
     self.message = message
     self.user_id = user_id
     self.memory = memory
     ```

  2. **组织历史消息**：调用 `organize_agent_history()` 从数据库重建历史对话
     ```python
     self.history_prompt_messages = self.organize_agent_history(prompt_messages=prompt_messages or [])
     ```

  3. **保存模型实例**：
     ```python
     self.model_instance = model_instance
     ```

  4. **初始化 Agent 回调处理器**：
     ```python
     self.agent_callback = DifyAgentCallbackHandler()
     ```

  5. **初始化数据集检索工具**：
     - 创建 `DatasetIndexToolCallbackHandler`，用于处理数据集检索的回调
     - 调用 `DatasetRetrieverTool.get_dataset_tools()` 获取当前应用关联的数据集检索工具
     - 参数包括：租户 ID、数据集 ID 列表、检索配置、是否返回资源、调用来源、回调、用户 ID、输入参数

  6. **查询已有的 Agent 思维记录数量**：
     ```python
     self.agent_thought_count = (
         db.session.scalar(
             select(func.count())
             .select_from(MessageAgentThought)
             .where(MessageAgentThought.message_id == self.message.id)
         ) or 0
     )
     ```
     - 通过 SQL COUNT 查询当前消息下已有的思维记录数量
     - 用于为新创建的思维记录计算 `position`（排序位置）

  7. **检测模型特性**：
     ```python
     llm_model = cast(LargeLanguageModel, model_instance.model_type_instance)
     model_schema = llm_model.get_model_schema(model_instance.model_name, model_instance.credentials)
     features = model_schema.features if model_schema and model_schema.features else []
     self.stream_tool_call = ModelFeature.STREAM_TOOL_CALL in features
     self.files = application_generate_entity.files if ModelFeature.VISION in features else []
     ```
     - 获取模型的 schema 和特性列表
     - `self.stream_tool_call`：模型是否支持流式工具调用（影响 FC 模式的调用方式）
     - `self.files`：模型是否支持视觉能力，如果支持则保留文件，否则清空

  8. **初始化查询和思维列表**：
     ```python
     self.query: str | None = ""
     self._current_thoughts: list[PromptMessage] = []
     ```
     - `self.query`：当前用户查询
     - `self._current_thoughts`：当前迭代中累积的思考消息列表（AssistantMessage + ToolPromptMessage）

---

### `_repack_app_generate_entity()`

```python
def _repack_app_generate_entity(
    self, app_generate_entity: AgentChatAppGenerateEntity
) -> AgentChatAppGenerateEntity:
    if app_generate_entity.app_config.prompt_template.simple_prompt_template is None:
        app_generate_entity.app_config.prompt_template.simple_prompt_template = ""
    return app_generate_entity
```

- **作用**：确保 `simple_prompt_template` 不为 None，如果为 None 则设为空字符串
- **设计意图**：避免后续代码中对 None 值进行字符串操作时报错

---

### `_convert_tool_to_prompt_message_tool()`

```python
def _convert_tool_to_prompt_message_tool(self, tool: AgentToolEntity) -> tuple[PromptMessageTool, Tool]:
    tool_entity = ToolManager.get_agent_tool_runtime(
        tenant_id=self.tenant_id,
        app_id=self.app_config.app_id,
        agent_tool=tool,
        user_id=self.user_id,
        invoke_from=self.application_generate_entity.invoke_from,
    )
    assert tool_entity.entity.description
    message_tool = PromptMessageTool(
        name=tool.tool_name,
        description=tool_entity.entity.description.llm,
        parameters=tool_entity.get_llm_parameters_json_schema(),
    )
    return message_tool, tool_entity
```

- **作用**：将配置层的 `AgentToolEntity` 转换为 LLM 运行时需要的 `PromptMessageTool` 和可执行的 `Tool` 实例
- **入参**：`tool: AgentToolEntity` -- Agent 工具配置
- **出参**：`tuple[PromptMessageTool, Tool]` -- (LLM 可识别的工具描述, 可执行的工具实例)
- **内部执行步骤**：
  1. 调用 `ToolManager.get_agent_tool_runtime()` 获取工具的运行时实例，传入租户 ID、应用 ID、工具配置、用户 ID、调用来源
  2. 断言工具实体必须有描述信息（`assert tool_entity.entity.description`），因为 LLM 需要工具描述来决定何时调用
  3. 构造 `PromptMessageTool`，包含三个关键字段：
     - `name`：工具名称
     - `description`：给 LLM 看的工具描述（`llm` 字段）
     - `parameters`：工具参数的 JSON Schema（通过 `get_llm_parameters_json_schema()` 获取）
  4. 返回消息工具和运行时工具实例的元组

---

### `_convert_dataset_retriever_tool_to_prompt_message_tool()`

```python
def _convert_dataset_retriever_tool_to_prompt_message_tool(self, tool: DatasetRetrieverTool) -> PromptMessageTool:
    assert tool.entity.description
    prompt_tool = PromptMessageTool(
        name=tool.entity.identity.name,
        description=tool.entity.description.llm,
        parameters={
            "type": "object",
            "properties": {},
            "required": [],
        },
    )
    for parameter in tool.get_runtime_parameters():
        parameter_type = "string"
        prompt_tool.parameters["properties"][parameter.name] = {
            "type": parameter_type,
            "description": parameter.llm_description or "",
        }
        if parameter.required:
            if parameter.name not in prompt_tool.parameters["required"]:
                prompt_tool.parameters["required"].append(parameter.name)
    return prompt_tool
```

- **作用**：将数据集检索工具转换为 LLM 可识别的 `PromptMessageTool`
- **与 `_convert_tool_to_prompt_message_tool` 的区别**：数据集检索工具的参数需要手动构建 JSON Schema，而普通工具通过 `get_llm_parameters_json_schema()` 自动获取
- **内部执行步骤**：
  1. 断言工具必须有描述
  2. 创建基础的 `PromptMessageTool`，参数 schema 初始为空对象
  3. 遍历 `tool.get_runtime_parameters()` 获取运行时参数
  4. 对每个参数，类型统一设为 `"string"`，添加描述和是否必填标记
  5. 返回完整的 `PromptMessageTool`

---

### `_init_prompt_tools()`

```python
def _init_prompt_tools(self) -> tuple[dict[str, Tool], list[PromptMessageTool]]:
    tool_instances = {}
    prompt_messages_tools = []

    for tool in self.app_config.agent.tools or [] if self.app_config.agent else []:
        try:
            prompt_tool, tool_entity = self._convert_tool_to_prompt_message_tool(tool)
        except Exception:
            continue
        tool_instances[tool.tool_name] = tool_entity
        prompt_messages_tools.append(prompt_tool)

    for dataset_tool in self.dataset_tools:
        prompt_tool = self._convert_dataset_retriever_tool_to_prompt_message_tool(dataset_tool)
        prompt_messages_tools.append(prompt_tool)
        tool_instances[dataset_tool.entity.identity.name] = dataset_tool

    return tool_instances, prompt_messages_tools
```

- **作用**：初始化所有工具，返回工具实例字典和 LLM 工具描述列表
- **出参**：`tuple[dict[str, Tool], list[PromptMessageTool]]`
  - `tool_instances`：以工具名为 key、工具运行时实例为 value 的字典，用于后续执行工具调用
  - `prompt_messages_tools`：LLM 可识别的工具描述列表，传给 `invoke_llm()` 的 `tools` 参数
- **内部执行步骤**：
  1. 初始化空字典和空列表
  2. 遍历 Agent 配置中的工具列表：
     - 调用 `_convert_tool_to_prompt_message_tool()` 转换每个工具
     - 如果转换失败（如 API 工具已被删除），捕获异常并跳过
     - 将工具实例和消息工具分别保存
  3. 遍历数据集检索工具列表：
     - 调用 `_convert_dataset_retriever_tool_to_prompt_message_tool()` 转换
     - 同样保存到字典和列表中
  4. 返回两个集合

---

### `update_prompt_message_tool()`

```python
def update_prompt_message_tool(self, tool: Tool, prompt_tool: PromptMessageTool) -> PromptMessageTool:
    prompt_tool.parameters = tool.get_llm_parameters_json_schema()
    return prompt_tool
```

- **作用**：更新工具的参数 schema（某些工具的参数可能随运行时状态变化）
- **入参**：`tool` -- 工具运行时实例，`prompt_tool` -- 需要更新的 PromptMessageTool
- **出参**：更新后的 PromptMessageTool

---

### `create_agent_thought()`

```python
def create_agent_thought(
    self, message_id: str, message: str, tool_name: str, tool_input: str, messages_ids: list[str]
) -> str:
```

- **作用**：在数据库中创建一条 Agent 思维记录，记录 Agent 在某一轮迭代中的状态
- **入参**：
  - `message_id: str` -- 关联的消息 ID
  - `message: str` -- 消息内容
  - `tool_name: str` -- 工具名称
  - `tool_input: str` -- 工具输入
  - `messages_ids: list[str]` -- 关联的消息文件 ID 列表
- **出参**：`str` -- 新创建的思维记录 ID
- **内部执行步骤**：
  1. 构造 `MessageAgentThought` 对象，所有字段初始化：
     - `thought=""` -- 思考内容初始为空，后续通过 `save_agent_thought()` 更新
     - `position=self.agent_thought_count + 1` -- 排序位置，基于已有记录数量递增
     - 价格相关字段初始为 0 或 0.001（单价单位）
     - `currency="USD"` -- 货币单位
  2. 将对象添加到数据库会话并提交
  3. 获取新记录的 ID 并转换为字符串
  4. 递增 `self.agent_thought_count`
  5. 关闭数据库会话
  6. 返回思维记录 ID

---

### `save_agent_thought()`

```python
def save_agent_thought(
    self,
    agent_thought_id: str,
    tool_name: str | None,
    tool_input: Union[str, dict, None],
    thought: str | None,
    observation: Union[str, dict, None],
    tool_invoke_meta: Union[str, dict, None],
    answer: str | None,
    messages_ids: list[str],
    llm_usage: LLMUsage | None = None,
):
```

- **作用**：更新已有的 Agent 思维记录，追加思考内容、工具调用结果、使用量等信息
- **入参**：
  - `agent_thought_id: str` -- 要更新的思维记录 ID
  - `tool_name: str | None` -- 工具名称（可选，不更新则传 None）
  - `tool_input: Union[str, dict, None]` -- 工具输入（可选）
  - `thought: str | None` -- 思考内容（追加到已有内容后面）
  - `observation: Union[str, dict, None]` -- 观测结果（可选）
  - `tool_invoke_meta: Union[str, dict, None]` -- 工具调用元数据（可选）
  - `answer: str | None` -- 回答内容（可选）
  - `messages_ids: list[str]` -- 关联的消息文件 ID 列表
  - `llm_usage: LLMUsage | None = None` -- LLM 使用量信息（可选）
- **内部执行步骤**：
  1. 从数据库查询指定 ID 的 `MessageAgentThought`，如果不存在则抛出 `ValueError`
  2. 如果 `thought` 不为空，**追加**到已有的 thought 内容后面（而非覆盖）
  3. 如果 `tool_name` 不为空，更新工具名称
  4. 如果 `tool_input` 不为空：
     - 如果是字典类型，先序列化为 JSON 字符串（优先使用 `ensure_ascii=False` 保留非 ASCII 字符）
     - 如果序列化失败，回退到默认的 ASCII 编码
     - 更新到数据库记录
  5. 如果 `observation` 不为空，同样处理字典到 JSON 的转换
  6. 如果 `answer` 不为空，更新回答内容
  7. 如果 `messages_ids` 不为空，序列化为 JSON 并更新
  8. 如果 `llm_usage` 不为空，更新所有 token 和价格相关字段：
     - `message_token` / `answer_token` -- prompt 和 completion 的 token 数
     - `message_unit_price` / `answer_unit_price` -- 单价
     - `message_price_unit` / `answer_price_unit` -- 价格单位
     - `tokens` / `total_price` -- 总 token 数和总价格
  9. 处理工具标签（`tool_labels`）：
     - 获取已有的标签字典
     - 将 `tool` 字段按 `;` 分割得到工具名列表
     - 对每个工具名，如果不在标签字典中，则通过 `ToolManager.get_tool_label()` 获取标签
     - 如果获取不到标签，则使用工具名本身作为英文和中文标签
     - 将标签字典序列化为 JSON 字符串
  10. 如果 `tool_invoke_meta` 不为空，处理字典到 JSON 的转换并更新
  11. 提交数据库事务并关闭会话

---

### `organize_agent_history()`

```python
def organize_agent_history(self, prompt_messages: list[PromptMessage]) -> list[PromptMessage]:
```

- **作用**：从数据库重建当前对话的完整历史消息列表，包括用户消息、Agent 思维（含工具调用和响应）
- **入参**：`prompt_messages: list[PromptMessage]` -- 初始的 prompt 消息列表（可能包含 SystemPromptMessage）
- **出参**：`list[PromptMessage]` -- 重建后的完整历史消息列表
- **内部执行步骤**：
  1. 初始化结果列表 `result`
  2. 从输入的 `prompt_messages` 中提取所有 `SystemPromptMessage`，添加到结果列表开头
  3. 从数据库查询当前对话的所有历史消息，按创建时间降序排列
  4. 调用 `extract_thread_messages()` 提取线程消息并反转（变为时间正序）
  5. 遍历每条历史消息（跳过当前消息）：
     a. 调用 `organize_agent_user_prompt()` 构建用户消息，添加到结果列表
     b. 如果消息有关联的 `agent_thoughts`：
        - 对每个 thought，检查 `tool` 字段是否非空
        - 如果有工具调用：
          - 将 `tool` 字段按 `;` 分割得到工具名列表
          - 解析 `tool_input` 和 `observation` 的 JSON
          - 为每个工具调用生成 UUID 作为 `tool_call_id`
          - 构造 `AssistantPromptMessage.ToolCall` 和 `ToolPromptMessage`
          - 将 Assistant 消息（含 tool_calls）和 Tool 响应消息添加到结果列表
        - 如果没有工具调用，直接添加 `AssistantPromptMessage`
     c. 如果消息没有 agent_thoughts 但有 answer，添加 `AssistantPromptMessage`
  6. 关闭数据库会话
  7. 返回结果列表

---

### `organize_agent_user_prompt()`

```python
def organize_agent_user_prompt(self, message: Message) -> UserPromptMessage:
```

- **作用**：将数据库中的 `Message` 对象转换为 `UserPromptMessage`，支持多模态内容（图片、文件）
- **入参**：`message: Message` -- 数据库消息对象
- **出参**：`UserPromptMessage` -- 可传给 LLM 的用户消息
- **内部执行步骤**：
  1. 查询消息关联的文件列表
  2. 如果没有文件，直接返回纯文本的 `UserPromptMessage`
  3. 如果有文件，获取文件上传配置中的图片详细级别
  4. 调用 `file_factory.build_from_message_files()` 构建文件对象列表
  5. 如果文件对象列表为空，返回纯文本消息
  6. 否则，将每个文件转换为 `PromptMessageContent`（图片/文件），并追加文本查询
  7. 返回包含多模态内容的 `UserPromptMessage`

---

**4. 补充说明**：
- `BaseAgentRunner` 不定义 `run()` 方法，由子类各自实现
- `organize_agent_history()` 是重建对话上下文的关键方法，它将数据库中的结构化数据还原为 LLM 可理解的消息序列
- `create_agent_thought()` 和 `save_agent_thought()` 的分离设计允许"先创建空记录，再逐步填充"的模式，这在流式处理中非常重要
