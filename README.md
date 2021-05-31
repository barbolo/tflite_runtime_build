# Build TensorFlow Lite Standalone Pip

## Install precompiled tflite_runtime

```bash
# python 3.8 - linux - x86_64 - tflite_runtime 2.5.0
pip3 install https://github.com/barbolo/tflite_runtime_build/raw/main/dist/tflite_runtime-2.5.0-cp38-cp38-linux_x86_64.whl

# python 3.9 - macosx - x86_64 - tflite_runtime 2.5.0
pip3 install https://github.com/barbolo/tflite_runtime_build/raw/main/dist/tflite_runtime-2.5.0-cp39-cp39-macosx_11_0_x86_64.whl
```

## Instructions to build

Use these instructions to build `tflite_runtime` with:

- Custom Ops from MediaPipe (`MaxPoolingWithArgmax2D`, `MaxUnpooling2D` and `Convolution2DTransposeBias`);
- `XNNPACK` with multi-thread support.

> For `tflite_runtime` 2.5.0 you should quantize your model to `float16`, since `integer` operations are still not
> supported by `XNNPACK`. If you quantize to `int8` your model will run slower than `float16` or `float32` in desktop
> CPUs.

> Based on:
> - https://github.com/tensorflow/tensorflow/tree/v2.5.0/tensorflow/lite/tools/pip_package
> - https://www.tensorflow.org/lite/guide/reduce_binary_size?hl=en
> - https://zenn.dev/pinto0309/articles/a0e40c2817f2ee
> - https://github.com/PINTO0309/TensorflowLite-bin

### 1. Set a work directory:

```bash
export MYWORKDIR=~/git/github
```

### 2. Clone repos:

```bash
cd $MYWORKDIR
git clone -b v2.5.0 https://github.com/tensorflow/tensorflow.git
git clone --depth=1 https://github.com/barbolo/tflite_runtime_build.git
```

### 3. Include mediapipe custom operations:

```bash
cp $MYWORKDIR/tflite_runtime_build/mediapipe/util/tflite/operations/max_pool_argmax.cc $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/mediapipe/util/tflite/operations/max_pool_argmax.h $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/mediapipe/util/tflite/operations/max_unpooling.cc $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/mediapipe/util/tflite/operations/max_unpooling.h $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/mediapipe/util/tflite/operations/transpose_conv_bias.cc $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/mediapipe/util/tflite/operations/transpose_conv_bias.h $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/tensorflow/lite/kernels/register.cc $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/tensorflow/lite/kernels/register_ref.cc $MYWORKDIR/tensorflow/tensorflow/lite/kernels
cp $MYWORKDIR/tflite_runtime_build/tensorflow/lite/kernels/BUILD $MYWORKDIR/tensorflow/tensorflow/lite/kernels
```

### 4. Update build tools:

```bash
cp $MYWORKDIR/tflite_runtime_build/tensorflow/tools/ci_build/Dockerfile.cpu $MYWORKDIR/tensorflow/tensorflow/tools/ci_build/
cp $MYWORKDIR/tflite_runtime_build/tensorflow/tools/ci_build/install/install_deb_packages.sh $MYWORKDIR/tensorflow/tensorflow/tools/ci_build/install/
```

### 5. XNNPACK's multi-thread patch

> https://github.com/NobuoTsukamoto/tensorflow/commit/f6f106380ac86ccf61ea9b01395f2911c4a6403c

```bash
cd $MYWORKDIR/tensorflow
patch -p1 < $MYWORKDIR/tflite_runtime_build/xnnpack_multi_threads.patch
cp $MYWORKDIR/tflite_runtime_build/tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh $MYWORKDIR/tensorflow/tensorflow/lite/tools/pip_package/
```

### 6. Build with Bazel:

> If you need **Flex delegate**, set `CUSTOM_BAZEL_FLAGS="--define=tflite_pip_with_flex=true"` This will increase the
> size of the final `tflite_runtime` binary (from ~ 5MB to ~ 380MB).

#### macOS

Install bazel 3.7.2:

```bash
cd /tmp
curl -fLO "https://github.com/bazelbuild/bazel/releases/download/3.7.2/bazel-3.7.2-installer-darwin-x86_64.sh"
chmod +x "bazel-3.7.2-installer-darwin-x86_64.sh"
./bazel-3.7.2-installer-darwin-x86_64.sh
```

```bash
cd $MYWORKDIR/tensorflow
brew install swig jpeg zlib
pip3 install numpy~=1.19.2 pybind11
brew install grep
PATH="/usr/local/opt/grep/libexec/gnubin:$PATH" sh tensorflow/lite/tools/make/download_dependencies.sh
bazel clean
PYTHON_BIN_PATH=/usr/local/bin/python3 \
  CUSTOM_BAZEL_FLAGS="--define=tflite_with_xnnpack=true" \
  tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native
pip3 install tensorflow/lite/tools/pip_package/gen/tflite_pip/python3/dist/tflite_runtime-2.5.0-cp39-cp39-macosx_11_0_x86_64.whl
```

#### Amazon Linux 2

Start a container with Amazon Linux 2 + Python 3.8:

```bash
cd $MYWORKDIR/tensorflow
docker run -it --entrypoint="" -w /tensorflow -v $(pwd):/tensorflow amazon/aws-lambda-python:3.8 bash
```

Continue the build inside the container's bash:

Install bazel 3.7.2:

```bash
yum install -y zip unzip which tar gzip git-core gcc gcc-c++ perl
cd /tmp
curl -fLO "https://github.com/bazelbuild/bazel/releases/download/3.7.2/bazel-3.7.2-installer-linux-x86_64.sh"
chmod +x "bazel-3.7.2-installer-linux-x86_64.sh"
./bazel-3.7.2-installer-linux-x86_64.sh
```

```bash
cd /tensorflow
yum install -y swig libjpeg-turbo-devel zlib1g-dev
python3 -m pip install --upgrade pip
pip3 install numpy~=1.19.2 pybind11 wheel
sh tensorflow/lite/tools/make/download_dependencies.sh
bazel clean
PYTHON_BIN_PATH=/var/lang/bin/python3 \
  CUSTOM_BAZEL_FLAGS="--config=avx2_linux --config=mkl --define=tflite_with_xnnpack=true" \
  tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native
pip3 install tensorflow/lite/tools/pip_package/gen/tflite_pip/python3/dist/tflite_runtime-2.5.0-cp38-cp38-linux_x86_64.whl
```

## Usage

You can run a code like the one below against tflite models `foo.tflite` and `foo_quant.tflite` to
confirm the `tflite_runtime` is working and to check their inferences latencies.

```python
from tflite_runtime.interpreter import Interpreter
import numpy as np
from time import time

def evaluate_tflite(path):
  print("Loading:", path)
  start_time = time()
  interpreter = Interpreter(model_path=path, num_threads=1)
  interpreter.allocate_tensors()
  input_details = interpreter.get_input_details()
  output_details = interpreter.get_output_details()
  input_shape = input_details[0]['shape']
  for i in range(10):
    input_data = np.array(np.random.random_sample(input_shape), dtype=np.float32)
    interpreter.set_tensor(input_details[0]['index'], input_data)
    interpreter.invoke()
  print('100 inferences with {0} ({1} sec)'.format(path, time() - start_time))

evaluate_tflite('foo.tflite')
evaluate_tflite('foo_quant.tflite')
```

## Build with Flex delegate and with selective registration of kernels

**WIP - binary file is still large**

> Learn more reading the comments in:
> - `$MYWORKDIR/tensorflow/tensorflow/core/framework/selective_registration.h`.
> - `$MYWORKDIR/tensorflow/tensorflow/python/tools/print_selective_registration_header.py`.

Find out which ops should be included based on your model `foo.tflite`:

```bash
bazel build tensorflow/lite/tools:list_flex_ops_no_kernel_main
./bazel-bin/tensorflow/lite/tools/list_flex_ops_no_kernel_main --graphs=foo.tflite > foo.ops_list
```

Generate `ops_to_register.h`:

```bash
bazel build tensorflow/python/tools:print_selective_registration_header
./bazel-bin/tensorflow/python/tools/print_selective_registration_header --graphs=foo.ops_list --proto_fileformat=ops_list > ops_to_register.h
cp ops_to_register.h $MYWORKDIR/tensorflow/tensorflow/core/framework/
```

Build with selective registration of kernels:

```bash
PYTHON_BIN_PATH=/usr/local/bin/python3 \
  CUSTOM_BAZEL_FLAGS="--define=tflite_with_xnnpack=true --define=tflite_pip_with_flex=true --copt=-DSELECTIVE_REGISTRATION --copt=-DSUPPORT_SELECTIVE_REGISTRATION" \
  tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native
```
