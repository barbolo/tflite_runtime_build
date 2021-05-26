# Build TensorFlow Lite Standalone Pip

> Based on:
> - https://github.com/tensorflow/tensorflow/tree/v2.5.0/tensorflow/lite/tools/pip_package
> - https://zenn.dev/pinto0309/articles/a0e40c2817f2ee
> - https://github.com/PINTO0309/TensorflowLite-bin

1. Define a workdir:

```bash
export MYWORKDIR=~/git/github
```

2. Clone repos:

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

3. Update build tools:

```bash
cp $MYWORKDIR/tflite_runtime_build/tensorflow/tools/ci_build/Dockerfile.cpu $MYWORKDIR/tensorflow/tensorflow/tools/ci_build/
cp $MYWORKDIR/tflite_runtime_build/tensorflow/tools/ci_build/install/install_deb_packages.sh $MYWORKDIR/tensorflow/tensorflow/tools/ci_build/install/
```

4. XNNPACK's multi-thread patch

> https://github.com/NobuoTsukamoto/tensorflow/commit/f6f106380ac86ccf61ea9b01395f2911c4a6403c

```bash
cd $MYWORKDIR/tensorflow
patch -p1 < $MYWORKDIR/tflite_runtime_build/xnnpack_multi_threads.patch
cp $MYWORKDIR/tflite_runtime_build/tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh $MYWORKDIR/tensorflow/tensorflow/lite/tools/pip_package/
```

5. Build with TF OP support (Flex delegate):

```bash
cd $MYWORKDIR/tensorflow

# instructions for macOS
## install bazel 3.7.2
curl -fLO "https://github.com/bazelbuild/bazel/releases/download/3.7.2/bazel-3.7.2-installer-darwin-x86_64.sh"
chmod +x "bazel-3.7.2-installer-darwin-x86_64.sh"
./bazel-3.7.2-installer-darwin-x86_64.sh

brew install swig jpeg zlib
pip3 install numpy pybind11
brew install grep
PATH="/usr/local/opt/grep/libexec/gnubin:$PATH" sh tensorflow/lite/tools/make/download_dependencies.sh
tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native
pip3 install tensorflow/lite/tools/pip_package/gen/tflite_pip/python3/dist/tflite_runtime-2.5.0-cp39-cp39-macosx_11_0_x86_64.whl
```


[TODO] Build in a linux docker container and install dependencies:

```bash
cd $MYWORKDIR/tensorflow
docker run -it -w /tensorflow -v $(pwd):/tensorflow python:3.7.10-buster bash
# inside docker bash:
apt-get update
apt-get install -y swig libjpeg-dev zlib1g-dev python3-dev python3-numpy
pip install numpy pybind11
sh tensorflow/lite/tools/make/download_dependencies.sh
sh tensorflow/lite/tools/pip_package/build_pip_package.sh

```


## Usage

```python
from tflite_runtime.interpreter import Interpreter
interpreter = Interpreter(model_path="foo.tflite", num_threads=4)
```
