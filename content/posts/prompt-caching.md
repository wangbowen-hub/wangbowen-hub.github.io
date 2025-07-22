---
title: "Prompt Caching"
date: 2025-07-21T18:39:38+08:00
# bookComments: false
# bookSearchExclude: false
---

# Prompt Caching

## Overview

### Prompt Caching是什么？

Prompt caching（提示缓存）是一种针对大型语言模型 API 的性能优化技术，它会把常见的提示前缀及其已计算好的自注意力 KV 状态缓存在服务器端，后续只要前缀命中就能直接复用这些状态、跳过重复前向计算，从而显著降低延迟、计算量与 token 成本。

Prompt caching有两个作用：降低延迟和降低成本。



