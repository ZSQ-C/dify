# cot_output_parser.py 详细讲解

**1. 文件定位**：ReAct 模式的**输出流解析器**，负责从 LLM 的流式文本输出中逐字符解析出 `Thought` 和 `Action` 信息。被 `cot_agent_runner.py` 引用。

**2. 代码结构**：包含 1 个类 `CotAgentOutputParser`，内含 1 个类方法 `handle_react_stream_output()` 和 2 个内部函数

| 函数名 | 作用 |
|---|---|
| `parse_action()` | 解析 Action JSON 为 AgentScratchpadUnit.Action |
| `extra_json_from_code_block()` | 从代码块中提取 JSON |
| `handle_react_stream_output()` | 主解析方法，处理流式输出 |

---

**3. 执行流程详解**：

### `parse_action()` (内部函数)

```python
def parse_action(action) -> Union[str, AgentScratchpadUnit.Action]:
    action_name = None
    action_input = None
    if isinstance(action, str):
        try:
            action = json.loads(action, strict=False)
        except json.JSONDecodeError:
            return action or ""

    # cohere always returns a list
    if isinstance(action, list) and len(action) == 1:
        action = action[0]

    for key, value in action.items():
        if "input" in key.lower():
            action_input = value
        else:
            action_name = value

    if action_name is not None and action_input is not None:
        return AgentScratchpadUnit.Action(
            action_name=action_name,
            action_input=action_input,
        )
    else:
        return json.dumps(action)
```

- **作用**：将解析出的 JSON 对象转换为 `AgentScratchpadUnit.Action` 结构化对象
- **入参**：`action` -- 可以是字符串、字典或列表
- **出参**：成功时返回 `AgentScratchpadUnit.Action`，失败时返回原始字符串
- **内部执行步骤**：
  1. 如果输入是字符串，尝试解析为 JSON；如果解析失败，返回原始字符串
  2. **Cohere 兼容**：如果 action 是列表且只有一个元素，取第一个元素（Cohere 模型总是返回列表格式）
  3. 遍历字典的 key-value 对：
     - 如果 key 包含 "input"（不区分大小写），则 value 是 `action_input`
     - 否则 value 是 `action_name`
  4. 如果 `action_name` 和 `action_input` 都不为 None，构造并返回 `AgentScratchpadUnit.Action`
  5. 否则将原始字典序列化为 JSON 字符串返回

---

### `extra_json_from_code_block()` (内部函数)

```python
def extra_json_from_code_block(code_block) -> list[Union[list, dict]]:
    blocks = re.findall(r"```[json]*\s*([\[{].*[]}])\s*```", code_block, re.DOTALL | re.IGNORECASE)
    if not blocks:
        return []
    try:
        json_blocks = []
        for block in blocks:
            json_text = re.sub(r"^[a-zA-Z]+\n", "", block.strip(), flags=re.MULTILINE)
            json_blocks.append(json.loads(json_text, strict=False))
        return json_blocks
    except:
        return []
```

- **作用**：从 Markdown 代码块中提取 JSON 对象
- **入参**：`code_block` -- 包含代码块的文本
- **出参**：解析出的 JSON 对象列表
- **内部执行步骤**：
  1. 使用正则表达式匹配 `` ```json {...} ``` `` 或 `` ``` [...] ``` `` 格式的代码块
  2. 正则 `r"```[json]*\s*([\[{].*[]}])\s*```"` 解释：
     - `` ```[json]* `` -- 匹配三个反引号开头，可选的 "json" 标记
     - `\s*` -- 可选空白
     - `([\[{].*[]}])` -- 捕获组，匹配以 `{` 或 `[` 开头、以 `}` 或 `]` 结尾的内容
     - `\s*``` ` -- 可选空白加三个反引号结尾
  3. 如果没有匹配到，返回空列表
  4. 对每个匹配的代码块：
     - 移除可能的语言标识行（如 `json\n`）
     - 解析 JSON
  5. 返回所有解析出的 JSON 对象列表

---

### `handle_react_stream_output()` (类方法)

```python
@classmethod
def handle_react_stream_output(
    cls, llm_response: Generator[LLMResultChunk, None, None], usage_dict: dict[str, Any]
) -> Generator[Union[str, AgentScratchpadUnit.Action], None, None]:
```

- **作用**：主解析方法，逐字符处理 LLM 的流式输出，识别 Thought、Action 和 JSON 块
- **入参**：
  - `llm_response` -- LLM 的流式响应生成器
  - `usage_dict` -- 用于累积 LLM 使用量的字典
- **出参**：`Generator[Union[str, AgentScratchpadUnit.Action], None, None]` -- 交替输出文本片段和解析出的 Action

- **状态变量**：

| 变量 | 初始值 | 作用 |
|---|---|---|
| `code_block_cache` | `""` | 缓存代码块内容 |
| `code_block_delimiter_count` | `0` | 连续反引号计数器 |
| `in_code_block` | `False` | 是否在代码块内 |
| `json_cache` | `""` | 缓存行内 JSON 内容 |
| `json_quote_count` | `0` | 花括号嵌套计数器 |
| `in_json` | `False` | 是否在行内 JSON 内 |
| `got_json` | `False` | 是否完整获取了一个 JSON |
| `action_cache` | `""` | 缓存 "action:" 前缀的字符 |
| `action_str` | `"action:"` | 要匹配的 "action:" 字符串 |
| `action_idx` | `0` | 当前匹配到 "action:" 的第几个字符 |
| `thought_cache` | `""` | 缓存 "thought:" 前缀的字符 |
| `thought_str` | `"thought:"` | 要匹配的 "thought:" 字符串 |
| `thought_idx` | `0` | 当前匹配到 "thought:" 的第几个字符 |
| `last_character` | `""` | 上一个处理的字符 |

- **主循环逻辑**（逐字符处理）：

  1. **提取使用量**：如果 chunk 包含 usage 信息，更新 `usage_dict`

  2. **跳过非字符串内容**：如果 `response_content` 不是字符串，跳过

  3. **逐字符遍历** `response_content`：

     a. **反引号检测**：
        - 如果不在 JSON 内且遇到反引号 `` ` ``：累加到 `code_block_cache`，递增 `code_block_delimiter_count`
        - 如果不在代码块内且有累积的反引号：先 `yield` 缓存内容，然后重置
        - 如果在代码块内：将字符追加到 `code_block_cache`

     b. **"action:" 前缀匹配**（不在代码块和 JSON 内时）：
        - 逐字符匹配 `action_str`（"action:"）
        - 如果匹配成功且是第一个字符：检查 `last_character` 是否是换行/空格/空（确保是行首）
        - 如果匹配成功且不是第一个字符：继续匹配下一个字符
        - 如果匹配完成（`action_idx == len(action_str)`）：重置缓存，跳过这些字符（不 yield）
        - 如果匹配失败：`yield` 已缓存的字符，重置匹配状态

     c. **"thought:" 前缀匹配**（不在代码块和 JSON 内时）：
        - 逻辑与 "action:" 匹配完全相同
        - 匹配成功后跳过这些字符（不 yield），因为 Thought 标签不需要传给调用方

     d. **代码块结束检测**：
        - 如果 `code_block_delimiter_count == 3`（三个反引号）：
          - 如果已在代码块内：提取 JSON 并 `yield parse_action(action_json)`
          - 切换 `in_code_block` 状态

     e. **行内 JSON 检测**（不在代码块内时）：
        - 遇到 `{`：递增 `json_quote_count`，进入 JSON 模式
        - 遇到 `}`：递减 `json_quote_count`，如果归零则标记 `got_json = True`
        - 在 JSON 内：将字符追加到 `json_cache`
        - 完整获取 JSON 后：`yield parse_action(json_cache)`，重置 JSON 状态

     f. **普通文本输出**（不在代码块和 JSON 内时）：
        - `yield delta.replace("`", "")` -- 输出字符，同时移除残留的反引号

  4. **处理残留缓存**：
     - 如果 `code_block_cache` 非空：`yield code_block_cache`
     - 如果 `json_cache` 非空：`yield parse_action(json_cache)`

---

**4. 补充说明**：
- 这个解析器是一个**有限状态机（FSM）**，通过多个布尔标志和计数器跟踪当前解析状态
- 它需要处理三种 JSON 输出格式：代码块格式（`` ```json{...}``` ``）、行内格式（`{...}`）、纯文本格式
- "action:" 和 "thought:" 前缀匹配的目的是**过滤掉这些标签**，只输出实际内容
- Cohere 模型的列表返回格式通过 `parse_action()` 中的列表解包逻辑兼容
- `strict=False` 参数在 `json.loads()` 中允许控制字符出现在 JSON 字符串中
