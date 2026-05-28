# template.py 详细讲解

**1. 文件定位**：定义了 ReAct 模式的 **Prompt 模板**，为 CoT 策略提供标准化的指令文本。被 `cot_chat_agent_runner.py` 和 `cot_completion_agent_runner.py` 间接引用（通过应用配置传入）。

**2. 代码结构**：包含 4 个模板常量和 1 个模板字典

| 常量名 | 作用 |
|---|---|
| `ENGLISH_REACT_COMPLETION_PROMPT_TEMPLATES` | Completion 模式的 ReAct Prompt 模板 |
| `ENGLISH_REACT_COMPLETION_AGENT_SCRATCHPAD_TEMPLATES` | Completion 模式的后续轮次模板 |
| `ENGLISH_REACT_CHAT_PROMPT_TEMPLATES` | Chat 模式的 ReAct Prompt 模板 |
| `ENGLISH_REACT_CHAT_AGENT_SCRATCHPAD_TEMPLATES` | Chat 模式的后续轮次模板（空字符串） |
| `REACT_PROMPT_TEMPLATES` | 模板注册字典 |

---

**3. 执行流程详解**：

### `ENGLISH_REACT_COMPLETION_PROMPT_TEMPLATES`

```python
ENGLISH_REACT_COMPLETION_PROMPT_TEMPLATES = """Respond to the human as helpfully and accurately as possible.

{{instruction}}

You have access to the following tools:

{{tools}}

Use a json blob to specify a tool by providing an action key (tool name) and an action_input key (tool input).
Valid "action" values: "Final Answer" or {{tool_names}}

Provide only ONE action per $JSON_BLOB, as shown:

```
{
  "action": $TOOL_NAME,
  "action_input": $ACTION_INPUT
}
```

Follow this format:

Question: input question to answer
Thought: consider previous and subsequent steps
Action:
```
$JSON_BLOB
```
Observation: action result
... (repeat Thought/Action/Observation N times)
Thought: I know what to respond
Action:
```
{
  "action": "Final Answer",
  "action_input": "Final response to human"
}
```

Begin! Reminder to ALWAYS respond with a valid json blob of a single action. Use tools if necessary. Respond directly if appropriate. Format is Action:```$JSON_BLOB```then Observation:.
{{historic_messages}}
Question: {{query}}
{{agent_scratchpad}}
Thought:"""
```

- **作用**：Completion 模式下的完整 ReAct Prompt 模板
- **模板结构分析**：
  1. **角色定义**：`Respond to the human as helpfully and accurately as possible.` -- 定义 Agent 的行为目标
  2. **指令注入**：`{{instruction}}` -- 用户自定义的指令文本
  3. **工具描述**：`{{tools}}` -- 所有可用工具的 JSON Schema 描述
  4. **工具名称列表**：`{{tool_names}}` -- 有效的 action 值列表（含 "Final Answer"）
  5. **格式说明**：详细说明 JSON blob 的格式要求
  6. **示例**：展示完整的 Thought/Action/Observation 循环示例
  7. **提醒**：强调必须返回有效的 JSON blob
  8. **历史消息**：`{{historic_messages}}` -- 之前的对话历史
  9. **当前查询**：`{{query}}` -- 当前用户的问题
  10. **推理暂存**：`{{agent_scratchpad}}` -- 之前的推理过程
  11. **引导词**：`Thought:` -- 模板末尾的引导词，让 LLM 直接开始思考

- **占位符列表**：
  - `{{instruction}}` -- 用户指令
  - `{{tools}}` -- 工具 JSON Schema
  - `{{tool_names}}` -- 工具名称列表
  - `{{historic_messages}}` -- 历史对话
  - `{{query}}` -- 当前查询
  - `{{agent_scratchpad}}` -- 推理暂存区

---

### `ENGLISH_REACT_COMPLETION_AGENT_SCRATCHPAD_TEMPLATES`

```python
ENGLISH_REACT_COMPLETION_AGENT_SCRATCHPAD_TEMPLATES = """Observation: {{observation}}
Thought:"""
```

- **作用**：Completion 模式下后续轮次使用的模板
- **逻辑**：在工具执行完成后，将 Observation 结果追加到 Prompt 中，并添加 `Thought:` 引导词让 LLM 继续推理
- **占位符**：`{{observation}}` -- 工具执行的观测结果

---

### `ENGLISH_REACT_CHAT_PROMPT_TEMPLATES`

```python
ENGLISH_REACT_CHAT_PROMPT_TEMPLATES = """Respond to the human as helpfully and accurately as possible.

{{instruction}}

You have access to the following tools:

{{tools}}

Use a json blob to specify a tool by providing an action key (tool name) and an action_input key (tool input).
Valid "action" values: "Final Answer" or {{tool_names}}

Provide only ONE action per $JSON_BLOB, as shown:

```
{
  "action": $TOOL_NAME,
  "action_input": $ACTION_INPUT
}
```

Follow this format:

Question: input question to answer
Thought: consider previous and subsequent steps
Action:
```
$JSON_BLOB
```
Observation: action result
... (repeat Thought/Action/Observation N times)
Thought: I know what to respond
Action:
```
{
  "action": "Final Answer",
  "action_input": "Final response to human"
}
```

Begin! Reminder to ALWAYS respond with a valid json blob of a single action. Use tools if necessary. Respond directly if appropriate. Format is Action:```$JSON_BLOB```then Observation:.
"""
```

- **作用**：Chat 模式下的 ReAct Prompt 模板
- **与 Completion 模板的区别**：
  - **没有** `{{historic_messages}}`、`{{query}}`、`{{agent_scratchpad}}` 占位符
  - **没有** 末尾的 `Thought:` 引导词
  - 因为 Chat 模式通过消息角色（system/user/assistant）来组织内容，不需要在模板中拼接

---

### `ENGLISH_REACT_CHAT_AGENT_SCRATCHPAD_TEMPLATES`

```python
ENGLISH_REACT_CHAT_AGENT_SCRATCHPAD_TEMPLATES = ""
```

- **作用**：Chat 模式下后续轮次使用的模板
- **值为空字符串**：因为 Chat 模式下，后续轮次的 Observation 和 Thought 通过 `AssistantPromptMessage` 和 `UserPromptMessage("continue")` 来传递，不需要额外的模板

---

### `REACT_PROMPT_TEMPLATES`

```python
REACT_PROMPT_TEMPLATES = {
    "english": {
        "chat": {
            "prompt": ENGLISH_REACT_CHAT_PROMPT_TEMPLATES,
            "agent_scratchpad": ENGLISH_REACT_CHAT_AGENT_SCRATCHPAD_TEMPLATES,
        },
        "completion": {
            "prompt": ENGLISH_REACT_COMPLETION_PROMPT_TEMPLATES,
            "agent_scratchpad": ENGLISH_REACT_COMPLETION_AGENT_SCRATCHPAD_TEMPLATES,
        },
    }
}
```

- **作用**：模板注册字典，按语言和模式组织所有模板
- **结构**：`语言 -> 模式 -> (prompt, agent_scratchpad)`
- **当前只支持英语**：`"english"` 是唯一的语言键
- **扩展性**：可以添加其他语言的模板（如 `"chinese"`）

---

**4. 补充说明**：
- Chat 模式和 Completion 模式的模板差异体现了两种 API 的本质区别：Chat 通过消息角色组织，Completion 通过文本拼接组织
- `Thought:` 引导词在 Completion 模板末尾出现，是因为 Completion API 会在模板后直接生成内容，需要引导 LLM 开始思考
- 模板中的 `$JSON_BLOB` 和 `$TOOL_NAME`、`$ACTION_INPUT` 是示例占位符（非实际替换），用于向 LLM 说明格式
- `{{tool_names}}` 的值格式为 `"web_search", "calculator", "Final Answer"`，让 LLM 知道合法的 action 名称
