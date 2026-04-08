### LangChain Agent Notes

#### 1. `create_agent()`

  * model
  * tools
  * state
  * context
  * middleware
  * system_prompt
  * response_format
  * checkpointer=InMemorySaver()

#### 2. MCP

```python
client = MultiServerMCPClient(
    {...}
)
tools = await client.get_tools()
```

#### 3. `state`（动态）

```python
class AuthenticatedState(AgentState):
    authenticated: bool
```
* 会变

#### 4. `context`（静态）

```python
@dataclass
class EmailContext:
    email_address: str
    password: str
```
* 不变

#### 5. Tool （执行具体任务）

```python
tavily_client = TavilyClient()

@tool
def web_search(query: str) -> Dict[str, Any]:
    """Search the web for information"""
    return tavily_client.search(query)
```

```python
@tool
def check_inbox()
```

```python
def authenticate(..., runtime: ToolRuntime)
    ...
    return Command(
        update={
            "authenticated": True
        }
    )
```
* 返回 `Command` 可以修改 state
* 可读 runtime.context, runtime.tool_call_id

#### 6. Middleware （控制行为策略）

##### (1) `SummarizationMiddleware`

```python
SummarizationMiddleware(
    model="...",
    trigger=("tokens", 100),
    keep=("messages", 1)
    )
```

##### (2) Trim/delete messages

```python
@before_agent
def trim_messages(state: AgentState, runtime: Runtime):
    """Remove all the tool messages from the state"""
```

##### (3) `wrap_model_call`

```python
@wrap_model_call
def dynamic_tool_call(request: ModelRequest, 
handler: Callable[[ModelRequest], ModelResponse]) -> ModelResponse:
    if ...:
        tools = [...]
    else:
        tools = [...]
    request = request.override(tools=tools)
    return handler(request) 
```
* 决定 tools 的使用

##### (4) `dynamic_prompt`

```python
@dynamic_prompt
def dynamic_prompt_func(request: ModelRequest)
    if ...:
        return "..."
    else:
        return "..."
```
* 修改 system prompt

##### (5) HumanInTheLoop

```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "send_email": True
    }
)
```
```python
agent.invoke(
    Command( 
        resume={"decisions": [{"type": "approve"}]} # reject/edit
    ), 
    config=config
)
```

#### ModelRequest vs. ToolRuntime

* ModelRequest（出现在 middleware）：读 state，读 context，修改模型配置（model / tools / temperature）
* ToolRuntime（出现在 tool）：使用 Command 更新 state，读 context

