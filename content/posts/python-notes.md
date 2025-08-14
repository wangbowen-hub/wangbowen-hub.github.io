---
title: "Python Notes"
date: 2025-07-31T14:31:37+08:00
# bookComments: false
# bookSearchExclude: false
---

# Python Notes

# Dict vs TypedDict vs dataclass

| 特性       | Dict                                                      | TypedDict                                                    | dataclass                                                    |
| ---------- | --------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 定义       | dict的泛型，让它可以写成 `Dict[K, V]`定义key和value的类型 | 相较于Dict会额外检查缺键、多键和键值类型错误，并且每个key对应的value类型可以不同 | 一个类装饰器，会根据类中带类型注解的字段自动生成 `__init__`、`__repr__`、`__eq__` 等常用方法，从而快速构建轻量级数据对象 |
| 创建方式   | 使用 `typing.Dict` 创建此字典                             | 使用 `typing.TypedDict` 创建此字典                           | 使用 `@dataclass` 装饰器定义类                               |
| 静态检查   | ✅                                                         | ✅                                                            | ✅                                                            |
| 运行时校验 | ❌                                                         | ❌                                                            | ❌                                                            |
| 访问方式   | data["key"]                                               | data["key"]                                                  | data.key                                                     |

Python作为一个动态语言，并不会做类型检查，所谓的“类型声明”只是一个“提示（hints）”或者“注解（annotations）”的作用。

## is None vs == None

| 对比点     | is                                                           | ==                                                           |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 用途       | is比较的是两个变量是否是同一个对象（为什么不说是比较的是内存地址，是因为python不像C，id(obj)返回的不是对象地址，而是其他的表示变量身份唯一性的值） | 调用的是`__eq__`方法进行比较，如果类中没实现就退回到父类或最终回退到`object.__eq__` |
| 是否可重写 | 不可重写                                                     | 可重写                                                       |

* 正常业务代码里始终使用is None！！！
* python解释器启动时只会创建一个None对象，任何地方使用的None都指向同一块内存

```python
x = None
y = None
print(x is None)   # True
print(x == None)   # True

# 自定义对象
class Demo:
    def __init__(self, val):
        self.val = val
    def __eq__(self, other):
        return self.val == other

d = Demo(None)
print(d == None)   # True（走 __eq__）
print(d is None)   # False（身份不同）

```

