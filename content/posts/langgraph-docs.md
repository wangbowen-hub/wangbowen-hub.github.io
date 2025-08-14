---
title: "Langgraph Docs"
date: 2025-07-30T21:43:23+08:00
# bookComments: false
# bookSearchExclude: false
---

# LangGraph Tourios

## Graph API

### Overview

LangGraph的图结构包含以下三部分：

* State: State是Graph的共享数据结构，包含schema和reducer两部分，分别用于定义State结构和更新State
  * schema有一下3中类型：
    * TypedDict 最广泛使用
    * dataclass 需要设置默认值的情况下使用
    * Pydantic的BaseModel 需要recursive data validation的情况下使用，性能可能不如TypedDict和dataclass
      * state_schema为BaseModel类型时只会验证输入，不会验证输出，且输出也不会是Pydantic实例
  * reducer默认为覆盖更新，可通过Annotated进行定义
* Nodes：Nodes是接受state，config和runtime参数的 Python 函数（同步或异步均可）
* Edges： Edges是Graph逻辑的流转路径，Edges类型有以下4种：
  * Normal Edges
  * Conditional Edges
  * Entry Point
  * Conditional Entry Point

### Send

map-reduce设计模式会用到

### Command

更新状态（update）加前往节点（goto）

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```

### Runtime Context

可以通过Runtime[ContextSchema]传递不属于state中的额外参数

通过config中的recursion_limit参数限制递归次数

```python
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

