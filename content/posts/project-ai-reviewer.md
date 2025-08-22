---
title: "Ai Reviewer"
date: 2025-08-21T17:00:57+08:00
# bookComments: false
# bookSearchExclude: false
---

# AI Reviewer

## 1. langchain_openai库中ChatOpenAI和OpenAI的区别

* ChatOpenAI面向现代聊天模型，这些模型支持多模态输入，工具调用和结构化输出等

* OpenAI面向旧的补全模型(gpt-3.5-turbo-instruct, text-davinci-003等)，这些模型只支持字符串输入和输出，不支持多模态输入，工具调用和结构化输出等现代特性

## 2. pydantic别名

* pydantic别名用于兼容历史/旧的字段名
* alias, ConfigDict, populate_by_name

```python

from pydantic import BaseModel, ConfigDict, Field


class User(BaseModel):
    user_id: str = Field(alias="id")


user1 = User(id="2")
# user2 = User(user_id="2") ❌ 用内部名初始化报错
print(user1)
# print(user2)


class User2(BaseModel):
    user_id: str = Field(alias="id")
    model_config = ConfigDict(
        populate_by_name=True
    )  # 设置 populate_by_name=True 后可用内部名进行初始化


user3 = User2(id="2")
user4 = User2(user_id="2")  # ✅
print(user3)
print(user4)


```

# 3. ChatOpenAI中model和model_name参数的区别

* model参数是model_name的别名，初始化用model或model_name都一样，最好是用model

# 4. langgraph图中如果有节点执行错误，并且重试超过最大次数，会有state输出吗？

* 不会有最终state输出，并且invoke/ainvoke会报错

# 5. 函数无返回值应该不写返回值类型，还是应该写->None?

* 如果其他函数有类型注解，就写->None,保持整体性。如果其他函数都没有类型注解，就不写返回值类型

# 6. any和typing.Any的区别

* any是python中的内置函数，判断可迭代对象中是否包含真值

```python

any([0, "", None, 5])   # True
any([])                 # False

```

* typing.Any表示任意类型
* 它们之间不是类似list和typing.List的关系

# 7. raise_for_status()

* raise_for_status()用来检查响应状态码，如果状态码是错误码，则会抛出异常，否则什么都不做

# 8. 0.0.0.0和127.0.0.1的区别

* 0.0.0.0上启动的服务外部可访问
* 127.0.0.1上启动的服务只本机可访问

# 9. python的真值和假值

* 假值: 数值零 (包括整数，浮点数，复数的零), False, None, 空序列/容器 (len(obj)==0)
* 真值: 除以上假值外的所有对象

# 10. langgraph的state schema类型的适用场景

* TypedDict: 最常用，大多数使用 (最快)
* dataclass: 需要设置默认值和必填字段时使用
* pydantic BaseModel: 需要递归数据校验时使用 (内部嵌套对象也需要进行数据校验) (最慢)

# 11

* docstring没有规范是三个单引号还是三个双引号，但最好是用三个双引号写

# 12

* 英文prompt中括号前后都要空一格，后面如果是句末（后面是,或.）则不需要空一格

```txt

* Compare the expected and actual values to determine if they match (True/False).✅

* Provide clear reasoning for your assessment in Chinese (limited to 250 characters), explaining why the values match or do not.✅

```

# 13. pytest单元测试

* 测试文件按功能模块进行划分，以test_开头
* 测试函数以test_开头
* 异步测试函数上方需要添加 @pytest.mark.asyncio注解
* 测试参数使用 @pytest.mark.parametrize注解

```python

import pytest
from utils import async_compress_and_encode_images_from_urls

@pytest.mark.parametrize(
    "image_urls",
    [
        [
            "https://trialedctest.oss-cn-hangzhou.aliyuncs.com/123.png?Expires=1755917451&OSSAccessKeyId=LTAItycD0loaPpSp&Signature=L2XaKIHaVoCgRKhPbsIZXmlO9Xw%3D"
        ]
    ],
)
@pytest.mark.asyncio
async def test_async_compress_and_encode_images_from_urls(image_urls):
    base64_images = await async_compress_and_encode_images_from_urls(
        request_id="123", image_urls=image_urls, quality=100
    )
    base64_images = [img for img in base64_images if img is not None]
    assert len(base64_images) > 0


```

# 14.
LangGraph实现按需并行扇出，需要多个节点就同时跑多个
详见：[LangGraph并行扇出实现]({{< relref "posts/LangGraph并行扇出实现.html" >}})

