# 1. What is `torch-dag`?

`torch-dag` is a repository in which we implement a graph-like representation for `torch` models so that we have a
unified structure for every model. This structure is a directed acyclic graph (DAG), which we call a `DagModule`. We do
this so that:

1. We can easily run graph modification algorithms on the whole model without the need to edit any code defining the
   model. For example:

* we can easily switch all activation functions in the model with just a couple lines of code
* we can easily insert new vertices (modules) into the model based on some predefined pattern, e.g., add batch norm
  layers after every convolution provided there is none present immediately after the convolution
* we can introduce hierarchy into the model by wrapping some parts of the graph -> this is very useful if one wants to
  run some Neural Architecture Search algorithms

2. We can build a model once in the plain `pytorch` way, then convert it to a `DagModule`, save it and load it **without
   having to have the source code for model building**. This is extremely useful - imagine all the trouble one has to go
   through when one needs to modify the model source code manually to some simple stuff like changing some of the model
   layers.

The conversion between a regular `torch.nn.Module` instance and `DagModule` uses `torch.FX`. We decide not to
use `torch.FX` graph representation directly for the following reasons:

* there is no serialization/deserialization
  for `torch.FX` [GraphModule](https://pytorch.org/docs/stable/fx.html#torch.fx.GraphModule)
* the graph generated by `torch.FX` is often not a computational deep learning graph one would like to work with since
  it contains a huge number of purely `python` operations as nodes
* there is no notion of hierarchy/nesting in `GraphModule`
* there is often a number of ways to represent the same operation in `torch.FX` `GraphModule` (think of addition
  as `x+y`, `torch.add(x, y)`, which would have a very different representation in `GraphModule`), which leads to plenty
  of issues when one tries to implement some method to modify graph based on the presence of additions -> one has to
  think of all the different ways in which addition can be implemented in `torch`!

> **Note**: Not every `torch.nn.Module` model can be converted to a `DagModule`! We mostly share the limitations
> of `torch.FX` in this respect, although we try to overcome that by explicitly considering some `torch.FX`-unfriendly
> modules as atomic modules of our representation.

# 2. Why do we need a unified DAG-like representation?

The short answer is **to run algorithms** for stuff like model compression. In torch-dag we implement a channel pruning
algorithm that aspires to be architecture agnostic and we try really hard to make it work for as many architectures as
possible. The exhaustive list of the `timm` models we support is given [here](resources/supported_models_table.md).

# 3. Installation

## 3.1 Installation from cloned source code

In the cloned project's root (where this README.md file is) run

```bash
pip install -e .
```

## 3.2 Installation from PyPI

```bash
pip install torch-dag
```

# 4. Basics

If you have a a `toch.nn.Module` model and you want to convert it to a `DagModule` simply run:

```python
import torch_dag as td
dag_model = td.build_from_unstructured_module(model)
```

For details and more extended documentation see [How to convert torch.nn.Module instances to DagModule?](resources/conversion_to_dag_module.md)

# 5. Model compression algorithms

## 5.1 Channel-pruning

A `jupyter` notebook with atoy example of channel pruning can be viewed [here](./resources/examples/mnist_notebook.ipynb).
If you want to read a more detailed intro to channel pruning with `torch-dag` havce a look at [pruning readme](resources/pruning_readme.md).

This is the algorithm we spent plenty of time developing and refining. It helped us to **WIN** [Mobile AI 2022 Single-Image Depth Estimation on Mobile Devices](https://arxiv.org/abs/2211.04470).

### Supported Models and Limitations

---
#### Current model coverage `timm==0.9.5`
|                         |   num models | percentage   |
|:------------------------|-------------:|:-------------|
| all models              |          945 | 100.00%      |
| convertible models      |          690 | 73.02%       |
| channel prunable_models |          585 | 61.90%       |
---

At this point we support plenty of convolutional models and a subset of vision transformer architectures. 
A full list of supported `timm` models and a proportion of FLOPS that can be removed in each mdoel can be seen
[channel pruning supported models](./resources/channel_pruning_supported_models_table.md).

We **do NOT** support models that cannot be traced using `torch.FX` (there are notable exceptions, that require
additional processing like `vit*` and `deit3*` models, which **are** supported)
see [limitations-of-symbolic-tracing](https://pytorch.org/docs/stable/fx.html#limitations-of-symbolic-tracing). If you
have a custom model that can be traced:

```python=
my_module = Model()
from torch.fx import symbolic_trace
# Symbolic tracing frontend - captures the semantics of the module
symbolic_traced : torch.fx.GraphModule = symbolic_trace(my_module)
```

then chances are it will be convertible to a `DagModule` and some part of it can be pruned! How much precisely?

### Results in `ImageNet1k` for some `timm` models.

> NOTE: as a proxy for model size we often use `FLOPs` normalized by input resolution expressed in thousands
> (`kmapp` for short - thousands of `FLOPs` per pixel), i.e.:
> `kmapp(model) = FLOPs(model) / (H * W * 1000)`, where `(H, W)` is
> input resolution.
---

#### [fbnetv3_g.ra2_in1k](https://huggingface.co/timm/fbnetv3_g.ra2_in1k)

[FBNETV3](https://arxiv.org/abs/2006.02049) is a highly optimized architecture so further slimming down
of it is challenging. Nonetheless, the results are still pretty good. We prune `fbnetv3g` and then fine-tune the pruned
model for 200 epochs.

### Results on `ImageNet1K` validation set.

| model      | acc (224x224) | acc (240x240) | GFLOPs (224x224) | params (m) | FLOPs reduction | 
| ---------- |:-------------:|:-------------:|:----------------:|------------|-----------------|
| fbnetv3g   |    0.8061     |    0.8132     |       2.14       | 16.6       |                 |
| fbnetv3d   |    0.7856     |    0.7927     |       1.04       | 10.3       |                 |
| fbnetv3b   |    0.7812     |    0.7871     |      0.845       | 8.6        |                 |
| m16        |    0.7742     |    0.7793     |      0.799       | 7.8        | ↓ 62.5%         | 
| m22        |    0.7920     |    0.7955     |       1.2        | 10.5       | ↓ 43.9%         | 
| m28        |    0.79686    |    0.8016     |      1.396       | 11.6       | ↓ 34.8%         |
| m32        |     0.803     |    0.8025     |       1.61       | 13.1       | ↓ 24.8%         |

![f1](resources/pruning_results_plots/f1.png "Title")
---

#### [convnextv2_nano.fcmae_ft_in22k_in1k](https://huggingface.co/timm/convnextv2_nano.fcmae_ft_in22k_in1k)

[ConvNextV2](https://arxiv.org/abs/2301.00808) is a purely convolutional model that is inspired
by the design of Vision Transformers. We prune `convnextv2_nano.fcmae_ft_in22k_in1k` and then fine-tune the pruned
model for 200 epochs.

### Results on `ImageNet1K` validation set.
|  model   | acc (224x224) | GFLOPs | params (m) | FLOPs reduction |
|:--------:|:-------------:|:------:|:----------:|-----------------|
| baseline |    0.8197     |  4.91  |    15.6    |                 |
|   m0.5   |    0.8155     |  2.64  |    9.4     | ↓  46.2%        |
|   m0.3   |    0.7922     |  1.68  |    6.3     | ↓  65.8%        |
|   m0.2   |    0.7531     |  0.93  |    3.8     | ↓ 81.0%         |

![f5](resources/pruning_results_plots/f5.png "Title")
---

To see more much more results have a look [here](resources/pruning_results.md)

## TODO Block-pruning

## TODO Learnable Low Rank Compression (LLRC)
