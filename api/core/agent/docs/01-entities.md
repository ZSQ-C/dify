# entities.py 详细讲解

**1. 文件定位**：这是 Agent 模块的**数据实体定义层**，定义了所有在 Agent 运行过程中流转的核心数据结构。它被 `base_agent_runner.py`、`cot_agent_runner.py`、`fc_agent_runner.py`、`cot_output_parser.py`、`strategy/base.py`、`strategy/plugin.py` 等几乎所有其他文件引用。

**2. 代码结构**：包含 5 个类

| 类名 | 作用 |
|---|---|
| `AgentToolEntity` | 描述 Agent 可调用的单个工具 |
| `AgentPromptEntity` | 描述 Agent 的 Prompt 模板配置 |
| `AgentScratchpadUnit` | 描述 Agent 单轮推理的暂存区（Thought/Action/Observation） |
| `AgentEntity` | 描述 Agent 的完整配置（策略、模型、工具、迭代上限） |
| `AgentInvokeMessage` | Agent 调用的消息类型 |

---

**3. 执行流程详解**：

### `AgentToolEntity`

```python
class AgentToolEntity(BaseModel):
    provider_type: ToolProviderType
    provider_id: str
    tool_name: str
    tool_parameters: dict[str, Any] = Field(default_factory=dict)
    plugin_unique_identifier: str | None = None
    credential_id: str | None = None
```

- **作用**：描述 Agent 可调用的一个工具的完整信息
- **字段逐行讲解**：
  - `provider_type: ToolProviderType` -- 工具提供方类型（如内置工具、API 工具、插件工具等），从 `core.tools.entities.tool_entities` 导入的枚举类型
  - `provider_id: str` -- 工具提供方的唯一标识符，用于定位工具属于哪个 provider
  - `tool_name: str` -- 工具名称，如 `web_search`、`calculator` 等，是工具在 provider 内的唯一标识
  - `tool_parameters: dict[str, Any] = Field(default_factory=dict)` -- 工具的默认参数，键值对形式，默认为空字典。使用 `Field(default_factory=dict)` 而非 `= {}` 是 Pydantic 的最佳实践，避免可变默认值的共享问题
  - `plugin_unique_identifier: str | None = None` -- 如果该工具来自插件，则存储插件的唯一标识符；否则为 None
  - `credential_id: str | None = None` -- 工具调用所需的凭证 ID，某些 API 工具需要认证信息

---

### `AgentPromptEntity`

```python
class AgentPromptEntity(BaseModel):
    first_prompt: str
    next_iteration: str
```

- **作用**：描述 CoT/ReAct 策略中使用的 Prompt 模板对
- **字段讲解**：
  - `first_prompt: str` -- 第一轮迭代使用的 Prompt 模板，包含 `{{instruction}}`、`{{tools}}`、`{{tool_names}}` 等占位符
  - `next_iteration: str` -- 后续迭代使用的 Prompt 模板，通常只包含 `Observation: {{observation}}\nThought:` 部分，用于引导模型继续推理
- **设计意图**：将首轮和后续轮次的 Prompt 分开，因为首轮需要完整的系统指令和工具描述，后续轮次只需要追加观测结果

---

### `AgentScratchpadUnit`

```python
class AgentScratchpadUnit(BaseModel):
    class Action(BaseModel):
        action_name: str
        action_input: Union[dict, str]

        def to_dict(self):
            return {
                "action": self.action_name,
                "action_input": self.action_input,
            }

    agent_response: str | None = None
    thought: str | None = None
    action_str: str | None = None
    observation: str | None = None
    action: Action | None = None

    def is_final(self) -> bool:
        return self.action is None or (
            "final" in self.action.action_name.lower() and "answer" in self.action.action_name.lower()
        )
```

- **作用**：这是 ReAct 推理模式中最核心的数据结构，记录 Agent 在**每一轮迭代**中的完整思维过程
- **内部类 `Action`**：
  - `action_name: str` -- 动作名称，可以是工具名（如 `web_search`）或 `"Final Answer"`（表示推理结束）
  - `action_input: Union[dict, str]` -- 动作输入，可以是字典（工具参数）或字符串（最终答案文本）
  - `to_dict()` -- 将 Action 转换为标准字典格式 `{"action": ..., "action_input": ...}`，用于序列化到 Prompt 中
- **外部字段**：
  - `agent_response: str | None = None` -- Agent 在这一轮的完整原始响应文本（包含 Thought 和 Action 的原始输出）
  - `thought: str | None = None` -- Agent 的思考过程文本，即 `Thought: ...` 后面的内容
  - `action_str: str | None = None` -- Action 部分的原始 JSON 字符串表示，用于在 Prompt 中回放
  - `observation: str | None = None` -- 工具执行后返回的观测结果文本，即 `Observation: ...` 后面的内容
  - `action: Action | None = None` -- 解析后的结构化 Action 对象，如果 LLM 输出中包含可解析的 Action 则不为 None
- **`is_final()` 方法**：
  - 如果 `self.action is None`，说明没有解析到 Action（可能是纯文本回复），返回 `True`
  - 如果 `action_name` 同时包含 `"final"` 和 `"answer"`（不区分大小写），说明 Agent 认为已经可以给出最终答案，返回 `True`
  - 否则返回 `False`，表示 Agent 还需要继续调用工具
  - **设计意图**：这个判断是 ReAct 循环的终止条件核心

---

### `AgentEntity`

```python
class AgentEntity(BaseModel):
    class Strategy(StrEnum):
        CHAIN_OF_THOUGHT = "chain-of-thought"
        FUNCTION_CALLING = "function-calling"

    provider: str
    model: str
    strategy: Strategy
    prompt: AgentPromptEntity | None = None
    tools: list[AgentToolEntity] | None = None
    max_iteration: int = 10
```

- **作用**：Agent 的完整配置实体，是整个模块的"配置中心"
- **内部枚举 `Strategy`**：
  - `CHAIN_OF_THOUGHT = "chain-of-thought"` -- 使用 ReAct 模式，通过 Prompt 引导推理
  - `FUNCTION_CALLING = "function-calling"` -- 使用 LLM 原生的函数调用能力
- **字段讲解**：
  - `provider: str` -- 模型提供方名称（如 `openai`、`anthropic`）
  - `model: str` -- 模型名称（如 `gpt-4`、`claude-3-opus`）
  - `strategy: Strategy` -- 使用的策略类型，决定走 FC 还是 CoT 分支
  - `prompt: AgentPromptEntity | None = None` -- CoT 策略的 Prompt 模板，FC 策略不需要此字段
  - `tools: list[AgentToolEntity] | None = None` -- Agent 可调用的工具列表
  - `max_iteration: int = 10` -- 最大迭代次数，防止无限循环，默认 10 次

---

### `AgentInvokeMessage`

```python
class AgentInvokeMessage(ToolInvokeMessage):
    pass
```

- **作用**：Agent 调用产生的消息类型，继承自 `ToolInvokeMessage`，没有添加额外字段
- **设计意图**：通过继承复用工具调用的消息结构，同时为未来扩展预留独立的类型空间

---

**4. 补充说明**：
- `AgentScratchpadUnit` 是 ReAct 模式的灵魂，它将 LLM 的非结构化文本输出解析为结构化的 `thought + action + observation` 三元组
- `AgentEntity.Strategy` 枚举是策略选择的路由键，外部调用方根据此值决定实例化 `FunctionCallAgentRunner` 还是 `CotAgentRunner` 的子类
- 所有实体均使用 Pydantic v2 的 `BaseModel`，确保数据验证和序列化的一致性
