---
title: "Workorder Sunqing Pdf Form to Excel"
date: 2025-08-15T13:55:24+08:00
# bookComments: false
# bookSearchExclude: false
---

# work order for sunqing in pdf_form_to_excel

1. pydantic模型用作langgraph的state的操作方式(https://langchain-ai.github.io/langgraph/how-tos/graph-api/#use-pydantic-models-for-graph-state)(官方文档有使用例子)
2. dataclass用作langgraph的state的操作方式(不好用，没pydantic方便)
3. dataclass用作langgraph的state比pydantic模型效率更高
4. langgraph的state类型可以是TypedDict、dataclass和pydantic model
5. pydantic BaseModel的model_dump()方法是将BaseModel类型转换成dict类型
6. pydantic BaseModel保存到excel文件里
7. structured outputs最外层模型必须是pydantic BaseModel，嵌套的内层模型可以是dataclass类型
8. python logging库的使用更进一步

## 9. uuid的使用

uuid.uuid4().hex 生成随机UUID,移除所有连字符，返回32位的纯十六进制字符串

| 版本  | 生成方式         | 确定性 | 适用场景             |
| ----- | ---------------- | ------ | -------------------- |
| UUID1 | 时间戳 + MAC地址 | 否     | 数据库主键、审计日志 |
| UUID3 | MD5哈希          | 是     | 内容标识符、缓存键   |
| UUID4 | 纯随机           | 否     | 临时标识符、随机令牌 |
| UUID5 | SHA-1哈希        | 是     | 内容标识符（更安全） |

## 10. python包模块管理理解更进一步

当执行 from graph.nodes import * 时，Python会：

* 首先查看 `__all__` 变量：如果 `__init__.py` 中定义了 `__all__`，则只导入 `__all__` 中列出的名称

* 如果没有 `__all__`：导入所有不以下划线开头的名称（但这些名称必须已经在 `__init__.py` 中被导入或定义）

* 如果 `__init__.py` 为空：什么都不导入！

总的来说就是：

* 模块导入先看`__all__`，如果`__all__`不为空的话，只导入 `__all__` 中列出的名称

* `__all__`为空的话再看`__init__.py`中的导入，如果`__init__.py`不为空，则导入`__init__.py`中导入的对象

* `__init__.py`为空的话则什么都不导入

即使`__init__.py`中导入带下划线对象，import *也不会导出下划线对象，必须显式引入

`__all__`中有带下划线对象，import *和显式引入都会导出下划线对象

## 11.python关键字参数收集器，位置参数收集器

*是位置参数收集器（收集到元组里），**是关键字参数收集器（收集到dict里）

```python
def my_function(name, age, **args):
    print(name)
    print(age)
    print(args)


my_function("张三", 25, city="北京", title="工程师") #调用必须要带关键字
# 张三
# 25
# {'city': '北京', 'title': '工程师'}

def my_function(name, age, *args):
    print(name)
    print(age)
    print(args)


my_function("张三", 25, "北京", "工程师")
# 张三
# 25
# ('北京', '工程师')
```



