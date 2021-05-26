# Build TensorFlow Lite Standalone Pip

## Install precompiled tflite_runtime

```bash
# python 3.9 - x86_64 - tensorflow 2.5.0
pip3 install https://github.com/barbolo/tflite_runtime_build/raw/main/dist/tflite_runtime-2.5.0-cp39-cp39-macosx_11_0_x86_64.whl
```

## Instructions to build

Use these instructions to build `tflite_runtime` with:

- Custom Ops from MediaPipe (`MaxPoolingWithArgmax2D`, `MaxUnpooling2D` and `Convolution2DTransposeBias`);
- `XNNPACK` with multi-thread support.

If you need Flex delegate, edit the file `tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh` and change
`tflite_pip_with_flex=false` with `tflite_pip_with_flex=true`. This will increase the size of the final binary (from
~ 5MB to ~ 380MB).

> Based on:
> - https://github.com/tensorflow/tensorflow/tree/v2.5.0/tensorflow/lite/tools/pip_package
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
git clone https://github.com/barbolo/tflite_runtime_build.git
```

2. Include mediapipe custom operations:

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

### 3. Update build tools:

```bash
cp $MYWORKDIR/tflite_runtime_build/tensorflow/tools/ci_build/Dockerfile.cpu $MYWORKDIR/tensorflow/tensorflow/tools/ci_build/
cp $MYWORKDIR/tflite_runtime_build/tensorflow/tools/ci_build/install/install_deb_packages.sh $MYWORKDIR/tensorflow/tensorflow/tools/ci_build/install/
```

### 4. XNNPACK's multi-thread patch

> https://github.com/NobuoTsukamoto/tensorflow/commit/f6f106380ac86ccf61ea9b01395f2911c4a6403c

```bash
cd $MYWORKDIR/tensorflow
patch -p1 < $MYWORKDIR/tflite_runtime_build/xnnpack_multi_threads.patch
cp $MYWORKDIR/tflite_runtime_build/tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh $MYWORKDIR/tensorflow/tensorflow/lite/tools/pip_package/
```

### 5. Build with Bazel:

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
pip3 install numpy pybind11
brew install grep
PATH="/usr/local/opt/grep/libexec/gnubin:$PATH" sh tensorflow/lite/tools/make/download_dependencies.sh
export PYTHON_BIN_PATH=/usr/local/bin/python3
bazel clean
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
wget https://github.com/bazelbuild/bazel/releases/download/0.4.4/bazel-0.4.4-jdk7-installer-linux-x86_64.sh
```

```bash
yum install -y swig libjpeg-turbo-devel zlib1g-dev
python3 -m pip install --upgrade pip
pip3 install numpy pybind11 wheel
sh tensorflow/lite/tools/make/download_dependencies.sh
export PYTHON_BIN_PATH=/var/lang/bin/python3.8
export BAZEL_FLAGS="--config=avx2_linux --config=mkl"
bazel clean
tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native
pip3 install tensorflow/lite/tools/pip_package/gen/tflite_pip/python3/dist/tflite_runtime-2.5.0-cp38-cp38-linux_x86_64.whl
```

## Usage

```python
from tflite_runtime.interpreter import Interpreter
interpreter = Interpreter(model_path="foo.tflite", num_threads=4)
```
