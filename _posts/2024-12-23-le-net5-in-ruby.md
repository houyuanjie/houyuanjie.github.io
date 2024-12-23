---
title: "使用 Ruby 实现 LeNet-5 卷积神经网络"
categories:
  - Machine Learning
  - Ruby
tags:
  - Convolutional Neural Network
---

如何在 Ruby 中实现一个 LeNet-5 卷积神经网络 (CNN) 结构，使用 MNIST 数据集在 PyTorch 上进行训练，最后将训练好的模型参数导入到我们自己的实现中？

完整代码请查看我的 Github [houyuanjie/lenet5-in-ruby](https://github.com/houyuanjie/lenet5-in-ruby)

# 1. LeNet-5

适用于 MNIST 数据集的 LeNet-5 是一种简单的 CNN 模型，它的结构示意如下：

```
网络结构                     # 数据维度
input: Array[Matrix]        # (1, 28, 28)
  >> conv1                  # (6, 24, 24)
  >> relu                   # (6, 24, 24)
  >> pool1                  # (6, 12, 12)

  >> conv2                  # (16, 8, 8)
  >> relu                   # (16, 8, 8)
  >> pool2                  # (16, 4, 4)

  >> flatten                # (256)

  >> fc1                    # (120)
  >> relu                   # (120)

  >> fc2                    # (84)
  >> relu                   # (84)

  >> fc3                    # (10)
output: Array               # (10)
```

- 在 `conv1` 和 `conv2` 中，使用的卷积核大小为 5x5
- 在 `pool1` 和 `pool2` 中，使用的池化核大小为 2x2

# 2. PyTorch 实现

```python
import torch.nn as nn
import torch.nn.functional as F


class LeNet5(nn.Module):
    def __init__(self):
        super(LeNet5, self).__init__()

        self.conv1 = nn.Conv2d(1, 6, 5)
        self.conv2 = nn.Conv2d(6, 16, 5)
        self.flatten = nn.Flatten()
        self.fc1 = nn.Linear(16 * 4 * 4, 120)
        self.fc2 = nn.Linear(120, 84)
        self.fc3 = nn.Linear(84, 10)

    def forward(self, input):
        last = self.conv1(input)
        last = F.relu(last)
        last = F.max_pool2d(last, kernel_size=2)

        last = self.conv2(last)
        last = F.relu(last)
        last = F.max_pool2d(last, kernel_size=2)

        last = self.flatten(last)

        last = self.fc1(last)
        last = F.relu(last)

        last = self.fc2(last)
        last = F.relu(last)

        return self.fc3(last)
```

# 3. Ruby 顶层实现

类比 PyTorch 代码，我们先在 Ruby 中设计出顶层实现 LeNet5 类

```ruby
# 代码有删改，完整代码请访问 Github

require 'matrix'

module Nn
  class LeNet5
    def initialize
      @conv1 = Conv2d.new(in_channels: 1, out_channels: 6, kernel_size: 5)
      @pool1 = MaxPool2d.new(kernel_size: 2)

      @conv2 = Conv2d.new(in_channels: 6, out_channels: 16, kernel_size: 5)
      @pool2 = MaxPool2d.new(kernel_size: 2)

      @fc1 = Linear.new(in_features: 16 * 4 * 4, out_features: 120)
      @fc2 = Linear.new(in_features: 120, out_features: 84)
      @fc3 = Linear.new(in_features: 84, out_features: 10)
    end

    def forward(input)
      input = [input] if input.is_a?(Matrix) # 1 * 28 * 28

      last = @conv1.forward(input) # 6 * 24 * 24
      last = ReLU.forward(last) # 6 * 24 * 24
      last = @pool1.forward(last) # 6 * 12 * 12

      last = @conv2.forward(last) # 16 * 8 * 8
      last = ReLU.forward(last) # 16 * 8 * 8
      last = @pool2.forward(last) # 16 * 4 * 4

      last = last.map(&:flatten).flatten # 256

      last = @fc1.forward(last) # 120
      last = ReLU.forward(last) # 120

      last = @fc2.forward(last) # 84
      last = ReLU.forward(last) # 84

      @fc3.forward(last) # 10
    end
  end
end
```

# 4. Ruby 底层实现

我们在顶层的 LeNet5 类中使用了我们需要的底层，现在我们需要分别实现这些底层的代码，下面我就以 `ReLU`, `Conv2d` 以及 `Linear` 为例，看看它们如何实现：

## 4.1 ReLU

`ReLU` 是实现起来最容易的一个层，简单来说，它就是 `x => if x.positive? then x else 0`，我们将它分为用来处理数组的 `ReLU` 类和用来处理矩阵的 `ReLU2d` 类

```ruby
# 代码有删改，完整代码请访问 Github

# re_lu.rb
module Nn
  class ReLU
    def self.forward(input) # input is an Array of Numeric
      input.map { |e| e.positive? ? e : 0 }
    end
  end
end

# re_lu2d.rb
require 'matrix'

module Nn
  class ReLU2d
    def self.forward(input) # input is an Array of Matrix
      input.map { |matrix| matrix.map { |e| e.positive? ? e : 0 } }
    end
  end
end
```

## 4.2 MaxPool2d

`MaxPool2d` 则接受参数，它需要 2 个参数，分别是 `kernel_size` 和 `stride`。

- `kernel_size` 表示池化核的大小
- `stride` 表示移动池化核的步长

输出数据的尺寸由这两个参数决定

```ruby
output_height = (input_height - kernel_size) / stride + 1
output_width = (input_width - kernel_size) / stride + 1
```

池化时移动池化核到输入矩阵的对应位置，取这个池化核套住的元素中的最大值作为输出

```ruby
# 代码有删改，完整代码请访问 Github

require 'matrix'

module Nn
  class MaxPool2d
    def initialize(kernel_size:, stride:, channels:, height:, width:)
      @kernel_size = kernel_size
      @stride = stride
      @channels = channels

      @output_height = (height - kernel_size) / stride + 1
      @output_width = (width - kernel_size) / stride + 1
    end

    def forward(input)
      Array.new(@channels) do |chn|
        Matrix.build(@output_height, @output_width) do |row, col|
          input[chn].slice(
            row_start: row * @stride, row_length: @kernel_size,
            col_start: col * @stride, col_length: @kernel_size
          ).max
        end
      end
    end
  end
end
```

## 4.3 Linear

而对于像 `Linear` `Conv2d` 这样的层，它们不仅接受像 `in_features`, `out_features` 这样的规格参数，还有能控制网络的权重 `weight` 和偏置 `bias` 参数。它们的具体实现请查看 Github 代码仓库

```ruby
# 代码有删改，完整代码请访问 Github

module Nn
  class Linear
    attr_reader :in_features, :out_features
    attr_accessor :weight, :bias

    def initialize(in_features:, out_features:)
      @in_features = in_features
      @out_features = out_features

      @weight = ...
      @bias = ...
    end

    def forward(input)
      ...
    end
  end
end
```

当我们训练模型时，我们调整的就是模型某些层中的权重 `weight` 和偏置 `bias`

# 5. 模型导出

在 PyTorch 训练好之后，我们将模型参数导出成 Ruby 脚本

我们需要导出的参数有：

```python
# 代码有删改，完整代码请访问 Github

model = LeNet5()

# 训练之后
conv1_weights = model.conv1.weight.data.cpu().numpy()
conv1_bias = model.conv1.bias.data.cpu().numpy()
conv2_weights = model.conv2.weight.data.cpu().numpy()
conv2_bias = model.conv2.bias.data.cpu().numpy()
fc1_weight = model.fc1.weight.data.cpu().numpy()
fc1_bias = model.fc1.bias.data.cpu().numpy()
fc2_weight = model.fc2.weight.data.cpu().numpy()
fc2_bias = model.fc2.bias.data.cpu().numpy()
fc3_weight = model.fc3.weight.data.cpu().numpy()
fc3_bias = model.fc3.bias.data.cpu().numpy()
```

需要将这些数据格式化成 Ruby 字面量，它们是多个维度的，使用 for 循环一层一层读

```python
# 代码有删改，完整代码请访问 Github

def format_array_as_ruby(array):
    length = len(array)
    ruby_str = "["
    for i in range(length):
        value = array[i]
        ruby_str += str(float(value)) + ","
    ruby_str += "]"
    return ruby_str


def format_matrix_as_ruby(matrix):
    matrix_height, matrix_width = matrix.shape
    ruby_str = "Matrix["
    for row in range(matrix_height):
        ruby_str += "["
        for col in range(matrix_width):
            value = matrix[row, col]
            ruby_str += str(float(value)) + ","
        ruby_str += "],"
    ruby_str += "]"
    return ruby_str


def format_conv_weights_as_ruby(conv_weights):
    out_channels, in_channels, kernel_height, kernel_width = conv_weights.shape
    ruby_str = "["
    for out_chn in range(out_channels):
        ruby_str += "["
        for in_chn in range(in_channels):
            ruby_str += "Matrix["
            for row in range(kernel_height):
                ruby_str += "["
                for col in range(kernel_width):
                    value = conv_weights[out_chn, in_chn, row, col]
                    ruby_str += str(float(value)) + ","
                ruby_str += "],"
            ruby_str += "],"
        ruby_str += "],"
    ruby_str += "]"
    return ruby_str
```

最后写入 `model/setup.rb` 脚本中

```python
# 代码有删改，完整代码请访问 Github

setup_rb_path = "model/setup.rb"


def make_setup_rb(file):
    file.write("require 'matrix'\n")
    file.write("\n")
    file.write("module Nn\n")
    file.write("  class LeNet5\n")
    file.write("    def self.setup(instance)\n")
    file.write("      instance.conv1.weights = {}.freeze\n".format(conv1_weights_ruby))
    file.write("      instance.conv1.bias = {}.freeze\n".format(conv1_bias_ruby))
    file.write("      instance.conv2.weights = {}.freeze\n".format(conv2_weights_ruby))
    file.write("      instance.conv2.bias = {}.freeze\n".format(conv2_bias_ruby))
    file.write("      instance.fc1.weight = {}.freeze\n".format(fc1_weight_ruby))
    file.write("      instance.fc1.bias = {}.freeze\n".format(fc1_bias_ruby))
    file.write("      instance.fc2.weight = {}.freeze\n".format(fc2_weight_ruby))
    file.write("      instance.fc2.bias = {}.freeze\n".format(fc2_bias_ruby))
    file.write("      instance.fc3.weight = {}.freeze\n".format(fc3_weight_ruby))
    file.write("      instance.fc3.bias = {}.freeze\n".format(fc3_bias_ruby))
    file.write("    end\n")
    file.write("  end\n")
    file.write("end\n")


with open(setup_rb_path, "w") as file:
    file.write("# This file is generated by train.py.\n")
    file.write("# Do not edit manually.\n")
    make_setup_rb(file)
```

# 导入 Ruby

生成 `model/setup.rb` 文件之后，我们就可以在 Ruby 中导入并开始使用了

```ruby
# 代码有删改，完整代码请访问 Github

require_relative 'model/setup'

model = Nn::LeNet5.new
Nn::LeNet5.setup(model)

output = model.forward(image)
predicted_label = output.index(output.max)
```
