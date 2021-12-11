# TFKG - A Tensorflow and Keras Golang port

## This is experimental and quite nasty under the hood*

## Summary

TFKG is a library for defining, training, saving, and running Tensorflow/Keras models with single GPU acceleration all
in Golang.

## The future of this project

See `ideas-todo.md` for what's in store

## Tested Platfoms

| Platform | OS           | CPU   | GPU       | Env    | CPU Support | GPU Acceleration |
|----------|--------------|-------|-----------|--------|-------------|------------------|
| Linux    | Ubuntu 18.04 | Intel | RTX 3090  | Docker | Yes         | Yes              |
| Linux    | Ubuntu 18.04 | Intel | RTX 3090  | Binary | Yes         | Yes              |
| Windows  | 11           | AMD   | RTX 3080  | Docker | Yes         | Yes              |
| Mac      | macOS 12     | Intel | AMD 5500m | Docker | Yes         | No               |
| Mac      | macOS 12     | M1    | M1        | Docker | Yes         | No               |

## Find your version

Versions starting with v0 are liable to change radically.

- Tensorflow 2.6 experimental support: `go get github.com/codingbeard/tfkg v0.2.6.10`

## Requirements

### Docker

Linux environments are recommended, no GPU support on macOS and docker volumes are slow on macOS/Windows

- Install `docker` and `docker-compose`
- Run `make init-docker` first to build the container
- If you're using a M1 Apple Silicon Mac on macOS use `make init-docker-m1`
- For GPU support on Linux in the container see: https://www.tensorflow.org/install/docker#gpu_support
- For GPU support on Windows 11 in the container see: https://docs.nvidia.com/cuda/wsl-user-guide/index.html

### Raw binary

**Make sure to install the correct versions to match the version of this library**

- Tensorflow C library: https://www.tensorflow.org/install/lang_c
- Python 3.8 - the binary "python" must be on your path and the correct version
- Tensorflow Python library: https://www.tensorflow.org/install
- CUDA 11.2 and cuDNN 8.1 if using GPU acceleration: https://www.tensorflow.org/install/gpu#hardware_requirements

## Features

- Nvidia CUDA support on applicable platforms during Golang training/evaluation due to using the Tensorflow C library
- Define, train, evaluate, save, load, and infer Tensorflow compatible models all in Golang
- Load, shuffle, and preprocess csv datasets efficiently, even very large ones (tested on 300+GB csv file on a nvme ssd)
    - String Tokenizer
    - Float/Int normalization to between 0-1

## Keras model types supported

- `tensorflow.keras.Sequential` (Single input)
- `tensorflow.keras.Model` (Multiple input)

## Keras Layers supported

- `tensorflow.keras.layers.Input`
- `tensorflow.keras.layers.Dense`
- `tensorflow.keras.layers.Concatenate`
- `tensorflow.keras.layers.BatchNormalization`
- `tensorflow.keras.layers.Dropout`
- `tensorflow.keras.layers.Embedding`
- `tensorflow.keras.layers.LSTM`
- More coming soon
- Interfaced - you can define custom layers

## Problem types supported

- Sparse categorical classification
- More coming soon

## Metrics

- Accuracy
- False positive rate at true positive rate (Specificity at Sensitivity)
- True positive rate at false positive rate (Sensitivity at Specificity)

## Limitations

- Python Tensorflow Libraries are still required to use this library, though the docker container has it all
- This is an incomplete port of Tensorflow/Keras: There are many layers, metrics, and optimisers not yet ported
- There is no community support or documentation. You must understand Tensorflow/Keras and Golang well to have a chance
  of getting this working on a new project
- Loading/Training models saved by vanilla python Tensorflow/Keras is not supported, but it may be possible
- Using multiple GPU training is not supported
- Image datasets and preprocessing is not yet supported
- Class weighting only works for datasets with two categories currently: negative/positive. It /will/ cause unintended
  side effects if you have more than two categories with imbalanced classes

## Examples:

| Model Type | Dataset              | Problem type               | Layers                                     | Location                     |
|------------|----------------------|----------------------------|--------------------------------------------|------------------------------|
| Sequential | Iris                 | Categorical Classification | Input, Dense                               | `./examples/iris`            |
| Functional | Iris                 | Categorical Classification | Input, Dense, Concatenate                  | `./examples/multiple_inputs` |
| Functional | Fraudulent Job Specs | Categorical Classification | Input, Embedding, LSTM, Concatenate, Dense | `./examples/jobs`            |

To test it out run:

```
make init-docker
make examples-iris
```

Define a model:

```go
m := model.NewSequentialModel(
logger,
    errorHandler,
    layer.NewInput(tf.MakeShape(-1, 4), layer.Float32),
    layer.NewDense(100, layer.Float32, layer.DenseConfig{Activation: "swish"}),
    layer.NewDense(100, layer.Float32, layer.DenseConfig{Activation: "swish"}),
    layer.NewDense(dataset.NumCategoricalClasses(), layer.Float32, layer.DenseConfig{Activation: "softmax"}),
)

batchSize := 3
e = m.CompileAndLoad(batchSize)
if e != nil {
    return
}
```

Load a dataset:

```go
dataset, e := data.NewSingleFileDataset(
    logger,
    errorHandler,
    data.SingleFileDatasetConfig{
        FilePath:          "examples/iris/data/iris.data",
        CacheDir:          cacheDir,
        CategoryOffset:    4,
        TrainPercent:      0.8,
        ValPercent:        0.1,
        TestPercent:       0.1,
        IgnoreParseErrors: true,
    },
    preprocessor.NewProcessor(
        errorHandler,
        "petal_sizes",
        preprocessor.ProcessorConfig{
            CacheDir:    cacheDir,
            LineOffset:  0,
            DataLength:  4,
            RequiresFit: true,
            Divisor:     preprocessor.NewDivisor(errorHandler),
            Reader:      preprocessor.ReadCsvFloat32s,
            Converter:   preprocessor.ConvertDivisorToFloat32SliceTensor,
        },
    ),
)
if e != nil {
  errorHandler.Error(e)
  return
}

e = dataset.SaveProcessors(saveDir)
if e != nil {
    return
}
```

Train a model:

```go
m.Fit(
    dataset,
    model.FitConfig{
        Epochs:     10,
        Validation: true,
        BatchSize:  batchSize,
        PreFetch:   10,
        Verbose:    1,
        Metrics: []metric.Metric{
            &metric.SparseCategoricalAccuracy{
                Name:       "acc",
                Confidence: 0.5,
                Average:    true,
            },
        },
        Callbacks: []callback.Callback{
            &callback.Logger{
                FileLogger: logger,
            },
            &callback.Checkpoint{
                OnEvent:    callback.EventEnd,
                OnMode:     callback.ModeVal,
                MetricName: "val_acc",
                Compare:    callback.CheckpointCompareMax,
                SaveDir:    saveDir,
            },
        },
    },
)
```

Load and predict using a saved TFKG model:

```go
inference, e := data.NewInference(
    logger,
    errorHandler,
    saveDir,
    preprocessor.NewProcessor(
        errorHandler,
        "petal_sizes",
        preprocessor.ProcessorConfig{
            Converter: preprocessor.ConvertDivisorToFloat32SliceTensor,
        },
    ),
)
if e != nil {
    return
}

inputTensors, e := inference.GenerateInputs([][]float32{{6.0, 3.0, 4.8, 1.8}})
if e != nil {
    return
}

outputTensor, e := m.Predict(inputTensors...)
if e != nil {
    return
}

outputValues := outputTensor.Value().([][]float32)

logger.InfoF(
    "main",
    "Predicted classes: %s: %f, %s: %f, %s: %f",
    "Iris-setosa",
    outputValues[0][0],
    "Iris-versicolor",
    outputValues[0][1],
    "Iris-virginica",
    outputValues[0][2],
)
```

## *Nasty under the hood

The Tensorflow/Keras python package saves a Graph (see more: https://www.tensorflow.org/guide/intro_to_graphs) which can
be executed in other languages using their C library as long as there are C bindings.

The C library does not contain all the functionality of the python library when it comes to defining and saving models,
it can only execute Graphs.

The Graph is calculated in python based on your model configuration, and a lot of clever code on the part of the
developers in optimising the graph.

While possible, it is not currently feasible for me to generate the Graph in Golang, so I am relying on python to do so.

This means while the model is technically defined and trained in Golang, it just generates a json config string which
static python code uses to configure the model and then saves it ready for loading in Golang for training. For the
moment this is a needed evil.

If some kind soul wants to replicate Keras and Autograph to generate the Graph in Golang, feel free to make a pull
request. I may eventually do it, but it is not likely. There is a branch origin/scratch which allows you to investigate
the graph of a saved model.

## Tensorflow C and Python library in a docker container on M1 Apple Silicon

See: https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/lib_package/README.md

See: https://www.tensorflow.org/install/source#docker_linux_builds

Docker did not play nicely with the amd64 precompiled Tensorflow C library so I had to compile it from source with avx
disabled on a different linux amd64 machine.

The compiled libraries and licenses can be found at: https://github.com/CodingBeard/tfkg/releases/tag/v0.2.6.5 and need
to be placed in `./docker/tf-jupyter-golang-m1/`

These are the steps I took to compile the library from sources to make it work:

```
// On a linux amd64 machine with docker installed:
git clone https://github.com/tensorflow/tensorflow
cd tensorflow
git checkout v2.6.0
docker run -it -w /tensorflow_src -v $PWD:/mnt -v $PWD:/tensorflow_src -e HOST_PERMS="$(id -u):$(id -g)" tensorflow/tensorflow:devel-gpu bash
> apt update && apt install apt-transport-https curl gnupg
> curl -fsSL https://bazel.build/bazel-release.pub.gpg | gpg --dearmor > bazel.gpg && \
    mv bazel.gpg /etc/apt/trusted.gpg.d/ && \
    echo "deb [arch=amd64] https://storage.googleapis.com/bazel-apt stable jdk1.8" | tee /etc/apt/sources.list.d/bazel.list
> apt update && apt install bazel-3.7.2 nano
> nano .bazelrc
// add the lines after the existing build:cuda lines:
build:cuda --linkopt=-lm
build:cuda --linkopt=-ldl
build:cuda --host_linkopt=-lm
build:cuda --host_linkopt=-ldl
> ./configure 
// take the defaults EXCEPT :
// ... "--config=opt" is specified [Default is -Wno-sign-compare]: -mno-avx
// The below will compile it for a specific GPU, find your gpu's compute capability and enter it twice separated by a comma (3000 series is 8.6)
// ... TensorFlow only supports compute capabilities >= 3.5 [Default is: 3.5,7.0]: 8.6,8.6
> bazel-3.7.2 build --config=cuda --config=opt //tensorflow/tools/lib_package:libtensorflow
> mkdir output
> cp bazel-bin/tensorflow/tools/lib_package/libtensorflow.tar.gz ./output/
> cp bazel-bin/tensorflow/tools/lib_package/clicenses.tar ./output/
> rm -r bazel-*
> bazel-3.7.2 build --config=cuda --config=opt //tensorflow/tools/pip_package:build_pip_package
> ./bazel-bin/tensorflow/tools/pip_package/build_pip_package ./output/tf-2.6.0-gpu-noavx
> quit
// copy the libs and wheel from ./output into the TFKG project under ./docker/tf-jupyter-golang-m1
...
```

## Acknowledgements

Big shout out to github.com/galeone for their Tensorflow Golang fork for 2.6 and again for their article on how to train
a model in golang which helped me figure out how to then save the trained
variables: https://pgaleone.eu/tensorflow/go/2020/11/27/deploy-train-tesorflow-models-in-go-human-activity-recognition/