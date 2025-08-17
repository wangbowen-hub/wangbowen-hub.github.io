---
title: "Kv Cache"
date: 2025-08-17T17:46:21+08:00
# bookComments: false
# bookSearchExclude: false
---

# KV cache

## Prerequisites 

### 单头注意力机制

```python
import numpy as np

def softmax(x, axis=-1):
    x = x - np.max(x, axis=axis, keepdims=True)  # 数值稳定
    ex = np.exp(x)
    return ex / np.sum(ex, axis=axis, keepdims=True)

def single_head_self_attention(X, Wq, Wk, Wv, causal=True):
    """
    X: 形状 (T, d_model) 的序列表示（例如 token embedding+pos）
    Wq, Wk, Wv: 形状分别为 (d_model, d_k), (d_model, d_k), (d_model, d_v)
    causal: True 表示解码器因果掩码（不看未来）

    返回:
    out: (T, d_v)  注意力输出
    attn: (T, T)   每个位置对所有位置的注意力权重
    """
    Q = X @ Wq                         # (T, d_k)
    K = X @ Wk                         # (T, d_k)
    V = X @ Wv                         # (T, d_v)

    dk = K.shape[-1]
    scores = (Q @ K.T) / np.sqrt(dk)   # (T, T)

    if causal:
        # 上三角(不含对角)设为 -inf，禁止看未来
        mask = np.triu(np.ones_like(scores), k=1)
        scores = scores + (mask * -1e9)

    attn = softmax(scores, axis=-1)    # (T, T)
    out = attn @ V                     # (T, d_v)
    return out, attn


# --- 示例 ---
if __name__ == "__main__":
    np.random.seed(0)
    T, d_model, d_k, d_v = 5, 8, 4, 4
    X  = np.random.randn(T, d_model).astype(np.float32)

    Wq = np.random.randn(d_model, d_k).astype(np.float32)
    Wk = np.random.randn(d_model, d_k).astype(np.float32)
    Wv = np.random.randn(d_model, d_v).astype(np.float32)

    out, attn = single_head_self_attention(X, Wq, Wk, Wv, causal=True)
    print("out shape:", out.shape)     # (5, 4)
    print("attn shape:", attn.shape)   # (5, 5)
    print("最后一步的注意力分布:", attn[-1])  # 对历史(含自己)的权重

```

### 多头注意力机制

```python
import numpy as np

def softmax(x, axis=-1):
    x = x - np.max(x, axis=axis, keepdims=True)  # 数值稳定
    ex = np.exp(x)
    return ex / np.sum(ex, axis=axis, keepdims=True)

def mha_self_attention(X, Wq, Wk, Wv, Wo, n_heads, causal=True):
    """
    X  : (T, d_model)  输入序列（embedding+位置编码 或 上一层输出）
    Wq : (d_model, n_heads * d_k)
    Wk : (d_model, n_heads * d_k)
    Wv : (d_model, n_heads * d_v)
    Wo : (n_heads * d_v, d_model)
    n_heads : 头数
    causal  : 是否使用因果掩码（禁看未来）

    返回:
    out  : (T, d_model)  多头注意力输出（拼接后再乘 Wo）
    attn : (n_heads, T, T) 各头的注意力权重矩阵
    """
    T, d_model = X.shape
    d_k = Wq.shape[-1] // n_heads
    d_v = Wv.shape[-1] // n_heads

    # 线性映射 & 变形到 (H, T, d_k/d_v)
    Q = X @ Wq   # (T, H*d_k)
    K = X @ Wk   # (T, H*d_k)
    V = X @ Wv   # (T, H*d_v)

    Q = Q.reshape(T, n_heads, d_k).transpose(1, 0, 2)  # (H, T, d_k)
    K = K.reshape(T, n_heads, d_k).transpose(1, 0, 2)  # (H, T, d_k)
    V = V.reshape(T, n_heads, d_v).transpose(1, 0, 2)  # (H, T, d_v)

    # 缩放点积注意力：scores (H, T, T)
    scores = np.matmul(Q, K.transpose(0, 2, 1)) / np.sqrt(d_k)

    if causal:
        # 上三角掩码（不含对角）-> -inf
        mask = np.triu(np.ones_like(scores), k=1)
        scores = scores + mask * -1e9

    attn = softmax(scores, axis=-1)        # (H, T, T)
    Z = np.matmul(attn, V)                 # (H, T, d_v)

    # 拼接各头 -> (T, H*d_v) -> 乘 Wo -> (T, d_model)
    Z = Z.transpose(1, 0, 2).reshape(T, n_heads * d_v)  # (T, H*d_v)
    out = Z @ Wo                                        # (T, d_model)

    return out, attn


# --- 最小可运行示例 ---
if __name__ == "__main__":
    np.random.seed(0)
    T, d_model, H = 5, 8, 2
    d_k = d_v = d_model // H  # 常见设置：每头维度相等

    X  = np.random.randn(T, d_model).astype(np.float32)
    Wq = np.random.randn(d_model, H * d_k).astype(np.float32)
    Wk = np.random.randn(d_model, H * d_k).astype(np.float32)
    Wv = np.random.randn(d_model, H * d_v).astype(np.float32)
    Wo = np.random.randn(H * d_v, d_model).astype(np.float32)

    out, attn = mha_self_attention(X, Wq, Wk, Wv, Wo, n_heads=H, causal=True)
    print("out shape:", out.shape)      # (5, 8)
    print("attn shape:", attn.shape)    # (2, 5, 5)
    # 验证每个头每一行权重和为 1
    print("头0最后一步注意力分布和 =", attn[0, -1].sum(), 
          "；头1最后一步注意力分布和 =", attn[1, -1].sum())

```

## KV Cache

https://zhuanlan.zhihu.com/p/662498827

https://drive.google.com/file/d/1-Wh776WzmDZYvMQ3-Seo9LBdL8qSM1z3/view?usp=sharing

* KV cache只能用于Decoder架构的模型，这是因为Decoder有Causal Mask，在推理的时候前面已经生成的字符不需要与后面的字符产生attention，从而使得前面已经计算的K和V可以缓存起来。

* KV cache会额外占用消耗内存。

