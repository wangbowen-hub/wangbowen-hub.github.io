---
title: "Python Logging"
date: 2025-07-22T15:01:16+08:00
# bookComments: false
# bookSearchExclude: false
---

# Python日志

## python日志处理异常原则

### 1.只在“处理异常的地方”日志记录一次

* 不要层层重复 log 同一个异常。

* 谁决定吞掉/转换/降级异常，谁负责记录；继续往上抛就别记录日志。
* logger和print输出e和str(e)的结果相同，e底层还是会调用str(e)

```python
❌ 错误示范：每层都 log
# dao.py
def get_user(uid):
    try:
        return db.query("SELECT * FROM users WHERE id=%s", uid)
    except Exception:
        logger.exception("DB error")   # 第一次记录
        raise

# service.py
def load_user_profile(uid):
    try:
        user = get_user(uid)
        return enrich(user)
    except Exception:
        logger.exception("Load user profile failed")  # 第二次记录
        raise

# controller.py
def user_profile_handler(request):
    try:
        uid = request.args["uid"]
        profile = load_user_profile(uid)
        return JSONResponse(profile)
    except Exception:
        logger.exception("Request failed")  # 第三次记录
        return JSONResponse({"error": "internal error"}, status=500)

```

```python
✅ 正确示范：只在“真正处理异常”的地方记录
# dao.py
def get_user(uid):
    # 不处理，抛给上层
    return db.query("SELECT * FROM users WHERE id=%s", uid)

# service.py
def load_user_profile(uid):
    try:
        user = get_user(uid)
    except DatabaseError as e:
        # 这里决定转换为业务异常并继续抛出 —— 不 log
        raise UserNotFoundError(f"user {uid} not found") from e
    return enrich(user)

# controller.py
def user_profile_handler(request):
    try:
        uid = request.args["uid"]
        profile = load_user_profile(uid)
        return JSONResponse(profile)
    except UserNotFoundError as e:
        # 这里我们选择向用户返回 404，并结束异常传播 —— 这里 log 一次
        logger.warning("User not found: uid=%s, msg=%s", uid, e, exc_info=True)
        return JSONResponse({"error": "user not found"}, status=404)
    except Exception as e:
        # 兜底处理：返回 500 —— 这里 log 一次
        logger.exception("Unexpected error when handling user_profile")
        return JSONResponse({"error": "internal error"}, status=500)

# DAO 层不处理、不记录：它不了解业务语义。

# Service 层把底层异常转换成业务异常（raise ... from e 保留链路），但不记录，因为它没“终结”异常。

# Controller 层决定了响应策略（404 或 500），它才是“处理者”，所以在这里记录。

```

**转换**：将异常转换后再抛出，语义更清晰，使用 `from` 保留原始堆栈。

**降级**：捕获后采用备选/默认方案继续执行，同时记录日志。

```python
# 转换
class UserNotFoundError(RuntimeError):
    pass

def get_user(uid):
    try:
        row = db.query("SELECT ...")
    except DatabaseError as e:
        # 转换为更有业务语义的异常，再抛出（保留原始异常链）
        raise UserNotFoundError(f"user {uid} not found") from e

```

```python
# 降级
def parse_config(path):
    try:
        with open(path) as f:
            return yaml.safe_load(f)
    except Exception as e:
        logger.warning("Failed to load config %s, use defaults. err=%s", path, e)
        return DEFAULT_CONFIG

```

**日志是否处理的判断标准：**

1. 我是否还要把异常继续抛给上层？

   - 是 → 一般不 log（除非仅 DEBUG 级别用于内部开发调试）。

   - 否 → 我结束传播 → 这里 log。

2. 我有没有改变异常的语义？

   - 如果只是 `raise` 原样抛 → 不 log。

   - 如果 `raise NewError(...) from e` 并让上层继续处理 → 仍不要 log（因为还没“终结”）。

### 2. 始终记录完整堆栈

* 可使用`logging.exception(...)` 或 `logger.error("msg", exc_info=True)`。

```python
try:
    1 / 0
except ZeroDivisionError as e:
    logger.error("calc failed: %s", e)   # 只有 'division by zero'，无堆栈
```

```python
try:
    1 / 0
except ZeroDivisionError:
    logger.exception("calc failed")
    # 或logger.error("calc failed: %s", e, exc_info=True)，日志会记录堆栈
    
##################
# 记录了堆栈的日志
ERROR calc failed
Traceback (most recent call last):
  File "/app/foo.py", line 3, in <module>
    1 / 0
ZeroDivisionError: division by zero

```



### 3. 日志需要选择合适的级别

| 情况                                    | 级别                      |
| --------------------------------------- | ------------------------- |
| 异常导致当前请求/任务失败               | `ERROR`                   |
| 异常可自动重试/有降级策略且业务未受影响 | `WARNING`                 |
| 调试信息（开发调试环境）                | `DEBUG`                   |
| 安全/合规事件（如认证失败）             | `INFO` 或专用审计日志通道 |

### 4. 日志要有上下文

包含关键参数，如用户/请求 ID、资源名、重试次数……（便于排查）。

### 5. 日志保存应该是结构化的

日志保存建议是JSON或其他key-value格式



