---
title: "Python Exception"
date: 2025-07-22T13:45:11+08:00
# bookComments: false
# bookSearchExclude: false
---

# Python异常处理原则

python推荐的是EAFP（Easier to Ask for Forgiveness than Permission）原则，原则的含义是："请求宽恕比请求许可更容易"。

这是Python推崇的一种编程哲学，核心思想是：直接尝试执行操作，如果失败了再处理异常，而不是事先检查所有可能的条件。

```python
import os
# LBYL（Look Before You Leap）方式
# 先检查，再操作
if os.path.exists('data.txt'):
    if os.access('data.txt', os.R_OK):
        with open('data.txt', 'r') as f:
            content = f.read()
            print("文件内容:", content)
    else:
        print("文件不可读")
else:
    print("文件不存在")


# EAFP方式（推荐）    
# 直接尝试，失败了再处理
try:
    with open('data.txt', 'r') as f:
        content = f.read()
        print("文件内容:", content)
except FileNotFoundError:
    print("文件不存在")
except PermissionError:
    print("文件不可读")
except Exception as e:
    print(f"其他错误: {e}")
```

## 1. 捕获的异常越具体越好，避免捕获裸异常，必要时用 `except Exception:` 兜底

裸exception捕获的是`BaseException`，会把 `BaseException` 的所有子类都吞掉，连 `KeyboardInterrupt`、`SystemExit` 等终止信号也拦下，导致程序无法优雅退出。因此，请始终指定异常类型，若不确定则直接使用 `Exception`

```python
try:
    time.sleep(10)   
except:             
    print("oops")
# ^Coops
```

```python
try:
    time.sleep(10)   
except Exception:             
    print("oops")
#######################
^CTraceback (most recent call last):
  File "/Users/bowenwang/Documents/code/AI-Extractor-ALL/ai-extractor/test.py", line 3, in <module>
    time.sleep(10)   
KeyboardInterrupt
```

## 2. 捕获异常后不要静默吞掉

捕获异常后，选择以下选项进行处理，不要不做任何处理，静默吞掉：

* 日志记录

* raise抛出原异常

* 转换成业务异常抛出

* 使用from保留异常链上下文抛出

  | 写法                             | 行为                                                         | 会显示原异常吗                                               | 用途                           |
  | -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------ |
  | `raise`（裸）                    | 在 except 中原样重新抛出当前捕获的异常                       | 会（就是同一个异常）                                         | 只是想继续往外抛，不改类型     |
  | `raise NewErr(...)`（不带 from） | 抛新异常，原异常做为 `__context__`                           | 会，输出是 “During handling of the above exception, another exception occurred...” | 语义不够明确                   |
  | `raise NewErr(...) from e`       | 抛新异常，原异常做为 `__cause__`，当设置 `__cause__` 属性时，异常上还会标记 `__suppress_context__ = True` 标志；当 `__suppress_context__` 被设为 `True` 时，打印回溯信息时将忽略 `__context__` 属性。 | 会，输出是 “The above exception was the direct cause of the following exception...” | 推荐；明确链路                 |
  | `raise NewErr(...) from None`    | 抛新异常，**隐藏**原异常链                                   | 不会                                                         | 刻意不想暴露底层细节（谨慎用） |

**raise VS raise e：**

| 点                         | `raise`                                      | `raise e`                                        |
| -------------------------- | -------------------------------------------- | ------------------------------------------------ |
| 位置要求                   | 只能在 `except` 块内部且当前有正在处理的异常 | 在任何地方都行，只要有异常对象 `e`               |
| Traceback 是否保留原始堆栈 | **是**（继续沿用原异常的 traceback）         | **否**，默认从这一行重新计，之前的调用栈会被截断 |
| 常见用法                   | 想把异常“层层抛出”，且不改动类型或信息       | 想在别处再次抛出已保存的异常                     |

## 3. 使用`finally`保证资源释放

* 使用finally保证资源始终被正确释放

* 在 `finally` 中别 `return`/`raise` 覆盖原异常，它会吞掉真实错误。

```python
try:
    file = open('data.txt')
    # 处理文件
except IOError as e:
    print(f"文件操作失败: {e}")
finally:
    if 'file' in locals() and not file.closed:
        file.close()
```

更加推荐的是使用with（上下文管理器）

```python
with open('data.txt') as file:
```

## 4. 必要时使用自定义异常表意

当业务需要暴露更语义化的错误（如 `TokenExpiredError`），或需要在异常对象中附带额外上下文（如订单 ID、用户 ID）时，自定义异常比内建异常更清晰 

```python
class ValidationError(Exception):
    """
    验证基础异常类，所有验证相关异常的父类。
    """
    pass

class EmailFormatError(ValidationError):
    """
    邮箱格式错误异常。
    """
    def __init__(self, email):
        self.email = email
        super().__init__(f"邮箱格式无效: {email}")

class PasswordTooWeakError(ValidationError):
    """
    密码过于简单异常。
    """
    def __init__(self, requirements="密码至少8位且包含字母和数字"):
        super().__init__(f"密码强度不足: {requirements}")

# ===============================
# 业务逻辑
# ===============================

def validate_user_registration(email, password):
    """
    验证用户注册信息。
    """
    # 邮箱格式验证
    if "@" not in email or "." not in email.split("@")[-1]:
        raise EmailFormatError(email)
    
    # 密码强度验证
    if len(password) < 8 or password.isdigit() or password.isalpha():
        raise PasswordTooWeakError()
```





