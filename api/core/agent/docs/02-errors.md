# errors.py 详细讲解

**1. 文件定位**：Agent 模块的**异常定义层**，定义了 Agent 运行过程中可能抛出的领域特定异常。被 `fc_agent_runner.py` 和 `cot_agent_runner.py` 引用。

**2. 代码结构**：包含 1 个类

---

**3. 执行流程详解**：

### `AgentMaxIterationError`

```python
class AgentMaxIterationError(Exception):
    """Raised when an agent runner exceeds the configured max iteration count."""

    def __init__(self, max_iteration: int):
        self.max_iteration = max_iteration
        super().__init__(
            f"Agent exceeded the maximum iteration limit of {max_iteration}. "
            f"The agent was unable to complete the task within the allowed number of iterations."
        )
```

- **作用**：当 Agent 的迭代次数超过配置的最大值时抛出此异常
- **入参**：
  - `max_iteration: int` -- 配置的最大迭代次数
- **内部执行步骤**：
  1. 将 `max_iteration` 保存为实例属性 `self.max_iteration`，方便调用方获取配置值
  2. 调用 `super().__init__()` 构造异常消息，消息格式为：`Agent exceeded the maximum iteration limit of {max_iteration}. The agent was unable to complete the task within the allowed number of iterations.`
- **触发场景**：
  - 在 `FunctionCallAgentRunner.run()` 中：当 `iteration_step == max_iteration_steps` 且 LLM 仍然返回了 tool_calls 时抛出
  - 在 `CotAgentRunner.run()` 中：当 `iteration_step == max_iteration_steps` 且解析出的 Action 不是 "Final Answer" 时抛出
- **设计意图**：防止 Agent 陷入无限循环，当模型持续要求调用工具而无法得出最终答案时，强制终止并报告错误

---

**4. 补充说明**：
- 此异常是 Agent 模块唯一的自定义异常，体现了"快速失败"的设计原则
- 调用方可以捕获此异常并向用户展示友好的错误提示，而非让 Agent 无限运行
- `self.max_iteration` 属性允许调用方在捕获异常后获取具体的迭代上限值，用于错误报告
