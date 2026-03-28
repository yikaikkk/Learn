# Agent分层
## 1. Agent层（智能体层）
职责 ：

- 任务协调与管理 ：负责接收用户输入，协调任务执行流程，管理执行状态
- 工具集成 ：整合各种工具，根据任务需求调用合适的工具
- 状态追踪 ：跟踪任务执行进度，维护对话历史和上下文
- 决策与控制 ：决定何时停止执行，何时调用工具，何时返回最终结果
核心类 ：

- BaseAgent ：所有Agent的基类，定义了核心执行逻辑
- AssistantAgent ：直接思考型Agent，支持技能和工具集成
- ToolCallingAgent ：专注于工具调用的Agent
- CodeAgent ：专注于代码执行的的Agent
工作流程 ：

1. 接收用户输入，初始化执行状态
2. 循环执行步骤，直到任务完成或达到最大步数
3. 每步执行中，调用Brain层处理输入
4. 根据执行结果更新状态，决定下一步操作
5. 任务完成时，返回最终结果
## 2. Brain层（大脑层）
职责 ：

- LLM管理 ：管理与大语言模型的交互，处理模型调用
- 工具管理 ：提供工具列表给LLM，处理工具调用请求
- 记忆管理 ：维护系统的记忆，支持上下文理解
- 思维管理 ：通过左右脑（left_mind、right_mind）实现不同类型的思考
核心类 ：

- Brain ：核心大脑类，管理minds和工具
- Mind ：思维模块，负责特定类型的思考
工作流程 ：

1. 接收Agent的输入，选择合适的Mind处理
2. 调用LLM生成响应，处理工具调用请求
3. 执行工具并获取结果，将结果返回给LLM
4. 处理LLM的最终响应，返回给Agent
## 3. Minion Worker层（执行层）
职责 ：

- 具体任务执行 ：根据路由选择，执行特定类型的任务
- 输入处理 ：处理不同格式的输入，如文本、消息列表等
- 输出生成 ：生成相应的输出，支持流式和非流式输出
- 工具调用 ：执行工具调用，处理工具执行结果
核心类 ：

- WorkerMinion ：所有Worker的基类
- RawMinion ：直接查询LLM的Worker，支持工具调用
- CotMinion ：链式思考Worker，支持逐步推理
- PlanMinion ：规划型Worker，支持任务分解
- PythonMinion ：代码执行Worker，支持Python代码运行
- ModeratorMinion ：协调型Worker，负责选择和管理其他Worker
- RouteMinion ：路由型Worker，负责根据输入选择合适的Worker
工作流程 ：

1. 接收Brain的输入，根据自身类型处理
2. 生成响应，可能包含工具调用请求
3. 执行工具调用，获取工具执行结果
4. 处理工具结果，生成最终响应
5. 将响应返回给Brain
## 层次间的协作关系
1. 用户 → Agent层 ：用户发送请求给Agent
2. Agent层 → Brain层 ：Agent调用Brain处理输入
3. Brain层 → Minion Worker层 ：Brain通过ModeratorMinion选择合适的Worker执行任务
4. Minion Worker层 → 工具 ：Worker执行工具调用，获取结果
5. Minion Worker层 → Brain层 ：Worker将执行结果返回给Brain
6. Brain层 → Agent层 ：Brain将处理结果返回给Agent
7. Agent层 → 用户 ：Agent将最终结果返回给用户
## 设计优势
1. 模块化 ：各层次职责明确，便于维护和扩展
2. 灵活性 ：通过路由机制，可以根据任务类型选择合适的Worker
3. 可扩展性 ：可以轻松添加新的Agent、Brain功能或Worker类型
4. 流式处理 ：支持流式输出，提升用户体验
5. 工具集成 ：统一的工具调用机制，便于集成各种外部工具