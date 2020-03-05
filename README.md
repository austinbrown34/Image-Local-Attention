# Image Local Attention: a Better PyTorch Implementation

## Introduction

Attention is widely used in deep learning now. Given a query, a collection of keys and values. The output of an attention module is the weighted sum of all values. The weights are obtained based on the similarities between the query and keys which are usually measured by their inner products. However, when number of keys is large, it is cost to apply such a module.

Researchers consider about local attention to address this problem. That is a small subset of keys are involved given a query. For images, "local" means a image region around a pixel. Image local attention achieves great success on image restoration tasks. However, current implementations are based on the `im2col` operation which is memory expensive especially when the local patch is large.

## Implementation

Here, queries **Q**, keys **K** and value **V** are represented in `CHW` (channel, height, width) tensors. They are generated by convolutions. And "local region" is a `CKK` sub tensor where `K` is the size of a patch. Current implementations are based on following steps:

* rearange **K** and **V** to `KKCHW` tensors via `im2col`.
* compute similarity matrix **W** between **Q** and **K**: `KKHW`.
* output **O** is sum of **V** weighted by **W**: `CHW`.

Clearly, the first and second steps require `KK` times memory to store **K** and **V**. However, this can be avoided. In our implementation, we compute **W** without extending keys and values. To this end, we write two CUDA kernels to compute **W** and **O**. And we build a PyTorch extension based on them. *Our implementation reduces the GPU memory by an order of magnitude and it is faster than the plain PyTorch implementation*.


## Install and usage

```bash
python setup.py install
```

Requirements:

```
PyTorch >= 1.4.0
CUDA >= 10.0
```

We write the Python warper in `function.py`.  Here is an example:

```python
import torch
from function import LocalAttention

# kH and kW for local patch size
module = LocalAttention(inp_channels=3, out_channels=16, kH=7, kW=7)

x = torch.rand(32, 3, 64, 64)
# Q, K, V are generated by convolutions of x
y = module(x)
```


## Performance

We evaluate the memory and speed of our implementation compared with the plain PyTorch implementation. The first table show performance for forward pass. The second table show over-all performance for forward-backward loop.

|  K   | Relative GPU Memory | Relative Time |
| :--: | :-----------------: | :-----------: |
|  5   |        10.2%        |     51.6%     |
|  11  |        3.2%         |     51.6%     |
|  21  |        2.0%         |     51.6%     |

|  K   | Relative GPU Memory | Relative Time |
| :--: | :-----------------: | :-----------: |
|  5   |        9.0%         |     71.1%     |
|  11  |        3.4%         |     65.1%     |
|  21  |        2.3%         |     70.7%     |

Refer `/test` for more results.
