# cot_completion_agent_runner.py 详细讲解

**1. 文件定位**：实现 CoT 策略在 **Completion 补全模式**下的具体 Prompt 组织方式。继承自 `CotAgentRunner`，实现了 `_organize_prompt_messages()` 抽象方法。

**2. 代码结构**：包含 1 个类 `CotCompletionAgentRunner`，内含 3 个方法

| 方法名 | 作用 |
|---|---|
| `_organize_instruction_prompt()` | 组织指令提示文本 |
| `_organize_historic_prompt()` | 将历史消息格式化为纯文本 |
| `_organize_prompt_messages()` | 实现抽象方法，组装单条消息 |

---

**3. 执行流程详解**：

### `_organize_instruction_prompt()`

```python
def _organize_instruction_prompt(self) -> str:
    if self.app_config.agent is None:
        raise ValueError("Agent configuration is not set")
    prompt_entity = self.app_config.agent.prompt
    if prompt_entity is None:
        raise ValueError("prompt entity is not set")
    first_prompt = prompt_entity.first_prompt

    system_prompt = (
        first_prompt.replace("{{instruction}}", self._instruction)
        .replace("{{tools}}", json.dumps(jsonable_encoder(self._prompt_messages_tools)))
        .replace("{{tool_names}}", ", ".join([tool.name for tool in self._prompt_messages_tools]))
    )
    return system_prompt
```

- **作用**：构建指令提示文本（与 Chat 模式的 `_organize_system_prompt()` 类似，但返回字符串而非 SystemPromptMessage）
- **差异**：Completion 模式下没有独立的 system 角色，所有内容都拼接到一个字符串中
- **与 Chat 模式的区别**：
  - Chat 模式返回 `SystemPromptMessage` 对象
  - Completion 模式返回纯字符串
  - Completion 模式增加了 `ValueError` 校验（agent 和 prompt 不能为 None）

---

### `_organize_historic_prompt()`

```python
def _organize_historic_prompt(self, current_session_messages: list[PromptMessage] | None = None) -> str:
    historic_prompt_messages = self._organize_historic_prompt_messages(current_session_messages)
    historic_prompt = ""

    for message in historic_prompt_messages:
        if isinstance(message, UserPromptMessage):
            historic_prompt += f"Question: {message.content}\n\n"
        elif isinstance(message, AssistantPromptMessage):
            if isinstance(message.content, str):
                historic_prompt += message.content + "\n\n"
            elif isinstance(message.content, list):
                for content in message.content:
                    if not isinstance(content, TextPromptMessageContent):
                        continue
                    historic_prompt += content.data

    return historic_prompt
```

- **作用**：将结构化的历史消息列表转换为纯文本格式
- **格式**：
  - UserPromptMessage -> `Question: {content}\n\n`
  - AssistantPromptMessage -> `{content}\n\n`
- **设计意图**：Completion 模式只有一个输入文本框，需要将所有历史对话压缩为纯文本
- **处理多模态内容**：如果 Assistant 消息的 content 是列表（多模态），只提取 `TextPromptMessageContent` 类型的内容

---

### `_organize_prompt_messages()`

```python
@override
def _organize_prompt_messages(self) -> list[PromptMessage]:
    system_prompt = self._organize_instruction_prompt()
    historic_prompt = self._organize_historic_prompt()

    agent_scratchpad = self._agent_scratchpad
    assistant_prompt = ""
    for unit in agent_scratchpad or []:
        if unit.is_final():
            assistant_prompt += f"Final Answer: {unit.agent_response}"
        else:
            assistant_prompt += f"Thought: {unit.thought}\n\n"
            if unit.action_str:
                assistant_prompt += f"Action: {unit.action_str}\n\n"
            if unit.observation:
                assistant_prompt += f"Observation: {unit.observation}\n\n"

    query_prompt = f"Question: {self._query}"

    prompt = (
        system_prompt.replace("{{historic_messages}}", historic_prompt)
        .replace("{{agent_scratchpad}}", assistant_prompt)
        .replace("{{query}}", query_prompt)
    )

    return [UserPromptMessage(content=prompt)]
```

- **作用**：组装 Completion 模式下的完整消息列表
- **消息结构**：只有一条 `UserPromptMessage`，内容是所有文本的拼接
- **替换的占位符**：
  - `{{historic_messages}}` -> 历史对话的纯文本
  - `{{agent_scratchpad}}` -> 当前推理过程的纯文本
  - `{{query}}` -> `Question: {query}`
- **最终 Prompt 结构**：
  ```
  [系统指令 + 工具描述]
  [历史对话文本]
  Question: [用户查询]
  Thought: [思考过程]
  Action: [动作]
  Observation: [观测结果]
  ...
  Thought:
  ```

---

**4. 补充说明**：
- Completion 模式与 Chat 模式的根本区别：Completion 将所有内容拼接到一条消息中，Chat 保持多消息结构
- Completion 模式使用 `{{historic_messages}}`、`{{agent_scratchpad}}`、`{{query}}` 三个额外占位符，这些在 Chat 模式中不需要（因为 Chat 模式通过消息角色区分）
- Completion 模式不支持多模态文件输入（没有 `_organize_user_query()` 中的文件处理逻辑）
- Completion 模式的 Prompt 模板末尾有 `Thought:` 引导词，让 LLM 直接开始思考
