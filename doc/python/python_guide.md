# MXNet Python Guide

This page gives a general overvie of MXNet python package. MXNet contains a
mixed flavor of elements you might need to bake flexible and efficient
applications. There are mainly three concepts in MXNet:

* Numpy style [NDArray](#ndarray-numpy-style-tensor-computations-on-cpu-gpu) offers matrix and tensor computations on both CPU and
GPU, with automatic parallelization

* [Symbol](#symbolic-and-automatic-differentiation) makes defining a neural network extremely easy, and it provides
  automatic differentiation.

* [KVStore](#distributed-key-value-store) allows data synchronization between
  multi-GPUs and multi-machine easily

## NDArray: Numpy style tensor computations on CPU/GPU

`NDArray` is the basic operation unit in MXNet for matrix and tensor
computations. It is similar to `numpy.ndarray`, but with two additional
features:

1. **multiple devices**: all operations can be run on various devices including
CPU and GPU
2. **automatic parallelization**: all operations are automatically executed in
   parallel with each other

### Create and Initialization

We can create `NDArray` on either GPU or GPU

```python
>>> import mxnet as mx
>>> a = mx.nd.empty((2, 3)) # create a 2-by-3 matrix on cpu
>>> b = mx.nd.empty((2, 3), mx.gpu()) # create a 2-by-3 matrix on gpu 0
>>> c = mx.nd.empty((2, 3), mx.gpu(2)) # create a 2-by-3 matrix on gpu 2
>>> c.shape # get shape
(2L, 3L)
>>> c.context # get device info
Context(device_type=gpu, device_id=2)
```

They can be initialized by various ways:

```python
>>> a = mx.nd.zeros((2, 3)) # create a 2-by-3 matrix and filled with 0
>>> b = mx.nd.ones((2, 3))  # create a 2-by-3 matrix and filled with 1
>>> b[:] = 2 # assign all elements of b with 2
```

We can copy the value from one to anther, even if they sit on different devices

```python
>>> a = mx.nd.ones((2, 3))
>>> b = mx.nd.zeros((2, 3), mx.gpu())
>>> a.copyto(b) # copy data from cpu to gpu
```

We can also convert `NDArray` to `numpy.ndarray`

```python
>>> a = mx.nd.ones((2, 3))
>>> b = a.asnumpy()
>>> type(b)
<type 'numpy.ndarray'>
>>> print b
[[ 1.  1.  1.]
 [ 1.  1.  1.]]
```

and verse vice

```python
>>> a = mx.nd.empty((2, 3))
>>> a[:] = np.random.uniform(-0.1, 0.1, a.shape)
>>> print a.asnumpy()
[[-0.06821112 -0.03704893  0.06688045]
 [ 0.09947646 -0.07700162  0.07681718]]
```

### Basic Operations

#### Elemental-wise operations

In default, `NDArray` performs elemental-wise operations:

```python
>>> a = mx.nd.ones((2, 3)) * 2
>>> b = mx.nd.ones((2, 3)) * 4
>>> print a.asnumpy()
[[ 4.  4.  4.]
 [ 4.  4.  4.]]
>>> c = a + b
>>> print c.asnumpy()
[[ 6.  6.  6.]
 [ 6.  6.  6.]]
>>> d = a * b
>>> print d.asnumpy()
[[ 8.  8.  8.]
 [ 8.  8.  8.]]
```

If two `NDArray` sit on different devices, we need explicitly move them into the
same one. The following example performing computations on GPU 0:

```python
>>> a = mx.nd.ones((2, 3)) * 2
>>> b = mx.nd.ones((2, 3), mx.gpu()) * 3
>>> c = a.copyto(mx.gpu()) * b
>>> print c.asnumpy()
[[ 6.  6.  6.]
 [ 6.  6.  6.]]
```

#### Indexing

TODO

#### Linear Algebra

TODO

### Load and Save

There are two ways to save data to (load from) disks easily. The first way uses
`pickle`.  `NDArray` is pickle compatible, which means you can simply pickle the
NArray like what you did with `numpy.ndarray`.

```python
>>> import mxnet as mx
>>> import pickle as pkl

>>> a = mx.nd.ones((2, 3)) * 2
>>> data = pkl.dumps(a)
>>> b = pkl.loads(data)
>>> print b.asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
```

On the second way, we directly dump a list of `NDArray` into disk in binary format.

```python
>>> a = mx.nd.ones((2,3))*2
>>> b = mx.nd.ones((2,3))*3
>>> mx.nd.save('mydata.bin', [a, b])
>>> c = mx.nd.load('mydata.bin')
>>> print c[0].asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
>>> print c[1].asnumpy()
[[ 3.  3.  3.]
 [ 3.  3.  3.]]
```

We can also dump a dict.

```python
>>> mx.nd.save('mydata.bin', {'a':a, 'b':b})
>>> c = mx.nd.load('mydata.bin')
>>> print c['a'].asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
>>> print c['b'].asnumpy()
[[ 3.  3.  3.]
 [ 3.  3.  3.]]
```

In addition, we have setup the distributed filesystem such as S3 and HDFS, we
can directly save to and load from them. For example:

```python
>>> mx.nd.save('s3://mybucket/mydata.bin', [a,b])
>>> mx.nd.save('hdfs///users/myname/mydata.bin', [a,b])
```

### Parallelization

The operations of `NDArray` are executed by third libraries such as `cblas`,
`mkl`, and `cuda`. In default, each operation is executed by multi-threads. In
addition, `NDArray` can execute operations in parallel. It is desirable when we
use multiple resources such as CPU, GPU cards, and CPU-to-GPU memory bandwidth.

For example, if we write `a += 1` followed by `b += 1`, and `a` is on CPU while
`b` is on GPU, then want to execute them in parallel to improve the
efficiency. Furthermore, data copy between CPU and GPU are also expensive, we
hope to run it parallel with other computations as well.

However, finding the codes can be executed in parallel by eye is hard. In the
following example, `a+=1` and `c*=3` can be executed in parallel, but `a+=1` and
`b*=3` should be in sequential.

```python
a = mx.nd.ones((2,3))
b = a
c = a.copyto(mx.cpu())
a += 1
b *= 3
c *= 3
```

Luckily, MXNet can automatically resolve the dependencies and
execute operations in parallel with correctness guaranteed. In other words, we
can write program as by assuming there is only a single thread, while MXNet will
automatically dispatch it into multi-devices, such as multi GPU cards or multi
machines.

It is achieved by lazy evaluation. Any operation we write down is issued into a
internal DAG engine, and then returned. For example, if we run `a += 1`, it
returns immediately after pushing the plus operator to the engine. This
asynchronous allows us to push more operators to the engine, so it can determine
the read and write dependency and find a best way to execute them in
parallel.

The actual computations are finished if we want to copy the results into some
other place, such as `print a.asnumpy()` or `mx.nd.save([a])`. Therefore, if we
want to write highly parallelized codes, we only need to postpone when we need
the results.


## Symbolic and Automatic Differentiation

Now you have seen the power of NArray of MXNet. It seems to be interesting and
we are ready to build some real deep learning.  Hmm, this seems to be really
exciting, but wait, do we need to build things from scratch?  It seems that we
need to re-implement all the layers in deep learning toolkits such as
[CXXNet](https://github.com/dmlc/cxxnet) in NArray?  Well, you do not have
to. There is a Symbolic API in MXNet that readily helps you to do all these.

More importantly, the Symbolic API is designed to bring in the advantage of C++
static layers(operators) to ***maximumly optimizes the performance and memory***
that is even better than CXXNet. Sounds exciting? Let us get started on this.

### Creating Symbols

A common way to create a neural network is to create it via some way of
configuration file or API.  The following code creates a configuration two layer
perceptrons.

```python
import mxnet.symbol as sym
data = sym.Variable('data')
net = sym.FullyConnected(data=data, name='fc1', num_hidden=128)
net = sym.Activation(data=net, name='relu1', act_type="relu")
net = sym.FullyConnected(data=net, name='fc2', num_hidden=10)
net = sym.Softmax(data=net, name = 'sm')
```

If you are familiar with tools such as cxxnet or caffe, the ```Symbol``` object
is like configuration files that configures the network structure. If you are
more familiar with tools like theano, the ```Symbol``` object something that
defines the computation graph. Basically, it creates a computation graph that
defines the forward pass of neural network.

The Configuration API allows you to define the computation graph via
compositions.  If you have not used symbolic configuration tools like theano
before, one thing to note is that the ```net``` can also be viewed as function
that have input arguments.

You can get the list of arguments by calling ```Symbol.list_arguments```.

```python
>>> net.list_arguments()
['data', 'fc1_weight', 'fc1_bias', 'fc2_weight', 'fc2_bias']
```

In our example, you can find that the arguments contains the parameters in each
layer, as well as input data.  One thing that worth noticing is that the
argument names like ```fc1_weight``` are automatically generated because it was
not specified in creation of fc1.  You can also specify it explicitly, like the
following code.

```python
>>> import mxnet.symbol as sym
>>> data = sym.Variable('data')
>>> w = sym.Variable('myweight')
>>> net = sym.FullyConnected(data=data, weight=w,
                             name='fc1', num_hidden=128)
>>> net.list_arguments()
['data', 'myweight', 'fc1_bias']
```

Besides the coarse grained neuralnet operators such as FullyConnected,
Convolution.  MXNet also provides fine graned operations such as elementwise
add, multiplications.  The following example first performs an elementwise add
between two symbols, then feed them to the FullyConnected operator.

```python
>>> import mxnet.symbol as sym
>>> lhs = sym.Variable('data1')
>>> rhs = sym.Variable('data2')
>>> net = sym.FullyConnected(data=lhs + rhs,
                             name='fc1', num_hidden=128)
>>> net.list_arguments()
['data1', 'data2', 'fc1_weight', 'fc1_bias']
```

### More Complicated Composition

In the previous example, Symbols are constructed in a forward compositional way.
Besides doing things in a forward compistion way. You can also treat composed
symbols as functions, and apply them to existing symbols.

```python
>>> import mxnet.symbol as sym
>>> data = sym.Variable('data')
>>> net = sym.FullyConnected(data=data,
                             name='fc1', num_hidden=128)
>>> net.list_arguments()
['data', 'fc1_weight', 'fc1_bias']
>>> data2 = sym.Variable('data2')
>>> in_net = sym.FullyConnected(data=data,
                                name='in', num_hidden=128)
>>> composed_net = net(data=in_net, name='compose')
>>> composed_net.list_arguments()
['data2', 'in_weight', 'in_bias', 'compose_fc1_weight', 'compose_fc1_bias']
```

In the above example, net is used a function to apply to an existing symbol
```in_net```, the resulting composed_net will replace the original ```data``` by
the the in_net instead. This is useful when you want to change the input of some
neural-net to be other structure.

### Shape Inference

Now we have defined the computation graph. A common problem in the computation
graph, is to figure out shapes of each parameters.  Usually, we want to know the
shape of all the weights, bias and outputs.

You can use ```Symbol.infer_shape``` to do that. THe shape inference function
allows you to pass in shapes of arguments that you know,
and it will try to infer the shapes of all arguments and outputs.

```python
>>> import mxnet.symbol as sym
>>> data = sym.Variable('data')
>>> net = sym.FullyConnected(data=data, name='fc1',
                             num_hidden=10)
>>> arg_shape, out_shape = net.infer_shape(data=(100, 100))
>>> dict(zip(net.list_arguments(), arg_shape))
{'data': (100, 100), 'fc1_weight': (10, 100), 'fc1_bias': (10,)}
>>> out_shape
[(100, 10)]
```

In common practice, you only need to provide the shape of input data, and it
will automatically infers the shape of all the parameters.  You can always also
provide more shape information, such as shape of weights.  The ```infer_shape```
will detect if there is inconsitency in the shapes, and raise an Error if some
of them are inconsistent.

### Bind the Symbols

Symbols are configuration objects that represents a computation graph (a
configuration of neuralnet).  So far we have introduced how to build up the
computation graph (i.e. a configuration).  The remaining question is, how we can
do computation using the defined graph.

TODO.

### How Efficient is Symbolic API

In short, they design to be very efficienct in both memory and runtime.

The major reason for us to introduce Symbolic API, is to bring the efficient C++
operations in powerful toolkits such as cxxnet and caffe together with the
flexible dynamic NArray operations. All the memory and computation resources are
allocated statically during Bind, to maximize the runtime performance and memory
utilization.

The coarse grained operators are equivalent to cxxnet layers, which are
extremely efficient.  We also provide fine grained operators for more flexible
composition. Because we are also doing more inplace memory allocation, mxnet can
be ***more memory efficient*** than cxxnet, and gets to same runtime, with
greater flexiblity.

## Distributed Key-value Store

## How to Choose between APIs

You can mix them all as much as you like. Here are some guidelines
* Use Symbolic API and coarse grained operator to create established structure.
* Use fine-grained operator to extend parts of of more flexible symbolic graph.
* Do some dynamic NArray tricks, which are even more flexible, between the calls of forward and backward of executors.

We believe that different ways offers you different levels of flexibilty and efficiency. Normally you do not need to
be flexible in all parts of the networks, so we allow you to use the fast optimized parts,
and compose it flexibly with fine-grained operator or dynamic NArray. We believe such kind of mixture allows you to build
the deep learning architecture both efficiently and flexibly as your choice. To mix is to maximize the peformance and flexiblity.