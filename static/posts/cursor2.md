# Review_handler 是否支持多并发
_Exported on 2025/8/25 at GMT+8 18:11:17 from Cursor (1.4.5)_

---

**User**

review_handler支持多并发吗

---

**Cursor**

我先查看 `handler.py` 中的 `review_handler` 实现，以及在 `app.py` 中是如何被调用的，同时快速扫一眼 `graph/graph.py` 看看是否存在异步或并发控制。

我先快速看了 `handler.py`、`app.py` 和 `graph/graph.py`，确认回调与路由的并发模型。

- 结论：当前的 `review_handler` 是进程内的全局单例，包含可变共享状态（`records` 列表和 `_start_times` 字典），并且在每个请求开头会执行 `review_handler.records.clear()`。在同一 worker 内存在异步并发时（`uvicorn` worker 支持并发处理请求，限流只限制速率不保证串行），这会产生竞争条件：一个请求可能清空另一个请求正在收集的统计数据。因此现在的实现并不支持安全的多并发。

- 修复建议（最简单稳妥）：为每个请求创建一个新的 `UsageAndLatencyHandler` 实例，避免共享可变状态；不要使用全局的 `review_handler`。下面是修改要点。

### 关键 edits

1) 修改 `app.py` 导入
```12:18:app.py
from fastapi import FastAPI, Response
from handler import UsageAndLatencyHandler
import time
from aiolimiter import AsyncLimiter
```

2) `POST /review` 每次请求内创建独立 handler，并用它收集用量
```59:115:app.py
@app.post("/review", response_model=ReviewResponse)
async def review_images(request: ReviewRequest, response: Response) -> ReviewResponse:
    start_time = time.perf_counter()
    handler = UsageAndLatencyHandler()
    try:
        async with review_limiter:
            pass
    except:
        ai_reviewer_logger.warning(
            "请求被限流", request_id=request.request_id, endpoint="/review"
        )
        response.status_code = 429
        return ReviewResponse(errors={REQUEST_ERROR: TOO_MANY_REQUESTS_ERROR})
    base64_images = await async_compress_and_encode_images_from_urls(
        request_id=request.request_id, image_urls=request.img_urls, quality=100
    )
    base64_images = [img for img in base64_images if img is not None]
    if len(base64_images) == 0:
        ai_reviewer_logger.error(
            "所有图像下载/解压出错",
            request_id=request.request_id,
            img_urls=request.img_urls,
        )
        response.status_code = 400
        return ReviewResponse(errors={IMAGE_ERROR: IMAGE_DOWNLOAD_OR_COMPRESS_ERROR})
    reference_information = json.dumps(
        request.reference_information, indent=2, ensure_ascii=False
    )
    review_criteria = f"- 类别必须为{request.category}\n" + request.review_criteria
    result = await review_fanout_app.ainvoke(
        {
            "reference_information": reference_information,
            "review_criteria": review_criteria,
        },
        context={"base64_images": base64_images},
        config={"callbacks": [handler]},
    )

    total_processing_time = time.perf_counter() - start_time
    usage_stats = None
    if handler.records:
        usage_stats = {
            "total_prompt_tokens": sum(
                r.get("prompt_tokens", 0)
                for r in handler.records
                if r.get("prompt_tokens")
            ),
            "total_completion_tokens": sum(
                r.get("completion_tokens", 0)
                for r in handler.records
                if r.get("completion_tokens")
            ),
            "total_tokens": 0,
            "total_latency_s": round(total_processing_time, 3),
            "node_details": handler.records,
        }
        usage_stats["total_tokens"] = (
            usage_stats["total_prompt_tokens"] + usage_stats["total_completion_tokens"]
        )
```

3) `POST /review-fast` 同样改为请求内实例化
```136:174:app.py
@app.post("/review-fast", response_model=ReviewResponse)
async def review_images_fast(
    request: ReviewRequest, response: Response
) -> ReviewResponse:
    start_time = time.perf_counter()
    handler = UsageAndLatencyHandler()
    try:
        async with review_limiter:
            pass
    except:
        ai_reviewer_logger.warning(
            "请求被限流", request_id=request.request_id, endpoint="/review"
        )
        response.status_code = 429
        return ReviewResponse(errors={REQUEST_ERROR: TOO_MANY_REQUESTS_ERROR})
    base64_images = await async_compress_and_encode_images_from_urls(
        request_id=request.request_id, image_urls=request.img_urls, quality=100
    )
    base64_images = [img for img in base64_images if img is not None]
    if len(base64_images) == 0:
        ai_reviewer_logger.error(
            "所有图像下载/解压出错",
            request_id=request.request_id,
            img_urls=request.img_urls,
        )
        response.status_code = 400
        return ReviewResponse(errors={IMAGE_ERROR: IMAGE_DOWNLOAD_OR_COMPRESS_ERROR})
    reference_information = json.dumps(
        request.reference_information, indent=2, ensure_ascii=False
    )
    review_criteria = f"- 类别必须为{request.category}\n" + request.review_criteria
    result = await review_fanout_fast_app.ainvoke(
        {
            "reference_information": reference_information,
            "review_criteria": review_criteria,
        },
        context={"base64_images": base64_images},
        config={"callbacks": [handler]},
    )

    total_processing_time = time.perf_counter() - start_time
    usage_stats = None
    if handler.records:
        usage_stats = {
            "total_prompt_tokens": sum(
                r.get("prompt_tokens", 0)
                for r in handler.records
                if r.get("prompt_tokens")
            ),
            "total_completion_tokens": sum(
                r.get("completion_tokens", 0)
                for r in handler.records
                if r.get("completion_tokens")
            ),
            "total_tokens": 0,
            "total_latency_s": round(total_processing_time, 3),
            "node_details": handler.records,
        }
        usage_stats["total_tokens"] = (
            usage_stats["total_prompt_tokens"] + usage_stats["total_completion_tokens"]
        )
```

4) `POST /extract-criteria` 同样改为请求内实例化
```215:237:app.py
@app.post("/extract-criteria", response_model=CriteriaExtractResponse)
async def extract_criteria(
    request: CriteriaExtractRequest, response: Response
) -> CriteriaExtractResponse:
    start_time = time.perf_counter()
    handler = UsageAndLatencyHandler()
    try:
        async with criteria_extract_limiter:
            pass
    except:
        ai_reviewer_logger.warning(
            "请求被限流", request_id=request.request_id, endpoint="/review"
        )
        response.status_code = 429
        return CriteriaExtractResponse(errors={REQUEST_ERROR: TOO_MANY_REQUESTS_ERROR})

    result = await criteria_extract_app.ainvoke(
        {
            "category": request.category,
            "user_input": request.user_input,
        },
        config={"callbacks": [handler]},
    )

    total_processing_time = time.perf_counter() - start_time
    usage_stats = None
    if handler.records:
        usage_stats = {
            "total_prompt_tokens": sum(
                r.get("prompt_tokens", 0)
                for r in handler.records
                if r.get("prompt_tokens")
            ),
            "total_completion_tokens": sum(
                r.get("completion_tokens", 0)
                for r in handler.records
                if r.get("completion_tokens")
            ),
            "total_tokens": 0,
            "total_latency_s": round(total_processing_time, 3),
            "node_details": handler.records,
        }
        usage_stats["total_tokens"] = (
            usage_stats["total_prompt_tokens"] + usage_stats["total_completion_tokens"]
        )
```

- 额外说明：`handler.py` 的 `on_llm_start/on_llm_end` 使用 `run_id` 作为键，本身在单请求内是安全的；一旦改为“每请求实例化”，就不会被其他请求清空或篡改，异步并发也安全。

如果你希望继续用全局实例，也可以用 `contextvars.ContextVar` 或按 `request_id` 将 `records` 拆分为 per-request 容器，但成本更高；以上“每请求一个实例”是最直接可靠的方案。

- 我将通过以上 edits 彻底消除并发竞态；需要我帮你直接改好 `app.py` 吗？

- 变更影响
  - 移除对全局 `review_handler` 的依赖，避免并发清空数据。
  - 每个请求独立收集统计，支持同一 worker 内的多请求并发。

```python
from fastapi import FastAPI, Response
from handler import UsageAndLatencyHandler
import time
from aiolimiter import AsyncLimiter
```

```python
@app.post("/review", response_model=ReviewResponse)
async def review_images(request: ReviewRequest, response: Response) -> ReviewResponse:
    start_time = time.perf_counter()
    handler = UsageAndLatencyHandler()
    try:
        async with review_limiter:
            pass
    except:
        ai_reviewer_logger.warning(
            "请求被限流", request_id=request.request_id, endpoint="/review"
        )
        response.status_code = 429
        return ReviewResponse(errors={REQUEST_ERROR: TOO_MANY_REQUESTS_ERROR})
    base64_images = await async_compress_and_encode_images_from_urls(
        request_id=request.request_id, image_urls=request.img_urls, quality=100
    )
    base64_images = [img for img in base64_images if img is not None]
    if len(base64_images) == 0:
        ai_reviewer_logger.error(
            "所有图像下载/解压出错",
            request_id=request.request_id,
            img_urls=request.img_urls,
        )
        response.status_code = 400
        return ReviewResponse(errors={IMAGE_ERROR: IMAGE_DOWNLOAD_OR_COMPRESS_ERROR})
    reference_information = json.dumps(
        request.reference_information, indent=2, ensure_ascii=False
    )
    review_criteria = f"- 类别必须为{request.category}\n" + request.review_criteria
    result = await review_fanout_app.ainvoke(
        {
            "reference_information": reference_information,
            "review_criteria": review_criteria,
        },
        context={"base64_images": base64_images},
        config={"callbacks": [handler]},
    )

    total_processing_time = time.perf_counter() - start_time
    usage_stats = None
    if handler.records:
        usage_stats = {
            "total_prompt_tokens": sum(
                r.get("prompt_tokens", 0)
                for r in handler.records
                if r.get("prompt_tokens")
            ),
            "total_completion_tokens": sum(
                r.get("completion_tokens", 0)
                for r in handler.records
                if r.get("completion_tokens")
            ),
            "total_tokens": 0,
            "total_latency_s": round(total_processing_time, 3),
            "node_details": handler.records,
        }
        usage_stats["total_tokens"] = (
            usage_stats["total_prompt_tokens"] + usage_stats["total_completion_tokens"]
        )
```

```python
@app.post("/review-fast", response_model=ReviewResponse)
async def review_images_fast(
    request: ReviewRequest, response: Response
) -> ReviewResponse:
    start_time = time.perf_counter()
    handler = UsageAndLatencyHandler()
    try:
        async with review_limiter:
            pass
    except:
        ai_reviewer_logger.warning(
            "请求被限流", request_id=request.request_id, endpoint="/review"
        )
        response.status_code = 429
        return ReviewResponse(errors={REQUEST_ERROR: TOO_MANY_REQUESTS_ERROR})
    base64_images = await async_compress_and_encode_images_from_urls(
        request_id=request.request_id, image_urls=request.img_urls, quality=100
    )
    base64_images = [img for img in base64_images if img is not None]
    if len(base64_images) == 0:
        ai_reviewer_logger.error(
            "所有图像下载/解压出错",
            request_id=request.request_id,
            img_urls=request.img_urls,
        )
        response.status_code = 400
        return ReviewResponse(errors={IMAGE_ERROR: IMAGE_DOWNLOAD_OR_COMPRESS_ERROR})
    reference_information = json.dumps(
        request.reference_information, indent=2, ensure_ascii=False
    )
    review_criteria = f"- 类别必须为{request.category}\n" + request.review_criteria
    result = await review_fanout_fast_app.ainvoke(
        {
            "reference_information": reference_information,
            "review_criteria": review_criteria,
        },
        context={"base64_images": base64_images},
        config={"callbacks": [handler]},
    )

    total_processing_time = time.perf_counter() - start_time
    usage_stats = None
    if handler.records:
        usage_stats = {
            "total_prompt_tokens": sum(
                r.get("prompt_tokens", 0)
                for r in handler.records
                if r.get("prompt_tokens")
            ),
            "total_completion_tokens": sum(
                r.get("completion_tokens", 0)
                for r in handler.records
                if r.get("completion_tokens")
            ),
            "total_tokens": 0,
            "total_latency_s": round(total_processing_time, 3),
            "node_details": handler.records,
        }
        usage_stats["total_tokens"] = (
            usage_stats["total_prompt_tokens"] + usage_stats["total_completion_tokens"]
        )
```

```python
@app.post("/extract-criteria", response_model=CriteriaExtractResponse)
async def extract_criteria(
    request: CriteriaExtractRequest, response: Response
) -> CriteriaExtractResponse:
    start_time = time.perf_counter()
    handler = UsageAndLatencyHandler()
    try:
        async with criteria_extract_limiter:
            pass
    except:
        ai_reviewer_logger.warning(
            "请求被限流", request_id=request.request_id, endpoint="/review"
        )
        response.status_code = 429
        return CriteriaExtractResponse(errors={REQUEST_ERROR: TOO_MANY_REQUESTS_ERROR})

    result = await criteria_extract_app.ainvoke(
        {
            "category": request.category,
            "user_input": request.user_input,
        },
        config={"callbacks": [handler]},
    )

    total_processing_time = time.perf_counter() - start_time
    usage_stats = None
    if handler.records:
        usage_stats = {
            "total_prompt_tokens": sum(
                r.get("prompt_tokens", 0)
                for r in handler.records
                if r.get("prompt_tokens")
            ),
            "total_completion_tokens": sum(
                r.get("completion_tokens", 0)
                for r in handler.records
                if r.get("completion_tokens")
            ),
            "total_tokens": 0,
            "total_latency_s": round(total_processing_time, 3),
            "node_details": handler.records,
        }
        usage_stats["total_tokens"] = (
            usage_stats["total_prompt_tokens"] + usage_stats["total_completion_tokens"]
        )
```

---

**User**

那这样的话run_id也没有什么作用了啊，因为每次invoke都会实例化新的handler，也就是说一个handler中不会有多个run_id

---

**Cursor**

不完全对。即使“每请求一个 handler”，单次 `ainvoke` 内也可能发生多次、甚至并发的 LLM 调用（例如 `fanout` 会并行跑多条分支、重试策略也会触发多次调用）。这些调用各自都有不同的 `run_id`，用 `run_id -> start_time` 的映射才能正确计算每次调用的延迟，避免相互覆盖。因此，`run_id` 在单个 handler 内依然有用。

- 如果某条路径恰好只触发一次 LLM 调用，`run_id` 只是“冗余但安全”；一旦出现并发/重试，它就变成“必要”。
- 若改成只存单一开始时间，遇到 `fanout` 并发就会算错延迟。建议保留当前按 `run_id` 记录的实现。

---

**User**

在同一 worker 内存在异步并发时（uvicorn worker 支持并发处理请求，限流只限制速率不保证串行），这会产生竞争条件：一个请求可能清空另一个请求正在收集的统计数据。因此现在的实现并不支持安全的多并发。
这个举例讲解一下，我是并发这方面的初学者

---

**Cursor**

好的，用一个时间线的小例子说明为什么全局单例的 `review_handler` 会在并发下出问题。

前提
- 现在是“全局一个 `review_handler`”，且每次请求开始都会执行 `review_handler.records.clear()`。
- 同一个 uvicorn worker 会并发处理多个请求；你的 `AsyncLimiter` 用法只是“放行速率”，不会把整个请求串行化（你写的是 `async with review_limiter: pass`，获取后立刻释放，后续请求体和 LLM 调用是并发进行的）。

并发交错示例
- T0：请求A进入，执行 clear → 全局 `records` 变为空。
- T1：请求A 触发 LLM 调用A1，`on_llm_start` 记录开始时间，稍后 `on_llm_end` 会向全局 `records` 追加一条记录。
- T2：请求B进入（与A并发），执行 clear → 把全局 `records` 再次清空（把A在 T1/T2 期间可能已经写入的记录清掉了）。
- T3：请求B 触发 LLM 调用B1，结束后把一条记录 append 到全局 `records`。
- T4：请求A先结束，A 从全局 `records` 汇总 usage。此时 `records` 里可能只有 B1（因为 T2 时被清空过，A 的早期记录丢了，且后来 B1 又写进来了），A 的 usage 被“掺”了 B 的数据，或丢了自己的数据。
- T5：请求B结束，B 再次从同一个全局 `records` 汇总 usage，此时里面可能混有 A 在 T4 之后追加的记录，B 的 usage 又被“掺”了 A 的数据。

会出现的两类问题
- 数据丢失：后到的请求执行 `clear()` 清掉先到请求已写入的记录。
- 数据串味：两个请求都向同一个 `records` append，结束时各自计算 usage，会把对方的记录也算进去。

为什么 `run_id` 解决不了这个问题
- `run_id` 只是在同一个 handler 实例内部区分“同一请求里的多次/并发 LLM 调用”。但你现在是“多个请求共享同一个 handler 实例”，列表本身是共享的，清空和汇总都是跨请求发生的，`run_id` 无法阻止清空或跨请求混算。

最简单的规避方式
- 每个请求内 new 一个 `UsageAndLatencyHandler()`，把它放进 `config={"callbacks": [handler]}`。这样不同请求的 `records` 和 `_start_times` 互不共享，自然就没有上述清空/串味的问题了。

---

**User**

那ai_reviewer_logger会导致竞态吗

---

**Cursor**

我先打开 `logger.py` 和 `woody_logger.py` 看看 `ai_reviewer_logger` 的实现，确认是否有共享可变状态或不安全的处理器。

我看了 `logger.py`/`woody_logger.py` 的实现，确认一下并发特性。

- 结论（同一 worker 内的异步并发）: 不会产生竞态。Python 标准 `logging` 在 handler 级别是加锁的；你的日志调用是同步的，单线程事件循环里不会写坏数据，最多只是多请求的日志“交错”输出，这是预期行为，不会相互污染。

- 需要注意的风险（多进程场景）:
  - 你在 `__main__` 下用 Gunicorn 多 worker（多进程）。标准的 `TimedRotatingFileHandler` 对“多进程同时写同一个文件”并不严格安全，可能出现轮转冲突、重复或丢行。
  - 规避方案（二选一或组合）:
    - 使用中心化日志（你已启用 Seq），可考虑关闭文件日志，仅保留 Seq + 控制台。
    - 给每个进程单独文件名（包含 PID），例如 `ai-reviewer-{pid}.log`。
    - 使用 `concurrent-log-handler` 这类支持多进程文件锁的 handler。
    - 用 `QueueHandler/QueueListener` 将各进程日志汇聚到单独的写入进程。

- 其他:
  - 你已经在日志里带了 `request_id` 等结构化字段（通过 kwargs），这能很好地区分并发请求的日志上下文。
  - 避免在高并发时动态调用 `log_to_console`/`log_to_file` 这类“临时增删 handler”的方法；当前代码只用 `info/warning/error`，是安全的。

- 小结
  - 同一 worker 的异步并发：安全。
  - 多 worker 多进程：文件轮转存在潜在竞态，建议采用上面的多进程安全方案或仅用 Seq。

---

**User**

用一句话总结结论及其原因

---

**Cursor**

同一 worker 内 ai_reviewer_logger 不会导致竞态，因为 Python logging 有加锁且调用同步；但多进程写同一日志文件可能冲突，因为 TimedRotatingFileHandler 非多进程安全。

---

**User**

那怎么修改日志避免写日志文件竞态

---

**Cursor**

我将把文件日志名改为包含进程 PID，确保每个 Gunicorn worker 写入各自文件，避免多进程写同一文件造成竞态。接着我会检查该文件的lint是否正常。

```python
import logging
import os
from logging.handlers import TimedRotatingFileHandler
from typing import Optional
import json

try:
    import seqlog

    HAS_SEQLOG = True
except ImportError:
    HAS_SEQLOG = False
    print("警告: seqlog 未安装，远程日志功能将不可用。请运行: pip install seqlog")


def _make_json_safe(obj):
    """
    将对象转换为可被 json.dumps 处理的安全结构。

    - 支持内置类型、dict、list/tuple/set
    - 对 Pydantic v2 使用 model_dump，v1 使用 dict
    - 回退到 __dict__，最后回退到 str
    """
    if obj is None or isinstance(obj, (str, int, float, bool)):
        return obj
    if isinstance(obj, dict):
        return {str(k): _make_json_safe(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple, set)):
        return [_make_json_safe(v) for v in obj]
    # Pydantic v2
    if hasattr(obj, "model_dump") and callable(getattr(obj, "model_dump")):
        try:
            return _make_json_safe(obj.model_dump())
        except Exception:
            pass
    # Pydantic v1
    if hasattr(obj, "dict") and callable(getattr(obj, "dict")):
        try:
            return _make_json_safe(obj.dict())
        except Exception:
            pass
    # 其它对象：尝试使用 __dict__
    if hasattr(obj, "__dict__"):
        try:
            return _make_json_safe(vars(obj))
        except Exception:
            pass
    # 最后兜底：如果仍不可序列化则转字符串
    try:
        json.dumps(obj)
        return obj
    except TypeError:
        return str(obj)


class WoodyLogger:
    """
    多功能日志记录器，支持控制台、远程 Seq 服务器和本地文件日志。

    特性：
    - 支持多种输出目标（控制台、文件、Seq服务器）
    - 支持按天轮转的日志文件
    - 支持结构化日志和额外上下文数据
    - 支持 exc_info、stack_info 等标准 logging 参数

    额外数据处理：
    - 可以直接通过 kwargs 传递额外数据：logger.info("消息", user_id=123)
    - kwargs 数据会以 JSON 格式附加到消息末尾

    Args:
        name (str): 日志记录器名称
        log_dir (str): 日志文件保存目录，默认为 'logs'
        seq_server_url (str, optional): Seq 服务器 URL
        seq_api_key (str, optional): Seq API 密钥
        default_level (str): 默认日志级别，默认为 'INFO'
    """

    def __init__(
        self,
        name: str,
        log_dir: str = "logs",
        seq_server_url: Optional[str] = None,
        seq_api_key: Optional[str] = None,
        default_level: str = "INFO",
    ):
        """
        初始化 WoodyLogger。

        Args:
            name (str): 日志记录器名称
            log_dir (str): 日志文件保存目录
            seq_server_url (str, optional): Seq 服务器 URL
            seq_api_key (str, optional): Seq API 密钥
            default_level (str): 默认日志级别
        """
        self.name = name
        self.log_dir = log_dir
        self.seq_server_url = seq_server_url
        self.seq_api_key = seq_api_key
        self.default_level = getattr(logging, default_level.upper())

        # 创建日志目录
        os.makedirs(log_dir, exist_ok=True)

        # 创建主日志记录器
        self.logger = logging.getLogger(name)
        self.logger.setLevel(logging.DEBUG)  # 设置最低级别，具体由处理器控制

        # 避免重复添加处理器
        self.logger.handlers = []

        # 设置日志格式
        self.console_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        )
        self.file_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s"
        )

        # 初始化处理器
        self._console_handler = None
        self._file_handler = None
        self._seq_configured = False

    def _get_console_handler(self) -> logging.StreamHandler:
        """
        获取或创建控制台处理器。

        Returns:
            logging.StreamHandler: 控制台处理器
        """
        if self._console_handler is None:
            self._console_handler = logging.StreamHandler()
            self._console_handler.setLevel(self.default_level)
            self._console_handler.setFormatter(self.console_formatter)
        return self._console_handler

    def _get_file_handler(self) -> TimedRotatingFileHandler:
        """
        获取或创建文件处理器（按天轮转）。

        Returns:
            TimedRotatingFileHandler: 文件处理器
        """
        if self._file_handler is None:
            log_filename = os.path.join(self.log_dir, f"{self.name}-{os.getpid()}.log")
            self._file_handler = TimedRotatingFileHandler(
                filename=log_filename,
                when="midnight",
                interval=1,
                backupCount=30,  # 保留30天的日志
                encoding="utf-8",
            )
            self._file_handler.setLevel(self.default_level)
            self._file_handler.setFormatter(self.file_formatter)
            # 设置文件名后缀格式
            self._file_handler.suffix = "%Y-%m-%d"
        return self._file_handler

    def _configure_seq(self):
        """
        配置 Seq 远程日志。
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if self.seq_server_url and not self._seq_configured:
            seqlog.configure_from_dict(
                {
                    "version": 1,
                    "disable_existing_loggers": False,
                    "handlers": {
                        "seq": {
                            "class": "seqlog.structured_logging.SeqLogHandler",
                            "server_url": self.seq_server_url,
                            "api_key": self.seq_api_key,
                            "batch_size": 10,
                            "auto_flush_timeout": 2,
                        }
                    },
                    "loggers": {
                        self.name: {
                            "handlers": ["seq"],
                            "level": "DEBUG",
                            "propagate": False,
                        }
                    },
                }
            )
            self._seq_configured = True

    def enable_console(self, level: Optional[str] = None):
        """
        启用控制台日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_console_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_console(self):
        """
        禁用控制台日志输出。
        """
        if self._console_handler and self._console_handler in self.logger.handlers:
            self.logger.removeHandler(self._console_handler)

    def enable_file(self, level: Optional[str] = None):
        """
        启用文件日志输出（按天保存）。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_file_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_file(self):
        """
        禁用文件日志输出。
        """
        if self._file_handler and self._file_handler in self.logger.handlers:
            self.logger.removeHandler(self._file_handler)

    def enable_seq(self, level: Optional[str] = None):
        """
        启用 Seq 远程日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")
        self._configure_seq()
        if level:
            # 对于 Seq，我们需要通过 seqlog 的配置来设置级别
            seq_logger = logging.getLogger(self.name)
            seq_logger.setLevel(getattr(logging, level.upper()))

    def log_to_console(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到控制台的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用控制台，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if file_was_enabled:
            self.logger.removeHandler(self._file_handler)

        # 确保控制台处理器已添加
        if not console_was_enabled:
            self.enable_console()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if not console_was_enabled:
            self.disable_console()
        if file_was_enabled:
            self.logger.addHandler(self._file_handler)

    def log_to_file(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到文件的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用文件，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if console_was_enabled:
            self.logger.removeHandler(self._console_handler)

        # 确保文件处理器已添加
        if not file_was_enabled:
            self.enable_file()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if console_was_enabled:
            self.logger.addHandler(self._console_handler)
        if not file_was_enabled:
            self.disable_file()

    def log_to_seq(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到 Seq 远程服务器的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据（将作为 Seq 的属性）
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")

        self._configure_seq()

        # 使用 seqlog 的结构化日志功能
        seq_logger = logging.getLogger(self.name)
        log_method = getattr(seq_logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 对于 Seq，直接使用 kwargs 作为结构化数据（先做 JSON 安全化）
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            log_method(message, exc_info=exc_info, stack_info=stack_info, **safe_kwargs)
        else:
            log_method(message, exc_info=exc_info, stack_info=stack_info)

    def _log(self, message: str, level: str = "INFO", **kwargs):
        """
        内部日志记录方法。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        log_method = getattr(self.logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 将剩余的 kwargs 作为 JSON 附加到消息中
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            extra_info = json.dumps(safe_kwargs, ensure_ascii=False)
            message = f"{message} | {extra_info}"

        # 调用日志方法，传递特殊参数
        log_method(message, exc_info=exc_info, stack_info=stack_info)

    # 便捷方法
    def debug(self, message: str, **kwargs):
        """
        记录 DEBUG 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "DEBUG", **kwargs)

    def info(self, message: str, **kwargs):
        """
        记录 INFO 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "INFO", **kwargs)

    def warning(self, message: str, **kwargs):
        """
        记录 WARNING 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "WARNING", **kwargs)

    def error(self, message: str, **kwargs):
        """
        记录 ERROR 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据，支持特殊参数：
                - exc_info (bool/tuple): 如果为 True，将自动添加异常追踪信息
                - stack_info (bool): 如果为 True，将添加调用堆栈信息
        """
        self._log(message, "ERROR", **kwargs)

    def critical(self, message: str, **kwargs):
        """
        记录 CRITICAL 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "CRITICAL", **kwargs)


# 导出便捷函数
def get_logger(name: str, **kwargs) -> WoodyLogger:
    """
    获取一个新的日志记录器实例。

    Args:
        name (str): 日志记录器名称
        **kwargs: 传递给 WoodyLogger 的其他参数

    Returns:
        WoodyLogger: 日志记录器实例
    """
    return WoodyLogger(name, **kwargs)

```

---

**User**

请先说你的修改方案，先不要输出代码

---

**Cursor**

### 修改方案（不贴代码）

- 核心目标：避免多进程同时写同一个日志文件导致的轮转/写入竞态。

- 方案选项（按改动成本由低到高）:
  - 方案A（推荐，最小改动）：按进程拆分文件
    - 将文件日志名改为包含进程 PID（例如 `ai-reviewer-<pid>.log`），每个 Gunicorn worker 只写自己的文件，继续使用 `TimedRotatingFileHandler`，保持按天轮转。
    - 优点：无需引入依赖，改动最小；缺点：同一天会有多份日志文件。
  - 方案B（更简单）：关闭文件日志，仅用 Seq + 控制台
    - 交由集中式日志系统/容器日志收集；在容器/云环境下是最佳实践。
    - 优点：完全规避文件并发；缺点：本地排障时不在磁盘留存。
  - 方案C（单文件诉求）：使用多进程安全的文件 handler
    - 将 `TimedRotatingFileHandler` 替换为 `ConcurrentRotatingFileHandler`（`concurrent-log-handler`），依赖文件锁保证多进程写单文件安全。
    - 优点：只有一个文件；缺点：引入额外依赖与少量配置。
  - 方案D（最高可靠）：队列聚合写
    - 各 worker 用 `QueueHandler` 将日志推到队列，由单独的 `QueueListener`/专用进程串行写文件。
    - 优点：从根上解决并发写；缺点：实现复杂度最高。

- 配置建议（可与任一方案组合）:
  - 增加环境变量开关：`ENABLE_FILE_LOG=true/false`，生产默认关闭文件日志，仅用 Seq；本地可打开。
  - 保留 `request_id` 等结构化字段，便于跨文件/平台关联。
  - 不在高并发路径动态增删 handler，避免 handler 级别的状态抖动。

你确认选哪一个？我建议优先采用方案A或B：A 改动最小、立即生效；B 最干净、符合云原生实践。选定后我再为你落地实现。 

- 变更小结
  - 方案A：每个进程独立日志文件，规避多进程文件写入竞态。
  - 方案B：停用文件日志，完全消除文件级并发问题。

---

**User**

我希望日志文件在本地留存，并且是按天保留单文件，应该使用方案C还是方案D

---

**Cursor**

建议用方案C：它原生支持多进程安全的“单文件按天轮转”，改动小、见效快，足以满足本地单文件留存；方案D只在极高吞吐或需要最强一致性/跨进程集中写入（如网络文件系统、严格日志顺序）时才值得上，复杂度明显更高。 

- 选择指引
  - 方案C：优先推荐，替换为多进程安全的文件处理器即可。
  - 方案D：超高并发/严格时序/网络盘场景再考虑。

---

**User**

为我实现方案C

---

**Cursor**

我先确认 concurrent-log-handler 是否提供基于时间的多进程安全轮转类，以确定具体实现方式。

我将把文件日志处理器替换为 concurrent-log-handler 的 ConcurrentTimedRotatingFileHandler（单文件按天轮转，支持多进程），恢复统一文件名（去掉 PID），并在 requirements.txt 中新增依赖；随后检查 lints。

我将检查 `woody_logger.py` 当前内容以确认需要的具体修改，然后替换为并发安全的按天轮转处理器，并在 `requirements.txt` 中加入依赖；随后检查lints。

我将创建待办来跟踪两步实现：替换并发安全文件处理器和添加依赖；现在先开始替换日志处理器为 ConcurrentTimedRotatingFileHandler，并更新 requirements。

我将把 TimedRotatingFileHandler 替换为 concurrent-log-handler 的 ConcurrentTimedRotatingFileHandler，并把日志文件名恢复为不含 PID 的单文件。然后我会在 requirements.txt 中添加 concurrent-log-handler 依赖并检查 lints。

```python
import logging
import os
from concurrent_log_handler import ConcurrentTimedRotatingFileHandler
from typing import Optional
import json

try:
    import seqlog

    HAS_SEQLOG = True
except ImportError:
    HAS_SEQLOG = False
    print("警告: seqlog 未安装，远程日志功能将不可用。请运行: pip install seqlog")


def _make_json_safe(obj):
    """
    将对象转换为可被 json.dumps 处理的安全结构。

    - 支持内置类型、dict、list/tuple/set
    - 对 Pydantic v2 使用 model_dump，v1 使用 dict
    - 回退到 __dict__，最后回退到 str
    """
    if obj is None or isinstance(obj, (str, int, float, bool)):
        return obj
    if isinstance(obj, dict):
        return {str(k): _make_json_safe(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple, set)):
        return [_make_json_safe(v) for v in obj]
    # Pydantic v2
    if hasattr(obj, "model_dump") and callable(getattr(obj, "model_dump")):
        try:
            return _make_json_safe(obj.model_dump())
        except Exception:
            pass
    # Pydantic v1
    if hasattr(obj, "dict") and callable(getattr(obj, "dict")):
        try:
            return _make_json_safe(obj.dict())
        except Exception:
            pass
    # 其它对象：尝试使用 __dict__
    if hasattr(obj, "__dict__"):
        try:
            return _make_json_safe(vars(obj))
        except Exception:
            pass
    # 最后兜底：如果仍不可序列化则转字符串
    try:
        json.dumps(obj)
        return obj
    except TypeError:
        return str(obj)


class WoodyLogger:
    """
    多功能日志记录器，支持控制台、远程 Seq 服务器和本地文件日志。

    特性：
    - 支持多种输出目标（控制台、文件、Seq服务器）
    - 支持按天轮转的日志文件
    - 支持结构化日志和额外上下文数据
    - 支持 exc_info、stack_info 等标准 logging 参数

    额外数据处理：
    - 可以直接通过 kwargs 传递额外数据：logger.info("消息", user_id=123)
    - kwargs 数据会以 JSON 格式附加到消息末尾

    Args:
        name (str): 日志记录器名称
        log_dir (str): 日志文件保存目录，默认为 'logs'
        seq_server_url (str, optional): Seq 服务器 URL
        seq_api_key (str, optional): Seq API 密钥
        default_level (str): 默认日志级别，默认为 'INFO'
    """

    def __init__(
        self,
        name: str,
        log_dir: str = "logs",
        seq_server_url: Optional[str] = None,
        seq_api_key: Optional[str] = None,
        default_level: str = "INFO",
    ):
        """
        初始化 WoodyLogger。

        Args:
            name (str): 日志记录器名称
            log_dir (str): 日志文件保存目录
            seq_server_url (str, optional): Seq 服务器 URL
            seq_api_key (str, optional): Seq API 密钥
            default_level (str): 默认日志级别
        """
        self.name = name
        self.log_dir = log_dir
        self.seq_server_url = seq_server_url
        self.seq_api_key = seq_api_key
        self.default_level = getattr(logging, default_level.upper())

        # 创建日志目录
        os.makedirs(log_dir, exist_ok=True)

        # 创建主日志记录器
        self.logger = logging.getLogger(name)
        self.logger.setLevel(logging.DEBUG)  # 设置最低级别，具体由处理器控制

        # 避免重复添加处理器
        self.logger.handlers = []

        # 设置日志格式
        self.console_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        )
        self.file_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s"
        )

        # 初始化处理器
        self._console_handler = None
        self._file_handler = None
        self._seq_configured = False

    def _get_console_handler(self) -> logging.StreamHandler:
        """
        获取或创建控制台处理器。

        Returns:
            logging.StreamHandler: 控制台处理器
        """
        if self._console_handler is None:
            self._console_handler = logging.StreamHandler()
            self._console_handler.setLevel(self.default_level)
            self._console_handler.setFormatter(self.console_formatter)
        return self._console_handler

    def _get_file_handler(self) -> ConcurrentTimedRotatingFileHandler:
        """
        获取或创建文件处理器（按天轮转）。

        Returns:
            ConcurrentTimedRotatingFileHandler: 文件处理器
        """
        if self._file_handler is None:
            log_filename = os.path.join(self.log_dir, f"{self.name}.log")
            self._file_handler = ConcurrentTimedRotatingFileHandler(
                filename=log_filename,
                when="midnight",
                interval=1,
                backupCount=30,  # 保留30天的日志
                encoding="utf-8",
            )
            self._file_handler.setLevel(self.default_level)
            self._file_handler.setFormatter(self.file_formatter)
            # 设置文件名后缀格式
            self._file_handler.suffix = "%Y-%m-%d"
        return self._file_handler

    def _configure_seq(self):
        """
        配置 Seq 远程日志。
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if self.seq_server_url and not self._seq_configured:
            seqlog.configure_from_dict(
                {
                    "version": 1,
                    "disable_existing_loggers": False,
                    "handlers": {
                        "seq": {
                            "class": "seqlog.structured_logging.SeqLogHandler",
                            "server_url": self.seq_server_url,
                            "api_key": self.seq_api_key,
                            "batch_size": 10,
                            "auto_flush_timeout": 2,
                        }
                    },
                    "loggers": {
                        self.name: {
                            "handlers": ["seq"],
                            "level": "DEBUG",
                            "propagate": False,
                        }
                    },
                }
            )
            self._seq_configured = True

    def enable_console(self, level: Optional[str] = None):
        """
        启用控制台日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_console_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_console(self):
        """
        禁用控制台日志输出。
        """
        if self._console_handler and self._console_handler in self.logger.handlers:
            self.logger.removeHandler(self._console_handler)

    def enable_file(self, level: Optional[str] = None):
        """
        启用文件日志输出（按天保存）。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_file_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_file(self):
        """
        禁用文件日志输出。
        """
        if self._file_handler and self._file_handler in self.logger.handlers:
            self.logger.removeHandler(self._file_handler)

    def enable_seq(self, level: Optional[str] = None):
        """
        启用 Seq 远程日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")
        self._configure_seq()
        if level:
            # 对于 Seq，我们需要通过 seqlog 的配置来设置级别
            seq_logger = logging.getLogger(self.name)
            seq_logger.setLevel(getattr(logging, level.upper()))

    def log_to_console(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到控制台的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用控制台，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if file_was_enabled:
            self.logger.removeHandler(self._file_handler)

        # 确保控制台处理器已添加
        if not console_was_enabled:
            self.enable_console()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if not console_was_enabled:
            self.disable_console()
        if file_was_enabled:
            self.logger.addHandler(self._file_handler)

    def log_to_file(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到文件的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用文件，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if console_was_enabled:
            self.logger.removeHandler(self._console_handler)

        # 确保文件处理器已添加
        if not file_was_enabled:
            self.enable_file()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if console_was_enabled:
            self.logger.addHandler(self._console_handler)
        if not file_was_enabled:
            self.disable_file()

    def log_to_seq(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到 Seq 远程服务器的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据（将作为 Seq 的属性）
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")

        self._configure_seq()

        # 使用 seqlog 的结构化日志功能
        seq_logger = logging.getLogger(self.name)
        log_method = getattr(seq_logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 对于 Seq，直接使用 kwargs 作为结构化数据（先做 JSON 安全化）
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            log_method(message, exc_info=exc_info, stack_info=stack_info, **safe_kwargs)
        else:
            log_method(message, exc_info=exc_info, stack_info=stack_info)

    def _log(self, message: str, level: str = "INFO", **kwargs):
        """
        内部日志记录方法。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        log_method = getattr(self.logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 将剩余的 kwargs 作为 JSON 附加到消息中
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            extra_info = json.dumps(safe_kwargs, ensure_ascii=False)
            message = f"{message} | {extra_info}"

        # 调用日志方法，传递特殊参数
        log_method(message, exc_info=exc_info, stack_info=stack_info)

    # 便捷方法
    def debug(self, message: str, **kwargs):
        """
        记录 DEBUG 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "DEBUG", **kwargs)

    def info(self, message: str, **kwargs):
        """
        记录 INFO 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "INFO", **kwargs)

    def warning(self, message: str, **kwargs):
        """
        记录 WARNING 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "WARNING", **kwargs)

    def error(self, message: str, **kwargs):
        """
        记录 ERROR 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据，支持特殊参数：
                - exc_info (bool/tuple): 如果为 True，将自动添加异常追踪信息
                - stack_info (bool): 如果为 True，将添加调用堆栈信息
        """
        self._log(message, "ERROR", **kwargs)

    def critical(self, message: str, **kwargs):
        """
        记录 CRITICAL 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "CRITICAL", **kwargs)


# 导出便捷函数
def get_logger(name: str, **kwargs) -> WoodyLogger:
    """
    获取一个新的日志记录器实例。

    Args:
        name (str): 日志记录器名称
        **kwargs: 传递给 WoodyLogger 的其他参数

    Returns:
        WoodyLogger: 日志记录器实例
    """
    return WoodyLogger(name, **kwargs)

```

```pip-requirements
fastapi
pydantic
aiolimiter
python-dotenv
gunicorn
uvicorn
langgraph
langchain-core
langchain-openai
aiohttp
Pillow
seqlog
pytest
concurrent-log-handler


```

我已开始按你的选择执行方案C：替换为多进程安全的按天轮转处理器，并添加依赖。现在我将修正导入为可选导入并提供回退，以消除刚才的 linter 警告，然后再次检查 lints。

```python
import logging
import os
try:
    from concurrent_log_handler import (
        ConcurrentTimedRotatingFileHandler as SafeTimedRotatingFileHandler,
    )
    HAS_CONCURRENT_LOG_HANDLER = True
except ImportError:
    from logging.handlers import TimedRotatingFileHandler as SafeTimedRotatingFileHandler
    HAS_CONCURRENT_LOG_HANDLER = False
from typing import Optional
import json

try:
    import seqlog

    HAS_SEQLOG = True
except ImportError:
    HAS_SEQLOG = False
    print("警告: seqlog 未安装，远程日志功能将不可用。请运行: pip install seqlog")


def _make_json_safe(obj):
    """
    将对象转换为可被 json.dumps 处理的安全结构。

    - 支持内置类型、dict、list/tuple/set
    - 对 Pydantic v2 使用 model_dump，v1 使用 dict
    - 回退到 __dict__，最后回退到 str
    """
    if obj is None or isinstance(obj, (str, int, float, bool)):
        return obj
    if isinstance(obj, dict):
        return {str(k): _make_json_safe(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple, set)):
        return [_make_json_safe(v) for v in obj]
    # Pydantic v2
    if hasattr(obj, "model_dump") and callable(getattr(obj, "model_dump")):
        try:
            return _make_json_safe(obj.model_dump())
        except Exception:
            pass
    # Pydantic v1
    if hasattr(obj, "dict") and callable(getattr(obj, "dict")):
        try:
            return _make_json_safe(obj.dict())
        except Exception:
            pass
    # 其它对象：尝试使用 __dict__
    if hasattr(obj, "__dict__"):
        try:
            return _make_json_safe(vars(obj))
        except Exception:
            pass
    # 最后兜底：如果仍不可序列化则转字符串
    try:
        json.dumps(obj)
        return obj
    except TypeError:
        return str(obj)


class WoodyLogger:
    """
    多功能日志记录器，支持控制台、远程 Seq 服务器和本地文件日志。

    特性：
    - 支持多种输出目标（控制台、文件、Seq服务器）
    - 支持按天轮转的日志文件
    - 支持结构化日志和额外上下文数据
    - 支持 exc_info、stack_info 等标准 logging 参数

    额外数据处理：
    - 可以直接通过 kwargs 传递额外数据：logger.info("消息", user_id=123)
    - kwargs 数据会以 JSON 格式附加到消息末尾

    Args:
        name (str): 日志记录器名称
        log_dir (str): 日志文件保存目录，默认为 'logs'
        seq_server_url (str, optional): Seq 服务器 URL
        seq_api_key (str, optional): Seq API 密钥
        default_level (str): 默认日志级别，默认为 'INFO'
    """

    def __init__(
        self,
        name: str,
        log_dir: str = "logs",
        seq_server_url: Optional[str] = None,
        seq_api_key: Optional[str] = None,
        default_level: str = "INFO",
    ):
        """
        初始化 WoodyLogger。

        Args:
            name (str): 日志记录器名称
            log_dir (str): 日志文件保存目录
            seq_server_url (str, optional): Seq 服务器 URL
            seq_api_key (str, optional): Seq API 密钥
            default_level (str): 默认日志级别
        """
        self.name = name
        self.log_dir = log_dir
        self.seq_server_url = seq_server_url
        self.seq_api_key = seq_api_key
        self.default_level = getattr(logging, default_level.upper())

        # 创建日志目录
        os.makedirs(log_dir, exist_ok=True)

        # 创建主日志记录器
        self.logger = logging.getLogger(name)
        self.logger.setLevel(logging.DEBUG)  # 设置最低级别，具体由处理器控制

        # 避免重复添加处理器
        self.logger.handlers = []

        # 设置日志格式
        self.console_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        )
        self.file_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s"
        )

        # 初始化处理器
        self._console_handler = None
        self._file_handler = None
        self._seq_configured = False

    def _get_console_handler(self) -> logging.StreamHandler:
        """
        获取或创建控制台处理器。

        Returns:
            logging.StreamHandler: 控制台处理器
        """
        if self._console_handler is None:
            self._console_handler = logging.StreamHandler()
            self._console_handler.setLevel(self.default_level)
            self._console_handler.setFormatter(self.console_formatter)
        return self._console_handler

    def _get_file_handler(self) -> logging.Handler:
        """
        获取或创建文件处理器（按天轮转）。

        Returns:
            logging.Handler: 文件处理器
        """
        if self._file_handler is None:
            log_filename = os.path.join(self.log_dir, f"{self.name}.log")
            self._file_handler = SafeTimedRotatingFileHandler(
                filename=log_filename,
                when="midnight",
                interval=1,
                backupCount=30,  # 保留30天的日志
                encoding="utf-8",
            )
            self._file_handler.setLevel(self.default_level)
            self._file_handler.setFormatter(self.file_formatter)
            # 设置文件名后缀格式
            self._file_handler.suffix = "%Y-%m-%d"
        return self._file_handler

    def _configure_seq(self):
        """
        配置 Seq 远程日志。
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if self.seq_server_url and not self._seq_configured:
            seqlog.configure_from_dict(
                {
                    "version": 1,
                    "disable_existing_loggers": False,
                    "handlers": {
                        "seq": {
                            "class": "seqlog.structured_logging.SeqLogHandler",
                            "server_url": self.seq_server_url,
                            "api_key": self.seq_api_key,
                            "batch_size": 10,
                            "auto_flush_timeout": 2,
                        }
                    },
                    "loggers": {
                        self.name: {
                            "handlers": ["seq"],
                            "level": "DEBUG",
                            "propagate": False,
                        }
                    },
                }
            )
            self._seq_configured = True

    def enable_console(self, level: Optional[str] = None):
        """
        启用控制台日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_console_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_console(self):
        """
        禁用控制台日志输出。
        """
        if self._console_handler and self._console_handler in self.logger.handlers:
            self.logger.removeHandler(self._console_handler)

    def enable_file(self, level: Optional[str] = None):
        """
        启用文件日志输出（按天保存）。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_file_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_file(self):
        """
        禁用文件日志输出。
        """
        if self._file_handler and self._file_handler in self.logger.handlers:
            self.logger.removeHandler(self._file_handler)

    def enable_seq(self, level: Optional[str] = None):
        """
        启用 Seq 远程日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")
        self._configure_seq()
        if level:
            # 对于 Seq，我们需要通过 seqlog 的配置来设置级别
            seq_logger = logging.getLogger(self.name)
            seq_logger.setLevel(getattr(logging, level.upper()))

    def log_to_console(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到控制台的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用控制台，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if file_was_enabled:
            self.logger.removeHandler(self._file_handler)

        # 确保控制台处理器已添加
        if not console_was_enabled:
            self.enable_console()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if not console_was_enabled:
            self.disable_console()
        if file_was_enabled:
            self.logger.addHandler(self._file_handler)

    def log_to_file(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到文件的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用文件，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if console_was_enabled:
            self.logger.removeHandler(self._console_handler)

        # 确保文件处理器已添加
        if not file_was_enabled:
            self.enable_file()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if console_was_enabled:
            self.logger.addHandler(self._console_handler)
        if not file_was_enabled:
            self.disable_file()

    def log_to_seq(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到 Seq 远程服务器的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据（将作为 Seq 的属性）
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")

        self._configure_seq()

        # 使用 seqlog 的结构化日志功能
        seq_logger = logging.getLogger(self.name)
        log_method = getattr(seq_logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 对于 Seq，直接使用 kwargs 作为结构化数据（先做 JSON 安全化）
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            log_method(message, exc_info=exc_info, stack_info=stack_info, **safe_kwargs)
        else:
            log_method(message, exc_info=exc_info, stack_info=stack_info)

    def _log(self, message: str, level: str = "INFO", **kwargs):
        """
        内部日志记录方法。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        log_method = getattr(self.logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 将剩余的 kwargs 作为 JSON 附加到消息中
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            extra_info = json.dumps(safe_kwargs, ensure_ascii=False)
            message = f"{message} | {extra_info}"

        # 调用日志方法，传递特殊参数
        log_method(message, exc_info=exc_info, stack_info=stack_info)

    # 便捷方法
    def debug(self, message: str, **kwargs):
        """
        记录 DEBUG 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "DEBUG", **kwargs)

    def info(self, message: str, **kwargs):
        """
        记录 INFO 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "INFO", **kwargs)

    def warning(self, message: str, **kwargs):
        """
        记录 WARNING 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "WARNING", **kwargs)

    def error(self, message: str, **kwargs):
        """
        记录 ERROR 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据，支持特殊参数：
                - exc_info (bool/tuple): 如果为 True，将自动添加异常追踪信息
                - stack_info (bool): 如果为 True，将添加调用堆栈信息
        """
        self._log(message, "ERROR", **kwargs)

    def critical(self, message: str, **kwargs):
        """
        记录 CRITICAL 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "CRITICAL", **kwargs)


# 导出便捷函数
def get_logger(name: str, **kwargs) -> WoodyLogger:
    """
    获取一个新的日志记录器实例。

    Args:
        name (str): 日志记录器名称
        **kwargs: 传递给 WoodyLogger 的其他参数

    Returns:
        WoodyLogger: 日志记录器实例
    """
    return WoodyLogger(name, **kwargs)

```

```python
try:
    from concurrent_log_handler import (
        ConcurrentTimedRotatingFileHandler as SafeTimedRotatingFileHandler,
    )
    HAS_CONCURRENT_LOG_HANDLER = True
except ImportError:
    from logging.handlers import TimedRotatingFileHandler as SafeTimedRotatingFileHandler
    HAS_CONCURRENT_LOG_HANDLER = False
```

我刚完成了两项修改：已将文件日志处理器替换为并发安全的按天轮转实现，并在依赖中加入 concurrent-log-handler。接下来我会静默处理一个导入的linter警告并再次检查。

```python
import logging
import os

try:
    from concurrent_log_handler import (  # type: ignore[import]
        ConcurrentTimedRotatingFileHandler as SafeTimedRotatingFileHandler,
    )

    HAS_CONCURRENT_LOG_HANDLER = True
except ImportError:
    from logging.handlers import (
        TimedRotatingFileHandler as SafeTimedRotatingFileHandler,
    )

    HAS_CONCURRENT_LOG_HANDLER = False
from typing import Optional
import json

try:
    import seqlog

    HAS_SEQLOG = True
except ImportError:
    HAS_SEQLOG = False
    print("警告: seqlog 未安装，远程日志功能将不可用。请运行: pip install seqlog")


def _make_json_safe(obj):
    """
    将对象转换为可被 json.dumps 处理的安全结构。

    - 支持内置类型、dict、list/tuple/set
    - 对 Pydantic v2 使用 model_dump，v1 使用 dict
    - 回退到 __dict__，最后回退到 str
    """
    if obj is None or isinstance(obj, (str, int, float, bool)):
        return obj
    if isinstance(obj, dict):
        return {str(k): _make_json_safe(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple, set)):
        return [_make_json_safe(v) for v in obj]
    # Pydantic v2
    if hasattr(obj, "model_dump") and callable(getattr(obj, "model_dump")):
        try:
            return _make_json_safe(obj.model_dump())
        except Exception:
            pass
    # Pydantic v1
    if hasattr(obj, "dict") and callable(getattr(obj, "dict")):
        try:
            return _make_json_safe(obj.dict())
        except Exception:
            pass
    # 其它对象：尝试使用 __dict__
    if hasattr(obj, "__dict__"):
        try:
            return _make_json_safe(vars(obj))
        except Exception:
            pass
    # 最后兜底：如果仍不可序列化则转字符串
    try:
        json.dumps(obj)
        return obj
    except TypeError:
        return str(obj)


class WoodyLogger:
    """
    多功能日志记录器，支持控制台、远程 Seq 服务器和本地文件日志。

    特性：
    - 支持多种输出目标（控制台、文件、Seq服务器）
    - 支持按天轮转的日志文件
    - 支持结构化日志和额外上下文数据
    - 支持 exc_info、stack_info 等标准 logging 参数

    额外数据处理：
    - 可以直接通过 kwargs 传递额外数据：logger.info("消息", user_id=123)
    - kwargs 数据会以 JSON 格式附加到消息末尾

    Args:
        name (str): 日志记录器名称
        log_dir (str): 日志文件保存目录，默认为 'logs'
        seq_server_url (str, optional): Seq 服务器 URL
        seq_api_key (str, optional): Seq API 密钥
        default_level (str): 默认日志级别，默认为 'INFO'
    """

    def __init__(
        self,
        name: str,
        log_dir: str = "logs",
        seq_server_url: Optional[str] = None,
        seq_api_key: Optional[str] = None,
        default_level: str = "INFO",
    ):
        """
        初始化 WoodyLogger。

        Args:
            name (str): 日志记录器名称
            log_dir (str): 日志文件保存目录
            seq_server_url (str, optional): Seq 服务器 URL
            seq_api_key (str, optional): Seq API 密钥
            default_level (str): 默认日志级别
        """
        self.name = name
        self.log_dir = log_dir
        self.seq_server_url = seq_server_url
        self.seq_api_key = seq_api_key
        self.default_level = getattr(logging, default_level.upper())

        # 创建日志目录
        os.makedirs(log_dir, exist_ok=True)

        # 创建主日志记录器
        self.logger = logging.getLogger(name)
        self.logger.setLevel(logging.DEBUG)  # 设置最低级别，具体由处理器控制

        # 避免重复添加处理器
        self.logger.handlers = []

        # 设置日志格式
        self.console_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        )
        self.file_formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s"
        )

        # 初始化处理器
        self._console_handler = None
        self._file_handler = None
        self._seq_configured = False

    def _get_console_handler(self) -> logging.StreamHandler:
        """
        获取或创建控制台处理器。

        Returns:
            logging.StreamHandler: 控制台处理器
        """
        if self._console_handler is None:
            self._console_handler = logging.StreamHandler()
            self._console_handler.setLevel(self.default_level)
            self._console_handler.setFormatter(self.console_formatter)
        return self._console_handler

    def _get_file_handler(self) -> logging.Handler:
        """
        获取或创建文件处理器（按天轮转）。

        Returns:
            logging.Handler: 文件处理器
        """
        if self._file_handler is None:
            log_filename = os.path.join(self.log_dir, f"{self.name}.log")
            self._file_handler = SafeTimedRotatingFileHandler(
                filename=log_filename,
                when="midnight",
                interval=1,
                backupCount=30,  # 保留30天的日志
                encoding="utf-8",
            )
            self._file_handler.setLevel(self.default_level)
            self._file_handler.setFormatter(self.file_formatter)
            # 设置文件名后缀格式
            self._file_handler.suffix = "%Y-%m-%d"
        return self._file_handler

    def _configure_seq(self):
        """
        配置 Seq 远程日志。
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if self.seq_server_url and not self._seq_configured:
            seqlog.configure_from_dict(
                {
                    "version": 1,
                    "disable_existing_loggers": False,
                    "handlers": {
                        "seq": {
                            "class": "seqlog.structured_logging.SeqLogHandler",
                            "server_url": self.seq_server_url,
                            "api_key": self.seq_api_key,
                            "batch_size": 10,
                            "auto_flush_timeout": 2,
                        }
                    },
                    "loggers": {
                        self.name: {
                            "handlers": ["seq"],
                            "level": "DEBUG",
                            "propagate": False,
                        }
                    },
                }
            )
            self._seq_configured = True

    def enable_console(self, level: Optional[str] = None):
        """
        启用控制台日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_console_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_console(self):
        """
        禁用控制台日志输出。
        """
        if self._console_handler and self._console_handler in self.logger.handlers:
            self.logger.removeHandler(self._console_handler)

    def enable_file(self, level: Optional[str] = None):
        """
        启用文件日志输出（按天保存）。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        handler = self._get_file_handler()
        if level:
            handler.setLevel(getattr(logging, level.upper()))
        if handler not in self.logger.handlers:
            self.logger.addHandler(handler)

    def disable_file(self):
        """
        禁用文件日志输出。
        """
        if self._file_handler and self._file_handler in self.logger.handlers:
            self.logger.removeHandler(self._file_handler)

    def enable_seq(self, level: Optional[str] = None):
        """
        启用 Seq 远程日志输出。

        Args:
            level (str, optional): 日志级别，如果不指定则使用默认级别
        """
        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")
        self._configure_seq()
        if level:
            # 对于 Seq，我们需要通过 seqlog 的配置来设置级别
            seq_logger = logging.getLogger(self.name)
            seq_logger.setLevel(getattr(logging, level.upper()))

    def log_to_console(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到控制台的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用控制台，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if file_was_enabled:
            self.logger.removeHandler(self._file_handler)

        # 确保控制台处理器已添加
        if not console_was_enabled:
            self.enable_console()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if not console_was_enabled:
            self.disable_console()
        if file_was_enabled:
            self.logger.addHandler(self._file_handler)

    def log_to_file(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到文件的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        # 临时启用文件，记录日志，然后恢复原状态
        console_was_enabled = (
            self._console_handler in self.logger.handlers
            if self._console_handler
            else False
        )
        file_was_enabled = (
            self._file_handler in self.logger.handlers if self._file_handler else False
        )

        # 暂时移除其他处理器
        if console_was_enabled:
            self.logger.removeHandler(self._console_handler)

        # 确保文件处理器已添加
        if not file_was_enabled:
            self.enable_file()

        # 记录日志
        self._log(message, level, **kwargs)

        # 恢复原状态
        if console_was_enabled:
            self.logger.addHandler(self._console_handler)
        if not file_was_enabled:
            self.disable_file()

    def log_to_seq(self, message: str, level: str = "INFO", **kwargs):
        """
        仅输出到 Seq 远程服务器的日志。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据（将作为 Seq 的属性）
        """
        if not HAS_SEQLOG:
            raise ImportError("seqlog 未安装，无法使用远程日志功能")

        if not self.seq_server_url:
            raise ValueError("Seq 服务器 URL 未配置")

        self._configure_seq()

        # 使用 seqlog 的结构化日志功能
        seq_logger = logging.getLogger(self.name)
        log_method = getattr(seq_logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 对于 Seq，直接使用 kwargs 作为结构化数据（先做 JSON 安全化）
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            log_method(message, exc_info=exc_info, stack_info=stack_info, **safe_kwargs)
        else:
            log_method(message, exc_info=exc_info, stack_info=stack_info)

    def _log(self, message: str, level: str = "INFO", **kwargs):
        """
        内部日志记录方法。

        Args:
            message (str): 日志消息
            level (str): 日志级别
            **kwargs: 额外的结构化数据
        """
        log_method = getattr(self.logger, level.lower())

        # 提取特殊的 logging 参数
        exc_info = kwargs.pop("exc_info", None)
        stack_info = kwargs.pop("stack_info", None)

        # 将剩余的 kwargs 作为 JSON 附加到消息中
        if kwargs:
            safe_kwargs = _make_json_safe(kwargs)
            extra_info = json.dumps(safe_kwargs, ensure_ascii=False)
            message = f"{message} | {extra_info}"

        # 调用日志方法，传递特殊参数
        log_method(message, exc_info=exc_info, stack_info=stack_info)

    # 便捷方法
    def debug(self, message: str, **kwargs):
        """
        记录 DEBUG 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "DEBUG", **kwargs)

    def info(self, message: str, **kwargs):
        """
        记录 INFO 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "INFO", **kwargs)

    def warning(self, message: str, **kwargs):
        """
        记录 WARNING 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "WARNING", **kwargs)

    def error(self, message: str, **kwargs):
        """
        记录 ERROR 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据，支持特殊参数：
                - exc_info (bool/tuple): 如果为 True，将自动添加异常追踪信息
                - stack_info (bool): 如果为 True，将添加调用堆栈信息
        """
        self._log(message, "ERROR", **kwargs)

    def critical(self, message: str, **kwargs):
        """
        记录 CRITICAL 级别日志。

        Args:
            message (str): 日志消息
            **kwargs: 额外的结构化数据
        """
        self._log(message, "CRITICAL", **kwargs)


# 导出便捷函数
def get_logger(name: str, **kwargs) -> WoodyLogger:
    """
    获取一个新的日志记录器实例。

    Args:
        name (str): 日志记录器名称
        **kwargs: 传递给 WoodyLogger 的其他参数

    Returns:
        WoodyLogger: 日志记录器实例
    """
    return WoodyLogger(name, **kwargs)

```

已按方案C实现：
- 在`woody_logger.py`中用并发安全的`ConcurrentTimedRotatingFileHandler`（缺库则回退到标准`TimedRotatingFileHandler`），文件名恢复为`logs/ai-reviewer.log`按天轮转，单文件留存。
- 在`requirements.txt`添加`concurrent-log-handler`；并处理了导入的linter告警。

要点
- 多进程环境下将自动使用并发安全处理器；未安装依赖时仍可跑，但不具备多进程文件安全，建议安装。
- 生产环境推荐开启 Seq，同步保留本地单文件便于排障。

