---
title: "Dair Ai Advanced Ai Agents"
date: 2025-07-16T22:32:59+08:00
# bookComments: false
# bookSearchExclude: false
---

# Advanced AI Agents --- Dair AI

## Building with Agents

Agent有基础和进阶两种，基础的agent可以看作是预先定义流程的workflow，基础和进阶agent的关键区别在于predefined和autonomous

* Basic agentic systems can include LLM workflows that involve LLMs and tools orchestrated with predefined control flows.（后续用workflow指代）

* More advanced agentic systems include agents made up of LLM/s that can autonomously perform more complex tasks through reasoning, reflection, and tool use.（后续用agent指代）
* workflow更具predictability and consistency，agent更具flexibility和model-driven decision-making，当构建LLM Application时，首先尝试最简单的解决方案，只在必要时增加复杂性，优先尝试workflow，当flexibility和model-driven decision-making更重要时再使用agent（simple solutions--->workflows--->agents）

## Augmented LLMs

Augmented LLMs是the foundational building block of agentic systems.

Augmented LLMs具有planning, tool use, memory（short-term memory和long-term memory ）能力

* short-term memory（working memory ）: in-context learning
* long-term memory: **external vector store** accessible to the agent at **query** time with fast retrieval

## workflow类型

* **Prompt chaining**：This workflow is ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. 

  ![img](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

* **Routing**： Routing works well for complex tasks where there are distinct categories that are better handled separately, and where classification can be handled accurately, either by an LLM or a more traditional classification model/algorithm.

  ![img](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

* **Parallelization**：以下场景比较适合并行化：

  * **Sectioning**: Breaking a task into independent subtasks run in parallel.
  * **Voting:** Running the same task multiple times to get diverse outputs.

![img](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

* **Orchestrator-workers**：This workflow is well-suited for complex tasks where you can’t predict the subtasks needed (in coding, for example, the number of files that need to be changed and the nature of the change in each file likely depend on the task). Whereas it’s topographically similar, the key difference from parallelization is its flexibility—subtasks aren't pre-defined, but determined by the orchestrator based on the specific input.

![img](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

* **Evaluator-optimizer**：This workflow is particularly effective when we have clear evaluation criteria, and when iterative refinement provides measurable value. The two signs of good fit are, first, that LLM responses can be demonstrably improved when a human articulates their feedback; and second, that the LLM can provide such feedback. This is analogous to the iterative writing process a human writer might go through when producing a polished document.

![img](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

## Agent类型

