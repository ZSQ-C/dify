# strategy/plugin.py 详细讲解

**1. 文件定位**：实现**插件化 Agent 策略**，通过远程调用插件提供的 Agent 策略来执行任务。继承自 `BaseAgentStrategy`，是插件策略体系的核心实现。

**2. 代码结构**：包含 1 个类 `PluginAgentStrategy`，内含 4 个方法

| 方法名 | 作用 |
|---|---|
| `__init__()` | 初始化插件策略 |
| `get_parameters()` | 获取策略参数定义 |
| `initialize_parameters()` | 初始化参数默认值 |
| `_invoke()` | 执行插件策略 |

---

**3. 执行流程详解**：

### `PluginAgentStrategy` 类定义

```python
class PluginAgentStrategy(BaseAgentStrategy):
    """
    Agent Strategy
    """

    tenant_id: str
    declaration: AgentStrategyEntity
    meta_version: str | None = None
```

- **作用**：插件化 Agent 策略的实现类
- **类属性**：
  - `tenant_id: str` -- 租户 ID，用于多租户隔离和插件调用鉴权
  - `declaration: AgentStrategyEntity` -- 策略声明实体，包含策略的元数据、参数定义、描述等
  - `meta_version: str | None = None` -- 元数据版本号

---

### `__init__()`

```python
def __init__(self, tenant_id: str, declaration: AgentStrategyEntity, meta_version: str | None):
    self.tenant_id = tenant_id
    self.declaration = declaration
    self.meta_version = meta_version
```

- **作用**：初始化插件策略实例
- **入参**：
  - `tenant_id: str` -- 租户 ID
  - `declaration: AgentStrategyEntity` -- 策略声明实体
  - `meta_version: str | None` -- 元数据版本号
- **内部执行步骤**：
  1. 保存 `tenant_id` 到实例属性
  2. 保存 `declaration` 到实例属性（包含策略的身份、参数、描述、特性等完整信息）
  3. 保存 `meta_version` 到实例属性

---

### `get_parameters()`

```python
@override
def get_parameters(self) -> Sequence[AgentStrategyParameter]:
    return self.declaration.parameters
```

- **作用**：获取策略的参数定义列表
- **出参**：`Sequence[AgentStrategyParameter]` -- 从策略声明中获取的参数列表
- **设计意图**：覆盖基类的默认实现（返回空列表），返回插件声明的实际参数定义

---

### `initialize_parameters()`

```python
def initialize_parameters(self, params: dict[str, Any]) -> dict[str, Any]:
    """
    Initialize the parameters for the agent strategy.
    """
    for parameter in self.declaration.parameters:
        params[parameter.name] = parameter.init_frontend_parameter(params.get(parameter.name))
    return params
```

- **作用**：初始化策略参数的默认值
- **入参**：`params: dict[str, Any]` -- 原始参数字典
- **出参**：`dict[str, Any]` -- 初始化后的参数字典
- **内部执行步骤**：
  1. 遍历策略声明中的所有参数定义
  2. 对每个参数，调用 `init_frontend_parameter()` 方法：
     - 如果 `params.get(parameter.name)` 为 None（调用方未提供该参数），则使用参数的默认值
     - 如果调用方已提供该参数，则保留原值
  3. 将初始化后的值写回 `params` 字典
  4. 返回更新后的参数字典
- **设计意图**：确保所有参数都有有效值，即使调用方没有显式提供

---

### `_invoke()`

```python
@override
def _invoke(
    self,
    params: dict[str, Any],
    user_id: str,
    conversation_id: str | None = None,
    app_id: str | None = None,
    message_id: str | None = None,
    credentials: InvokeCredentials | None = None,
) -> Generator[AgentInvokeMessage, None, None]:
    """
    Invoke the agent strategy.
    """
    manager = PluginAgentClient()

    initialized_params = self.initialize_parameters(params)
    params = convert_parameters_to_plugin_format(initialized_params)

    yield from manager.invoke(
        tenant_id=self.tenant_id,
        user_id=user_id,
        agent_provider=self.declaration.identity.provider,
        agent_strategy=self.declaration.identity.name,
        agent_params=params,
        conversation_id=conversation_id,
        app_id=app_id,
        message_id=message_id,
        context=PluginInvokeContext(credentials=credentials or InvokeCredentials()),
    )
```

- **作用**：执行插件策略，通过 `PluginAgentClient` 远程调用插件提供的 Agent 策略
- **入参**：与基类 `_invoke()` 相同
- **出参**：`Generator[AgentInvokeMessage, None, None]` -- 流式输出的 Agent 调用消息
- **内部执行步骤**：
  1. **创建插件客户端**：`manager = PluginAgentClient()` -- 创建插件 Agent 调用的客户端实例
  2. **初始化参数**：`initialized_params = self.initialize_parameters(params)` -- 确保所有参数都有有效值
  3. **转换参数格式**：`params = convert_parameters_to_plugin_format(initialized_params)` -- 将参数从内部格式转换为插件可识别的格式
  4. **远程调用**：`manager.invoke()` -- 通过插件客户端发起远程调用
     - `tenant_id` -- 租户 ID
     - `user_id` -- 用户 ID
     - `agent_provider` -- 策略提供方名称（从声明中获取）
     - `agent_strategy` -- 策略名称（从声明中获取）
     - `agent_params` -- 转换后的参数
     - `conversation_id` -- 对话 ID
     - `app_id` -- 应用 ID
     - `message_id` -- 消息 ID
     - `context` -- 插件调用上下文（含凭证信息）
  5. **流式输出**：`yield from` -- 将远程调用的结果透传给调用方

---

**4. 补充说明**：
- `PluginAgentStrategy` 是插件化 Agent 策略的唯一实现，它将策略执行委托给远程插件
- 与内置策略（FC 和 CoT）的核心区别：
  - 内置策略在本地执行推理循环，直接调用 LLM 和工具
  - 插件策略将推理逻辑完全委托给远程插件，本地只负责参数初始化和格式转换
- `convert_parameters_to_plugin_format()` 负责将 Pydantic 模型参数转换为插件可识别的 JSON 格式
- `PluginInvokeContext` 封装了调用上下文，包括凭证信息，确保插件可以在正确的权限下执行
- 插件策略的流式输出特性（`yield from`）保证了与内置策略一致的调用体验
