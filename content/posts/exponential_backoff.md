---
title: "Exponential Backoff"
date: 2025-08-07T18:22:19+08:00
# bookComments: false
# bookSearchExclude: false
---

# 指数退避算法

指数退避算法适用于重试场景，避免大量客户端请求失败后立即重试，造成服务崩溃

对于第n次重试（n从1开始），需等待时间：

$delay(n) = base*(factor)^{n-1}$

- **base**：起始等待（秒）；常用 0.5~2
- **factor**：指数因子；常用 2

在实际生产环境中，需要：

* 设置最大重试次数/最大等待时间，避免无限增大等待
* 添加抖动，避免大量客户端在同一时间发出重试请求
