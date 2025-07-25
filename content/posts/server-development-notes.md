---
title: "Server Development Notes"
date: 2025-07-24T16:16:21+08:00
# bookComments: false
# bookSearchExclude: false
---

# Insight

## 日志异常记录

日志异常记录遵循谁吞掉/转换/降级谁记录的原则，如果还需要向上抛异常，不需要记录该异常，不然会出现异常重复记录的情况

## 异常处理

根据业务自定义异常，分为两级，业务基类异常和业务具体异常，基类异常定义异常类型（error_type），具体业务异常定义错误含义(error_message)

```python
# ai-extractor项目
LLM_GENERATE_ERROR = "generate_error"

class LLMGenerateError(Exception):
    def __init__(self):
        self.message = "LLM生成错误"
        self.type = LLM_GENERATE_ERROR
        super().__init__(self.message)

class ExtractGenerateError(LLMGenerateError):
    def __init__(self):
        self.message = "抽取LLM生成错误"
        super().__init__(self.message)
        

class ValidateGenerateError(LLMGenerateError):
    def __init__(self):
        self.message = "类别验证LLM生成错误"
        super().__init__(self.message)

```

如在ai-extractor项目中，定义了业务基类异常LLMGenerateError，表示LLM生成方面的异常。ExtractGenerateError和ValidateGenerateError继承LLMGenerateError，为业务具体异常，分别表示抽取LLM和类别验证LLM生成异常，其type继承父类LLM_GENERATE_ERROR，message阐述具体业务异常。

还可通过以下方式传入上下文，result作为上下文传入：

```python
# ai-extractor项目
class LLMParseError(Exception):
    def __init__(self):
        self.type = LLM_PARSE_ERROR
        self.message = "LLM解析错误"
        super().__init__(self.message)

class ExtractParseError(LLMParseError):
    def __init__(self,result):
        self.result=result
        self.message = f"图像抽取结果解析错误: {result}"
        super().__init__(self.message)
```

这样自定义异常处理方式层级清晰，业务表意具体。

## 算法端与服务端解耦合

算法端与服务端尽量低耦合，算法端异常不应该抛给服务端进行处理，而应该在算法端自身处理并日志打印

如下，算法端在最高调用层级捕捉并打印异常，通过数据返回的形式返回给服务调用端异常类型和异常信息，服务端接收到该数据后可根据"error_type"是否为None判断业务处理是否出现异常，通过"error_msg"获取具体异常业务信息，正好与上一节“异常处理”相配合

```python
# ai-extractor项目
except ExtractGenerateError as e:
        logger.error(f"图像抽取回复生成错误:\n{str(e)}", exc_info=True)
        return {
            "result": None,
            "error_type": e.type,
            "error_msg": e.message,
            "validation_warning": None,
            "quality_warning": None
        }
```

## 算法端返回数据格式统一

算法端返回的数据格式最好统一，不存在的字段设置为None即可

```python
# ai-extractor项目
res = {
    "result": extract_result,
    "error_type": None,
    "error_msg": None,
    "validation_warning": validation_warning,
    "quality_warning": evaluate_warning
}
return res
```

