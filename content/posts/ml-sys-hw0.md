+++
title = 'CMU 10-414/714: Deep Learning Systems (2020) - 深度学习系统 hw0'
date = 2025-03-04T16:46:50-05:00
draft = false
tags = ['Learning Notes', 'Deep Learning']
+++

## 10-714: Homework 0

build a basic softmax regression algorithm, plus a simple two-layer neural network

> [github hw0](https://github.com/dlsyscourse/hw0.git)

### Question 1: A basic `add` function, and testing/autograding basics

plement `simple_ml.add()` function in `src/simple_ml.py`

testing: `!python3 -m pytest -k "add"`

> 根据测试，由于 add(x, y) 传入的参数可以是任意类型，则直接返回 return x + y
>
> 没有注册 mugrade 等 autograde system，所以就基本跑本地的测试了

### Question 2: Loading MNIST data

`parse_mnist_data()` function

```python
def parse_mnist(image_filename, label_filename):
    """ Read an images and labels file in MNIST format.  See this page:
    http://yann.lecun.com/exdb/mnist/ for a description of the file format.

    Args:
        image_filename (str): name of gzipped images file in MNIST format
        label_filename (str): name of gzipped labels file in MNIST format

    Returns:
        Tuple (X,y):
            X (numpy.ndarray[np.float32]): 2D numpy array containing the loaded
                data.  The dimensionality of the data should be
                (num_examples x input_dim) where 'input_dim' is the full
                dimension of the data, e.g., since MNIST images are 28x28, it
                will be 784.  Values should be of type np.float32, and the data
                should be normalized to have a minimum value of 0.0 and a
                maximum value of 1.0 (i.e., scale original values of 0 to 0.0
                and 255 to 1.0).

            y (numpy.ndarray[dtype=np.uint8]): 1D numpy array containing the
                labels of the examples.  Values should be of type np.uint8 and
                for MNIST will contain the values 0-9.
    """
```

> 注意返回的是 Tuple
>
> 使用 gzip 打开文件 `gzip.open(image_filename, 'rb')` 和 `gzip.open(label_filename, 'rb')`

![](https://s2.loli.net/2025/03/08/MxOdNoDwmC4Xl7s.png)

MNIST `image_filename` 的格式为 `[magic number 32bits] [number of images 32bits] [number of rows 32bits] [number of columns 32bits] [... pixel]`

MNIST `label_filename` 的格式为 `[magic number 32bits] [number of items 32bits] [...label]`

1. 利用 `struct.unpack('>IIII', f.read(16))` 读取 `image_file` 表示 `big-endian` 大端读 `16 bytes` 获取 `magic, num_images, rows, cols`

   1. 然后使用 `img_data = np.frombuffer(f.read(), dtype=np.uint8)` 获取剩下的图像数据（一维），进行 `img_data.reshape(num_images, rows * cols) / 255.0` 转成二维

2. 利用 `struct.unpack('>II', f.read(8))` 读取 `image_file` 表示 `big-endian` 大端读 `8 bytes` 获取 `magic, num_items`

   1. 然后用 `y = np.frombuffer(f.read(), dtype=np.uint8)` 获取剩下的标签数据

### Question 3: Softmax loss

Implement the softmax (a.k.a. cross-entropy) loss as defined in `softmax_loss()` function in `src/simple_ml.py`.

$$
\begin{equation}
\ell_{\mathrm{softmax}}(z, y) = \log\sum_{i=1}^k \exp z_i - z_y.
\end{equation}
$$

`softmax_loss()` takes a _2D array_ of logits (i.e., the $k$ dimensional logits for a batch of different samples), plus a corresponding 1D array of true labels, and should output the _average_ softmax loss over the entire batch. Note that to do this correctly, you should _not_ use any loops, but do all the computation natively with numpy vectorized operations (to set expectations here, we should note for instance that our reference solution consists of a single line of code).

```python
def softmax_loss(Z, y):
    """ Return softmax loss.  Note that for the purposes of this assignment,
    you don't need to worry about "nicely" scaling the numerical properties
    of the log-sum-exp computation, but can just compute this directly.

    Args:
        Z (np.ndarray[np.float32]): 2D numpy array of shape
            (batch_size, num_classes), containing the logit predictions for
            each class.
        y (np.ndarray[np.uint8]): 1D numpy array of shape (batch_size, )
            containing the true label of each example.

    Returns:
        Average softmax loss over the sample.
    """
```

按照公式完成即可，注意输入是 `Z (np.ndarray[np.float32]) (batch_size, num_classes)` 每一行代表一个样本的 logits（预测分数），`y (np.ndarray[np.uint8])` 是对应的 label

拿出 `batch_size = Z.shape[0]`，根据公式写出 `np.log(np.sum(np.exp(Z), axis=1)) - Z[np.arange(batch_size), y]` 最后返回 `np.mean(loss)`

> `np.arange(batch_size)` 生成一个从 0 到 batch_size-1 的数组，这样可以和 y 配合从 Z 中提取每个样本真实类别对应的 logit。

### Question 4: Stochastic gradient descent for softmax regression

implement stochastic gradient descent (SGD) for (linear) softmax regression.

consider a hypothesis function that makes $n$-dimensional inputs to $k$-dimensional logits via the function

$$
\begin{equation}
h(x) = \Theta^T x
\end{equation}
$$

where $x \in \mathbb{R}^n$ is the input, and $\Theta \in \mathbb{R}^{n \times k}$ are the model parameters

the gradient of the linear softmax objective is given by

$$
\begin{equation}
\nabla_\Theta \ell_{\mathrm{softmax}}(\Theta^T x, y) = x (z - e_y)^T
\end{equation} \\

\begin{equation}
z = \frac{\exp(\Theta^T x)}{1^T \exp(\Theta^T x)} \equiv normalize(\exp(\Theta^T x))
\end{equation} \\
$$

Using these gradients, implement the `softmax_regression_epoch()` function, which runs a single epoch of SGD (one pass over a data set) using the specified learning rate / step size `lr` and minibatch size `batch`. As described in the docstring, your function should modify the `Theta` array in-place.

```python
def softmax_regression_epoch(X, y, theta, lr = 0.1, batch=100):
    """ Run a single epoch of SGD for softmax regression on the data, using
    the step size lr and specified batch size.  This function should modify the
    theta matrix in place, and you should iterate through batches in X _without_
    randomizing the order.

    Args:
        X (np.ndarray[np.float32]): 2D input array of size
            (num_examples x input_dim).
        y (np.ndarray[np.uint8]): 1D class label array of size (num_examples,)
        theta (np.ndarrray[np.float32]): 2D array of softmax regression
            parameters, of shape (input_dim, num_classes)
        lr (float): step size (learning rate) for SGD
        batch (int): size of SGD minibatch

    Returns:
        None
    """
```

按照 `batch` 大小遍历数据集，其中 `X.shape[0]` 是 `num_examples`

每次获得 `X[i : i+batch]` 输入和 `y[i : i+batch]` 相应的 label

`theta` 是参数矩阵，在 `numpy` 里用 `@` 进行矩阵乘法 `X_batch @ theta` 计算 minibatch logits 也就是预测分数

根据公式计算 `np.exp(logits)` 求和并且进行 `normalize` 归一化

One-hot 编码：归一化结果 `one_hot = [np.arange(X_batch.shape[0]), y_batch] = 1.0` 就得到了标签的的编码。

> 这里是 python 的语法，`np.arange(X_batch.shape[0])` 创建一个大小为参数的数组，并且从 0 开始，也就是行索引，而 `y_batch` 则是标签的索引，列索引。

梯度计算：`grad = (X_batch.T @ (probs - one_hot)) / X_batch.shape[0]`

> 梯度公式 $\frac{1}{m} X^T (Z - I_y)$

modify the `Theta` array in-place: `theta -= lr * grad` 其中 lr 就是学习率

> $\theta_{\text{new}} = \theta - \eta \cdot \text{grad}$

Training MNIST with softmax regression:

For reference, as seen below, our implementation runs in ~3 seconds on Colab, and achieves 7.97% error.

### Question 5: SGD for a two-layer neural network

```python

def nn_epoch(X, y, W1, W2, lr = 0.1, batch=100):
    """ Run a single epoch of SGD for a two-layer neural network defined by the
    weights W1 and W2 (with no bias terms):
        logits = ReLU(X * W1) * W2
    The function should use the step size lr, and the specified batch size (and
    again, without randomizing the order of X).  It should modify the
    W1 and W2 matrices in place.

    Args:
        X (np.ndarray[np.float32]): 2D input array of size
            (num_examples x input_dim).
        y (np.ndarray[np.uint8]): 1D class label array of size (num_examples,)
        W1 (np.ndarray[np.float32]): 2D array of first layer weights, of shape
            (input_dim, hidden_dim)
        W2 (np.ndarray[np.float32]): 2D array of second layer weights, of shape
            (hidden_dim, num_classes)
        lr (float): step size (learning rate) for SGD
        batch (int): size of SGD minibatch

    Returns:
        None
    """
```

consider a two-layer neural network (without bias terms) of the form

$$ z = W_2^T \mathrm{ReLU}(W_1^T x) $$

where $W_1 \in \mathbb{R}^{n \times d}$ and $W_2 \in \mathbb{R}^{d \times k}$ represent the weights of the network (which has a $d$-dimensional hidden unit), and where $z \in \mathbb{R}^k$ represents the logits output by the network.

again use the softmax / cross-entropy loss, meaning that we want to solve the optimization problem

$$
minimize_{W_1, W_2} \;\; \frac{1}{m} sum_{i=1}^m \ell_{\mathrm{softmax}}(W_2^T \mathrm{ReLU}(W_1^T x^{(i)}), y^{(i)})
$$

Using the chain rule, we can derive the **backpropagation** updates for this network

$$
\begin{equation}
\begin{split}
Z_1 \in \mathbb{R}^{m \times d} & = \mathrm{ReLU}(X W_1) \\
G_2 \in \mathbb{R}^{m \times k} & = normalize(\exp(Z_1 W_2)) - I_y \\
G_1 \in \mathbb{R}^{m \times d} & = \mathrm{1}\{Z_1 > 0\} \circ (G_2 W_2^T)
\end{split}
\end{equation}
$$

> 相比 SGD，2 layer 的实现也很类似，但需要实现 forward 和 backprop，以及链式法则

$$ \text{softmax}(z*i) = \frac{\exp(z_i)}{\sum*{j=1}^{k} \exp(z_j)} $$

ReLU 的误差计算：

$$
\nabla_{W_1} \ell_{\mathrm{softmax}}(\mathrm{ReLU}(X W_1) W_2, y) = \frac{1}{m} X^T G_1  \\
G_1 \in \mathbb{R}^{m \times d} = \mathrm{1}\{Z_1 > 0\} \circ (G_2 W_2^T)
$$

```python
def nn_epoch(X, y, W1, W2, lr=0.1, batch=100):
    m = X.shape[0]
    for i in range(0, m, batch):
        X_batch = X[i : i + batch]
        y_batch = y[i : i + batch]
        m_batch = X_batch.shape[0]

        # forward, ReLU(XW1)W2, np.maximum(0, ...) 是 ReLU 的实现形式
        Z1 = np.maximum(0, X_batch @ W1)
        logits = Z1 @ W2

        # 计算 softmax 输出
        exp_logits = np.exp(logits)
        sums = np.sum(exp_logits, axis=1, keepdims=True)
        probs = exp_logits / sums

        # 编码
        one_hot = np.zeros_like(probs)
        one_hot[np.arange(m_batch), y_batch] = 1.0

        # backprop
        # G2: 编码后的 softmax - label
        G2 = probs - one_hot
        # W2梯度: (Z1^T @ G2) / m_batch.
        gradW2 = (Z1.T @ G2) / m_batch

        # Propagate the gradients back through the ReLU nonlinearity.
        # (Z1 > 0) is the indicator of activated neurons.
        # ReLU 反向传播，Z1 大于 0 表示激活
        G1 = (G2 @ W2.T) * (Z1 > 0)

        # Gradient for W1: (X_batch^T @ G1) / m_batch.
        gradW1 = (X_batch.T @ G1) / m_batch

        # inplace 原地更新
        W1 -= lr * gradW1
        W2 -= lr * gradW2
```

Training a full neural network:

trains a two-layer network with 400 hidden units.

```python
import sys

# Reload the simple_ml module which has been cached from the earlier experiment
import importlib
import simple_ml
importlib.reload(simple_ml)

sys.path.append("src/")
from simple_ml import train_nn, parse_mnist

X_tr, y_tr = parse_mnist("data/train-images-idx3-ubyte.gz",
                         "data/train-labels-idx1-ubyte.gz")
X_te, y_te = parse_mnist("data/t10k-images-idx3-ubyte.gz",
                         "data/t10k-labels-idx1-ubyte.gz")
train_nn(X_tr, y_tr, X_te, y_te, hidden_dim=400, epochs=20, lr=0.2)
```

it achieve an error of 1.89% on MNIST.

### Question 6: Softmax regression in C++

pybind11, a header-only library, which allows you to implement the entire Python/C++ interface within a single C++ source library

```cpp
void softmax_regression_epoch_cpp(const float *X, const unsigned char *y,
								  float *theta, size_t m, size_t n, size_t k,
								  float lr, size_t batch) {

}
```

it requires passing some additional arguments because we are operating on raw pointers to the array data rather than any sort of higher-level "matrix" data structure.

Specifically, `X`, `y`, and `theta` are **pointers** to the raw data of the corresponding numpy arrays from the previous section; for 2D arrays, these are stored in C-style (row-major) format, meaning that the first row of $X$ is stored sequentially as the first bytes in `X`, then the second row, etc (this contrasts with _column major_ ordering, which stores the first _column_ of the matrirx sequentially, then the second column, etc).

> row major 行存储

As an illustration of how to access the data, note that because `X` represents a row-major, $m \times n$ matrix, if we want to access the $(i,j)$ element of $X$ (the element in the $i$th row and the $j$th column), we would use the index `X[i*n + j]`

pybind11 code that actually provides the Python interface, this code essentially just extracts the raw pointers from the provided inputs (using pybinds numpy interface), and then calls the corresponding `softmax_regression_epoch_cpp` function.

```cpp
PYBIND11_MODULE(simple_ml_ext, m) {
    m.def("softmax_regression_epoch_cpp",
        [](py::array_t<float, py::array::c_style> X,
           py::array_t<unsigned char, py::array::c_style> y,
           py::array_t<float, py::array::c_style> theta,
           float lr,
           int batch) {
        softmax_regression_epoch_cpp(
            static_cast<const float*>(X.request().ptr),
            static_cast<const unsigned char*>(y.request().ptr),
            static_cast<float*>(theta.request().ptr),
            X.request().shape[0],
            X.request().shape[1],
            theta.request().shape[1],
            lr,
            batch
           );
    },
    py::arg("X"), py::arg("y"), py::arg("theta"),
    py::arg("lr"), py::arg("batch"));
}
```

相比 python，用 cpp 实现 softmax regression 的难点在于矩阵大小和下标访问：

```cpp
void softmax_regression_epoch_cpp(const float *X, const unsigned char *y,
                                  float *theta, size_t m, size_t n, size_t k,
                                  float lr, size_t batch) {
    for (size_t i = 0; i < m; i += batch) {             // m 个样本，每次 batch
        size_t currentBatch = std::min(batch, m - i);   // 当前 batch 数量
        // Allocate a gradient buffer for theta (size n*k) and initialize to 0.
        float *grad = new float[n * k];                 //
        std::memset(grad, 0, sizeof(float) * n * k);

        // process the minibatch
        for (size_t s = 0; s < currentBatch; s++) {
            // forward: X_batch @ theta
            size_t exampleIdx = i + s;
            // logits = \theta^T X,  x \in R^n, \theta \in R^{n x k}
            // logits \in k dimension
            float *logits = new float[k];
            // matrix multiplication
            // Compute logits = X[exampleIdx] dot theta, for each class j.
            for (size_t j = 0; j < k; j++) {
                float dot = 0.0f;
                for (size_t l = 0; l < n; l++) {
                    // X is row-major: element (exampleIdx, l)
                    // theta is row-major: element (l, j)
                    dot += X[exampleIdx * n + l] * theta[l * k + j];
                }
                logits[j] = dot;
            }

            // exp(logits) / np.sum(exp(logtis))
            float sumExp = 0.0f;
            for (size_t j = 0; j < k; j++) {
                logits[j] = std::exp(logits[j]);
                sumExp += logits[j];
            }

            // one-hot encoding
            // compute the probability and the difference (error)
            // compared to the one-hot label indicator.
            for (size_t j = 0; j < k; j++) {
                float prob = logits[j] / sumExp;
                // If j equals the true label then indicator is 1, else 0.
                float delta = prob - ( (j == static_cast<size_t>(y[exampleIdx])) ? 1.0f : 0.0f );
                // Accumulate gradients for each feature.
                for (size_t l = 0; l < n; l++) {
                    grad[l * k + j] += X[exampleIdx * n + l] * delta;
                }
            }
            delete[] logits;
        }

        // update theta in-place
        // mean gradients over the mini-batch and update theta
        // divide by float(currentBatch) to avoid integer division issues
        for (size_t l = 0; l < n; l++) {
            for (size_t j = 0; j < k; j++) {
                theta[l*k + j] -= lr * (grad[l*k + j] / static_cast<float>(currentBatch));
            }
        }
        delete[] grad;
    }
}
```
