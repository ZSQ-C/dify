# strategy/base.py 详细讲解

**1. 文件定位**：Agent 策略的**抽象基类**，定义了所有 Agent 策略（包括内置策略和插件策略）的统一接口。被 `strategy/plugin.py` 继承，也被插件管理模块引用。

**2. 代码结构**：包含 1 个类 `BaseAgentStrategy`，内含 3 个方法

| 方法名 | 作用 |
|---|---|
| `invoke()` | 策略的公共调用入口（模板方法） |
| `get_parameters()` | 获取策略的参数定义（默认返回空列表） |
| `_invoke()` | 策略的实际执行逻辑（抽象方法） |

---

**3. 执行流程详解**：

### `BaseAgentStrategy` 类定义

```python
class BaseAgentStrategy(ABC):
    """
    Agent Strategy
    """
```

- **作用**：定义 Agent 策略的抽象基类
- **继承**：`ABC`（Abstract Base Class），不能直接实例化
- **设计意图**：为所有 Agent 策略提供统一的调用接口，实现策略模式

---

### `invoke()`

```python
def invoke(
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
    yield from self._invoke(params, user_id, conversation_id, app_id, message_id, credentials)
```

- **作用**：策略的公共调用入口，实现模板方法模式
- **入参**：
  - `params: dict[str, Any]` -- 策略参数（键值对，键对应 `AgentStrategyParameter.name`）
  - `user_id: str` -- 用户 ID
  - `conversation_id: str | None = None` -- 对话 ID
  - `app_id: str | None = None` -- 应用 ID
  - `message_id: str | None = None` -- 消息 ID
  - `credentials: InvokeCredentials | None = None` -- 调用凭证
- **出参**：`Generator[AgentInvokeMessage, None, None]` -- 流式输出的 Agent 调用消息
- **内部执行步骤**：
  1. 直接委托给 `_invoke()` 抽象方法
  2. 使用 `yield from` 将子生成器的输出透传给调用方
- **设计意图**：
  - `invoke()` 是公共接口，未来可以在此添加日志、鉴权、限流等横切关注点
  - `_invoke()` 是受保护的实现方法，由子类实现具体逻辑
  - 这种分离保证了接口的稳定性和实现的灵活性

---

### `get_parameters()`

```python
def get_parameters(self) -> Sequence[AgentStrategyParameter]:
    """
    Get the parameters for the agent strategy.
    """
    return []
```

- **作用**：获取策略的参数定义列表
- **出参**：`Sequence[AgentStrategyParameter]` -- 参数定义序列
- **默认实现**：返回空列表
- **设计意图**：内置策略可能不需要参数，因此提供空列表的默认实现；插件策略需要覆盖此方法返回实际的参数定义

---

### `_invoke()` (抽象方法)

```python
@abstractmethod
def _invoke(
    self,
    params: dict[str, Any],
    user_id: str,
    conversation_id: str | None = None,
    app_id: str | None = None,
    message_id: str | None = None,
    credentials: InvokeCredentials | None = None,
) -> Generator[AgentInvokeMessage, None, None]:
    pass
```

- **作用**：策略的实际执行逻辑，由子类实现
- **入参**：与 `invoke()` 完全相同
- **出参**：`Generator[AgentInvokeMessage, None, None]` -- 流式输出的 Agent 调用消息
- **设计意图**：抽象方法强制子类必须实现此方法，确保策略的完整性

---

**4. 补充说明**：
- `BaseAgentStrategy` 与 `BaseAgentRunner` 是两套并行的体系：
  - `BaseAgentRunner` 是内置策略（FC 和 CoT）的运行器基类
  - `BaseAgentStrategy` 是插件策略的策略基类
- 两者不共享继承关系，但共享 `AgentInvokeMessage` 作为输出类型
- `invoke()` / `_invoke()` 的分离是经典的模板方法模式，为未来扩展预留了空间
- `credentials` 参数支持插件级别的认证，允许插件策略使用自己的 API 密钥
