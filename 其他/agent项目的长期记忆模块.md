# 1. 长期记忆（mem0）
## 用 user_id / agent_id / run_id 做作用域
所有读写都带上当前 BaseAgent 上的三个字段（run_id 对应 session_id）：

写入 add_memory：brain.mem.add(..., user_id=, agent_id=, run_id=)

读取 get_all_memories / search_memories / get_conversation_history：同样传入这三项

也就是说：隔离语义主要由 mem0 按这些维度过滤/分区；minion 负责始终把「这个 Agent 实例」的 ID 绑在每次调用上。

```python
base_agent.py
Lines 1384-1388


            self.brain.mem.add(
                messages=messages,
                user_id=self.user_id,
                agent_id=self.agent_id,
                run_id=self.session_id,
base_agent.py
Lines 1403-1407
            response = self.brain.mem.get_all(
                user_id=self.user_id,
                agent_id=self.agent_id,
                run_id=self.session_id
            )

```
另外，user_id / agent_id / session_id 至少要有一个非空 才允许 add_memory，避免无标数据写进库。


2. 默认「不同 Agent 实例互相看不见」：靠随机 ID
每个 BaseAgent 默认会生成新的 agent_id 和 session_id（UUID），所以两个 Agent 若你不改配置，传给 mem0 的键不同，自然落在不同分区（在 mem0 的约定下即相互隔离）。

```python
base_agent.py
Lines 42-44


    user_id: Optional[str] = None
    agent_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    session_id: str = field(default_factory=lambda: str(uuid.uuid4()))


```

user_id 默认是 None，若你只依赖 agent_id/session_id，隔离仍主要靠这两项（以及 mem0 对 None 字段的处理方式——具体以 mem0 文档为准）。


3. 读路径上的「二次过滤」：metadata 可选
get_all_memories 在拿到 mem0 返回结果后，还可以按 metadata 字典再筛一层，这是 应用层细粒度隔离/分类，不是存储引擎本身。


4. 和长期记忆无关的「隔离」

History（短期上下文）：挂在各自 AgentState 上，不同 Agent 不同 state 对象 → 进程内天然分开。

AssistantAgent 的 conversation_history：enable_state_tracking 时也是该实例自己的列表。



一句话：minion 侧的「记忆隔离」= 每个 Agent 带一组 user_id/agent_id/session_id（默认 UUID 保证实例间不同） + 所有 mem0 调用都带上这组 ID，由 mem0 按维度划分数据；若要跨实例共享，就要故意设成相同的 ID，而不是框架再单独开一套隔离逻辑。