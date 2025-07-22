---
title: "Image Token Calculate"
date: 2025-07-17T18:35:49+08:00
# bookComments: false
# bookSearchExclude: false
---

# OpenAI 图像 token 输入计费方法

| 计费方法  | 介绍                                                         | 适用范围                            |
| --------- | ------------------------------------------------------------ | ----------------------------------- |
| patch方案 | token计数为拿32*32大小的patch覆盖图像所需的patch数×scaling_factor<br />detail参数被忽略，不会影响token计数和模型输出质量 | GPT-4.1-mini, GPT-4.1-nano, o4-mini |
| tile方案  | detail = "low"：固定基数 tokens。<br/>detail = "high"：基数 + 512 px 分块(tile) 数 × 每 tile tokens。 | 除patch适用范围外的所有模型         |

* image token费用和文本token费用相同
* image token同样会占用模型上下文窗口
* 模型不会处理原始文件名或元数据，且可能会在分析前会对图像进行尺寸调整，改变其原始尺寸。

## patch方案

| 模型         | scaling_factor |
| ------------ | -------------- |
| gpt-4.1-mini | 1.62           |
| gpt-4.1-nano | 2.46           |
| o4-mini      | 1.72           |

* patch方案的token计数为拿32*32大小的patch覆盖图像所需的patch数

```text
image_patches = ceil(width/32)×ceil(height/32)
```

* 若计算出的patch数量<=1536

```python
image_tokens = image_patches*scaling_factor
```

* 若计算出的patch数量超过1536，则需要对原图像进行缩放

```python
# 先把总 patch 数压到 ≤ 1536
r = √(32²×1536/(width×height))
# 再微调 r，让缩完后的宽/高都正好落在 patch 网格上
r = r × min( floor(width×r/32) / (width×r/32), floor(height×r/32) / (height×r/32) )
```

r为缩放系数，所需的token数量为：

```python
resized_width=width*r
resized_height=height*r
image_tokens = ceil(resized_width/32)×ceil(resized_height/32)×scaling_factor
```

### 示例

一张2048*4096分辨率的图像，使用的是o4-mini模型：

* ceil(2048/32)*ceil(2048/32)=4,096 > 1536

* 计算缩放系数
  * r = √(32²×1536/(2048×4096))=0.43
  * r = 0.43 × min( floor(2048×0.43/32) / (2048×0.43/32), floor(4096×0.43/32) / (4096×0.43/32) )=0.42
* 最终所需token：
  * resized_width=2048*0.42=861
  * resized_height=4096*0.42=1721
  * image_tokens = ceil(861/32)×ceil(1721/32)×1.72=27×54×1.72=2508
  * 该图像消耗2508个token



## tile方案

![image-20250718180004396](/Users/bowenwang/Library/Application Support/typora-user-images/image-20250718180004396.png)

* 如果detail = "low"，则消耗固定基数token，图像被缩放成512*512分辨率输入

* 如果detail = "high":

  * 判断图像最长边是否<=2048px，如果是，图片大小不变，如果否，图像等比例缩放到最长边为2048px

  * 将图像最短边等比例缩放至768px

  * 计算缩放后的图像需要多少个512*512px个tile覆盖

  * ```python
    image_tokens = tile_num*tile_tokens+base_tokens
    ```

### 示例

一张2048*4096分辨率的图像，使用的是o3模型：

* detail = "low"，消耗75个token
* detail = "high":
  * 图像最长边为4096，>2048，将图像缩放至1024*2048
  * 图像最短边为1024，需缩放至768，将图像缩放至768*1536
  * 缩放后的图像需要ceil(768/512)*ceil(1536/512)=6个tile
  * 该图像需消耗6*150+75=975个token



