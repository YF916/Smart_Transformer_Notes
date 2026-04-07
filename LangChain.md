### LangChain Agent Notes

#### 1. `create_agent()`
  * model
  * tools
  * state
  * context
  * middleware

#### 2. `state`（动态）

* 会随着流程改变
* 用来记录 agent 当前状态

```python
class AuthenticatedState(AgentState):
    authenticated: bool
```

#### 3. `context`（静态）

* 外部传入，不变
* 类似“环境信息”

```python
@dataclass
class EmailContext:
    email_address: str
    password: str
```

#### 4. Tool

### 工具分三类：

#### 1. 功能型工具

```python
@tool
def check_inbox()
def send_email()
```

👉 做具体事情（读邮件 / 发邮件）

---

#### 2. 控制型工具（⭐重点）

```python
def authenticate(...)
```

特点：

* 返回 `Command`（不是普通 string）
* 可以 **修改 state**

```python
return Command(
    update={
        "authenticated": True
    }
)
```

👉 本质：

> tool 不只是工具，还能控制 agent 状态机

---

## 4️⃣ ToolRuntime（隐藏重点）

```python
runtime.context.email_address
```

👉 用途：

* 在 tool 里访问 context / metadata
* 比直接传参数更安全

---

## 5️⃣ Middleware：核心控制层（最重要🔥）

---

### ① `wrap_model_call`（控制工具权限）

```python
@wrap_model_call
def dynamic_tool_call(...)
```

作用：
👉 **动态决定 LLM 能用哪些 tools**

```python
if authenticated:
    tools = [check_inbox, send_email]
else:
    tools = [authenticate]
```

👉 example：

| 状态  | 可用工具            |
| --- | --------------- |
| 未登录 | 只能 authenticate |
| 已登录 | 可以读邮件 / 发邮件     |

👉 本质：

> 做“权限控制”（像 backend auth layer）

---

### ② `dynamic_prompt`（动态系统提示）

```python
@dynamic_prompt
def dynamic_prompt_func(...)
```

作用：
👉 根据 state 改 system prompt

```python
if authenticated:
    return "You can check inbox"
else:
    return "You need to authenticate"
```

👉 本质：

> prompt 也是“状态驱动”的

---

### ③ HumanInTheLoop（人工拦截）

```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "send_email": True
    }
)
```

👉 example：

* send_email → ❗暂停，等人确认
* check_inbox → ✅直接执行

👉 本质：

> 给高风险操作加“审批流程”

---

## 6️⃣ Agent 行为流程（完整链路🔥）

### 🧩 Step-by-step

1. 用户输入
2. Agent 看 state（是否 authenticated）
3. Middleware：

   * 限制 tools
   * 修改 prompt
4. LLM 决定调用哪个 tool
5. Tool 执行：

   * 返回结果 / Command
6. 更新 state
7. 循环

---

### 🧠 Example Flow

#### 🟡 未登录

用户：

> check my inbox

👉 LLM：

* 发现不能用 `check_inbox`
* 只能用 `authenticate`

---

#### 🟢 登录成功

用户：

> email = [julie@example.com](mailto:julie@example.com), password = password123

👉 authenticate：

```python
authenticated = True
```

---

#### 🔵 登录后

用户：

> check inbox

👉 now allowed ✅

---

## 7️⃣ Command vs 普通 return（核心理解‼️）

| 类型             | 作用            |
| -------------- | ------------- |
| return string  | 给 LLM 看       |
| return Command | 改 state + 控流程 |

👉 example：

```python
return Command(update={"authenticated": True})
```

👉 本质：

> Command = “状态机操作”

---

## 8️⃣ 整体架构一句话总结

👉

> Agent = 状态机 + LLM 决策 + Middleware 控制 + Tools 执行

---

## 9️⃣ 你这段代码的核心思想（最重要总结）

👉

* 用 **state 管登录状态**
* 用 **middleware 控权限**
* 用 **tool 改 state**
* 用 **human-in-loop 控风险**

---

## 🔟 一句话理解这个 demo

👉

> 这是一个“带登录系统 + 权限控制 + 人工审批”的 LLM Agent

---

如果你接下来要学 **LangGraph**，这套其实就是：

👉 **LangChain（简单版） → LangGraph（可控流程版）**

要不要我帮你把这段代码“翻译成 LangGraph 思维”？会直接帮你理解下一步 👍

```
创建 agent
create_agent():
model
tools
state
context
middleware



我帮你整理成那种**小点 + example**的 notes👇（偏理解导向）

---

# 🧠 LangChain Agent + Middleware（认证流程）Notes

## 1️⃣ Agent 基本结构

* 用 `create_agent()` 创建 agent
* 核心组成：

  * **model**（gpt-5-nano）
  * **tools**
  * **state**
  * **context**
  * **middleware**

👉 本质：

> Agent = LLM + tools + 状态管理 + 控制逻辑

---

## 2️⃣ State vs Context（很关键‼️）

### ✅ `state`（动态）

* 会随着流程改变
* 用来记录 agent 当前状态

```python
class AuthenticatedState(AgentState):
    authenticated: bool
```

👉 example：

* 刚开始：`authenticated = False`
* 登录成功后：`authenticated = True`

---

### ✅ `context`（静态）

* 外部传入，不变
* 类似“环境信息”

```python
@dataclass
class EmailContext:
    email_address: str
    password: str
```

👉 example：

* 存真实账号密码（验证用）

---

## 3️⃣ Tool 设计逻辑

### 工具分三类：

#### 1. 功能型工具

```python
@tool
def check_inbox()
def send_email()
```

👉 做具体事情（读邮件 / 发邮件）

---

#### 2. 控制型工具（⭐重点）

```python
def authenticate(...)
```

特点：

* 返回 `Command`（不是普通 string）
* 可以 **修改 state**

```python
return Command(
    update={
        "authenticated": True
    }
)
```

👉 本质：

> tool 不只是工具，还能控制 agent 状态机

---

## 4️⃣ ToolRuntime（隐藏重点）

```python
runtime.context.email_address
```

👉 用途：

* 在 tool 里访问 context / metadata
* 比直接传参数更安全

---

## 5️⃣ Middleware：核心控制层（最重要🔥）

---

### ① `wrap_model_call`（控制工具权限）

```python
@wrap_model_call
def dynamic_tool_call(...)
```

作用：
👉 **动态决定 LLM 能用哪些 tools**

```python
if authenticated:
    tools = [check_inbox, send_email]
else:
    tools = [authenticate]
```

👉 example：

| 状态  | 可用工具            |
| --- | --------------- |
| 未登录 | 只能 authenticate |
| 已登录 | 可以读邮件 / 发邮件     |

👉 本质：

> 做“权限控制”（像 backend auth layer）

---

### ② `dynamic_prompt`（动态系统提示）

```python
@dynamic_prompt
def dynamic_prompt_func(...)
```

作用：
👉 根据 state 改 system prompt

```python
if authenticated:
    return "You can check inbox"
else:
    return "You need to authenticate"
```

👉 本质：

> prompt 也是“状态驱动”的

---

### ③ HumanInTheLoop（人工拦截）

```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "send_email": True
    }
)
```

👉 example：

* send_email → ❗暂停，等人确认
* check_inbox → ✅直接执行

👉 本质：

> 给高风险操作加“审批流程”

---

## 6️⃣ Agent 行为流程（完整链路🔥）

### 🧩 Step-by-step

1. 用户输入
2. Agent 看 state（是否 authenticated）
3. Middleware：

   * 限制 tools
   * 修改 prompt
4. LLM 决定调用哪个 tool
5. Tool 执行：

   * 返回结果 / Command
6. 更新 state
7. 循环

---

### 🧠 Example Flow

#### 🟡 未登录

用户：

> check my inbox

👉 LLM：

* 发现不能用 `check_inbox`
* 只能用 `authenticate`

---

#### 🟢 登录成功

用户：

> email = [julie@example.com](mailto:julie@example.com), password = password123

👉 authenticate：

```python
authenticated = True
```

---

#### 🔵 登录后

用户：

> check inbox

👉 now allowed ✅

---

## 7️⃣ Command vs 普通 return（核心理解‼️）

| 类型             | 作用            |
| -------------- | ------------- |
| return string  | 给 LLM 看       |
| return Command | 改 state + 控流程 |

👉 example：

```python
return Command(update={"authenticated": True})
```

👉 本质：

> Command = “状态机操作”

---

## 8️⃣ 整体架构一句话总结

👉

> Agent = 状态机 + LLM 决策 + Middleware 控制 + Tools 执行

---

## 9️⃣ 你这段代码的核心思想（最重要总结）

👉

* 用 **state 管登录状态**
* 用 **middleware 控权限**
* 用 **tool 改 state**
* 用 **human-in-loop 控风险**

---

## 🔟 一句话理解这个 demo

👉

> 这是一个“带登录系统 + 权限控制 + 人工审批”的 LLM Agent

---

如果你接下来要学 **LangGraph**，这套其实就是：

👉 **LangChain（简单版） → LangGraph（可控流程版）**

要不要我帮你把这段代码“翻译成 LangGraph 思维”？会直接帮你理解下一步 👍

```
