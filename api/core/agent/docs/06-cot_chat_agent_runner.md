# cot_chat_agent_runner.py 详细讲解

**1. 文件定位**：实现 CoT 策略在 **Chat 对话模式**下的具体 Prompt 组织方式。继承自 `CotAgentRunner`，实现了 `_organize_prompt_messages()` 抽象方法。

**2. 代码结构**：包含 1 个类 `CotChatAgentRunner`，内含 3 个方法

| 方法名 | 作用 |
|---|---|
| `_organize_system_prompt()` | 组织系统提示消息 |
| `_organize_user_query()` | 组织用户查询消息 |
| `_organize_prompt_messages()` | 实现抽象方法，组装完整消息列表 |

---

**3. 执行流程详解**：

### `_organize_system_prompt()`

```python
def _organize_system_prompt(self) -> SystemPromptMessage:
    assert self.app_config.agent
    assert self.app_config.agent.prompt

    prompt_entity = self.app_config.agent.prompt
    first_prompt = prompt_entity.first_prompt

    system_prompt = (
        first_prompt.replace("{{instruction}}", self._instruction)
        .replace("{{tools}}", json.dumps(jsonable_encoder(self._prompt_messages_tools)))
        .replace("{{tool_names}}", ", ".join([tool.name for tool in self._prompt_messages_tools]))
    )
    return SystemPromptMessage(content=system_prompt)
```

- **作用**：构建系统提示消息，将模板中的占位符替换为实际内容
- **替换的占位符**：
  - `{{instruction}}` -> 用户定义的指令文本
  - `{{tools}}` -> 工具描述的 JSON 字符串（使用 `jsonable_encoder` 确保可序列化）
  - `{{tool_names}}` -> 工具名称的逗号分隔列表
- **设计意图**：Chat 模式下，系统提示作为独立的 `SystemPromptMessage` 存在，与用户消息分离

---

### `_organize_user_query()`

- **作用**：与 FC 策略中的同名方法类似，组织用户查询消息，支持多模态内容
- **逻辑**：如果有文件，组合为多内容消息；否则纯文本消息

---

### `_organize_prompt_messages()`

```python
@override
def _organize_prompt_messages(self) -> list[PromptMessage]:
    system_message = self._organize_system_prompt()

    agent_scratchpad = self._agent_scratchpad
    if not agent_scratchpad:
        assistant_messages = []
    else:
        content = ""
        for unit in agent_scratchpad:
            if unit.is_final():
                content += f"Final Answer: {unit.agent_response}"
            else:
                content += f"Thought: {unit.thought}\n\n"
                if unit.action_str:
                    content += f"Action: {unit.action_str}\n\n"
                if unit.observation:
                    content += f"Observation: {unit.observation}\n\n"
        assistant_messages = [AssistantPromptMessage(content=content)]

    query_messages = self._organize_user_query(self._query, [])

    if assistant_messages:
        historic_messages = self._organize_historic_prompt_messages(
            [system_message, *query_messages, *assistant_messages, UserPromptMessage(content="continue")]
        )
        messages = [
            system_message,
            *historic_messages,
            *query_messages,
            *assistant_messages,
            UserPromptMessage(content="continue"),
        ]
    else:
        historic_messages = self._organize_historic_prompt_messages([system_message, *query_messages])
        messages = [system_message, *historic_messages, *query_messages]

    return messages
```

- **作用**：组装 Chat 模式下的完整消息列表
- **消息结构**（有 scratchpad 时）：
  ```
  [SystemPromptMessage]           <- 系统提示（含工具描述）
  [historic_messages...]          <- 历史对话
  [UserPromptMessage]             <- 当前用户查询
  [AssistantPromptMessage]        <- Agent 的推理过程（Thought/Action/Observation）
  [UserPromptMessage("continue")] <- 引导模型继续推理
  ```
- **关键设计**：
  - `UserPromptMessage(content="continue")` 是一个"伪用户消息"，用于在 Chat 模式下引导模型继续输出
  - 如果没有 scratchpad（第一轮），不需要 assistant_messages 和 continue 消息
  - 历史消息通过 `_organize_historic_prompt_messages()` 处理，考虑 token 预算

---

**4. 补充说明**：
- Chat 模式的消息结构遵循 ChatML 格式，每条消息有明确的角色（system/user/assistant）
- `jsonable_encoder` 用于处理 Pydantic 模型到 JSON 的转换，确保所有类型都可序列化
- `UserPromptMessage(content="continue")` 是 Chat 模式特有的设计，因为 Chat API 要求消息角色交替出现，不能连续出现两个 assistant 消息
