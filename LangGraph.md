State: Data
Node: functions
edges: Control Flow
Checkpointing/Memory
Human in the loop

```python
class State(TypedDict):
    nlist: Annotated[List[str], operator.add]
```
LangGraph 设计：
LangGraph 期望 node 是“无副作用、声明式”
reducer 才是合并逻辑的统一入口，是“线程安全合并机制”
LangGraph 内部会：
收集所有 node 输出
再统一用 reducer 合并
