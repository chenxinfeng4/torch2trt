# torch2trt - A PyTorch -> TensorRT Converter

torch2trt is a PyTorch to TensorRT converter which utilizes the 
TensorRT Python API.  The converter is

* Easy to use - Convert models with a single function call ``torch2trt``
* Easy to extend - Write your own layer converter in Python and register it with ``@tensorrt_converter``

If you find an issue, please [let us know](../..//issues)!  We'd also love to hear if you create your own ``@tensorrt_converter``. It may be helpful to others.

### Setup

```bash
python setup.py install --user
```

### Usage

```python
from torch2trt import torch2trt
from torchvision.models.alexnet import alexnet

# create some regular pytorch model...
model = alexnet(pretrained=True).eval().cuda()

# create example data
x = torch.ones((1, 3, 224, 224)).cuda()

# convert to TensorRT feeding sample data as input
model_trt = torch2trt(model, [x])
```

We can then test the output of the regular and TensorRT optimized models

```
y = model(x)
y_trt = model_trt(x)

print(torch.max(torch.abs(y - y_trt)))
```

### Tested models

Below are models that we benchmarked on NVIDIA Jetson Nano.  Timing just includes model execution (not data copy).

| Model | PyTorch FP16 (Jetson Nano) | TensorRT FP16 (Jetson Nano) |
|-------|--------------|-----------------|
| alexnet | 18ms | 13ms |
| squeezenet1_0 | 21ms | 8.4ms |
| squeezenet1_1 | 16ms | 5.5ms |
| resnet18 | 32ms | 11ms |
| resnet34 | 58ms | 21ms |
| resnet50 | 77ms | 38ms |
| resnet101 | 135ms | 62ms |
| resnet152 | 200ms | 93ms |
| densenet121 | 83ms | 46ms |
| densenet169 | 116ms | 58ms |
| densenet201 | 139ms | 75ms |
| densenet161 | 209ms | 97ms |
| vgg11 | 61ms | 17ms |
| vgg13 | 96ms | 33ms |
| vgg16 | 137ms | 44ms |
| vgg19 |  |  |
| vgg11_bn |  |  |
| vgg13_bn |  |  |
| vgg16_bn |  |  |
| vgg19_bn |  |  |


### Add (or override) a converter

Here we show how to add an example converter using the TensorRT
python API.

```python
import tensorrt as trt
from torch2trt import tensorrt_converter

@tensorrt_converter('torch.nn.ReLU.forward')
def convert_ReLU(ctx):
    input_tensor = ctx.method_args[1]
    output_tensor = ctx.method_return
    trt_input = ctx.trt_tensors[input_tensor.__hash__()]
    layer = ctx.network.add_activation(input=trt_input, type=trt.ActivationType.RELU)  
    ctx.trt_tensors[output_tensor.__hash__()] = layer.get_output(0)
```

The converter takes one argument, a ``ConversionContext``, which will contain
the following

* network: The TensorRT network that is being constructed.
* method_args: Positional arguments that were passed to the specified Torch function.
* method_kwargs: Keyword arguments that were passed to the specified Torch function.
* method_return: The value returned by the specified Torch function.
* trt_tensors: A dictionary mapping Torch tensors (by hash value) to TensorRT tensors.  The
  converter must the set values for any output Tensors.  Otherwise, if a later function uses
  the Torch tensor, and there is not an associated TensorRT tensor in the map, results 
  may be unexpected.

Please see the ``torch2trt.py`` module for more examples.

### A comment on variable size tensors

TensorRT currently does not support variable size Tensors, so whatever input shape you use when converting, you must use
when executing.  While this may seem
limiting, it can actually be a good constraint when designing your model for use in embedded systems.  By 
restricting to a fixed input size, we can expect similar memory usage and runtime.  Ultimately, even if 
TensorRT didn't have this constraint, you'd probably want to have it anyways :)