# LangGraph Notes
```text
State: Data
Node: Functions
Edges: Control Flow
Checkpointing/Memory
Human in the loop
```

### State

```python
class State(TypedDict):
    nlist: Annotated[List[str], operator.add]
```

* **多个 node 同时更新 state 时，由 reducer 决定如何合并**

### Conditional Edge
#### 方式一：`add_conditional_edges` （节点外部）

```python
builder.add_conditional_edges("a", conditional_edge)
```

```python
def conditional_edge(state: State) -> Literal["b", "c", END]:
    ...
```

##### 执行流程

```text
node a 执行
   ↓
返回 state
   ↓
调用 conditional_edge(state)
   ↓
决定 next node
```

#### 方式二：`Command`（节点内部）

```python
def node_a(state: State) -> Command[Literal["b", "c", END]]:
    return Command(
        update=State(...),
        goto=[next_node]
    )
```

##### 执行流程

```text
node a 执行
   ↓
返回 Command:
   - update state
   - goto next node
   ↓
直接跳转
```

### Memory
```python
from langgraph.checkpoint.memory import InMemorySaver

memory = InMemorySaver()
graph = builder.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "1"}}
result = graph.invoke(input_state, config)
```
* state 会累加
* 每个 thread_id 是独立 memory
* invoke 是在已有 state 上继续执行

### Interrupt
#### Setup

1. 在 node 中使用 interrupt()
```python
from langgraph.types import interrupt

def node_a(state):
    if unexpected_input:
        return interrupt("Invalid input, please confirm")
```
2. 必须有 checkpointer
```python
graph = builder.compile(checkpointer=memory)
```
* interrupt 期间：state = 被保存（checkpoint）
* resume 时：恢复之前的 state → 接着执行

#### Execution
```
node_a → interrupt()
        ↓
graph 停止
        ↓
admin 需要给 decision
        ↓
load state → 继续 graph
```

### Email Agent Workflow Notes
* **State** 是核心，node 读/写 state，不传参，而是用 state 传递信息
* 3个流程控制方式：
    * Edge: builder.add_edge("a", "b")
    * Command: return Command(goto="xxx")
    * interrupt: interrupt(...)
* 工作流结构上融入了 RAG
    * R（Retrieve）
      ```python
      def search_documentation(state: EmailAgentState) -> EmailAgentState:
      """Search knowledge base for relevant information"""
      ```
    * A（Augment）
      ```python
      def write_response(state: EmailAgentState) -> Command[Literal["human_review", "send_reply"]]:
      "Generate response using context and route based on quality"""
      ...
      context_sections = []
      context_sections.append(f"Relevant documentation:\n{formatted_docs}")
      ...
      ```
    * G（Generate）
      ```python
      response = llm.invoke(draft_prompt)
      ```
