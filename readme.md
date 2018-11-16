# cross-compile tensorflow use hisilicon hisiv400 toolchain. 
# 使用海思hisiv400工具链交叉编译tensorflow

### Basic info
This repo contain patches and instructions for cross-compile tensorflow-1.9(with python api) using hisiv400 toolchain.  
Tested on hisilicon Hi3536(armv7-a) soc.  
The cross-compile procedure is painful, you can go to release for the .whl file(if you have same toolchain and soc) when you don't care how to do it.  
* [tensorflow-1.9.0](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/tensorflow-1.9.0-cp36-cp36m-linux_armv7l.whl)
* [absl-0.5.0](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/absl_py-0.5.0-py3-none-linux_armv7l.whl)
* [astor-0.6](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/astor-0.6-py2.py3-none-linux_armv7l.whl)
* [gast-0.2.0](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/gast-0.2.0-py3-none-linux_armv7l.whl)
* [protobuf-3.6.1](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/protobuf-3.6.1-py2.py3-none-linux_armv7l.whl)
* [six-1.11.0](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/six-1.11.0-py2.py3-none-linux_armv7l.whl)
* [wheel-1.0.0b1](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/wheel-1.0.0b1-py2.py3-none-linux_armv7l.whl)

### About hisiv300
Hisiv300 toolchain use uClibc, which is lighter than glibc. But uClibc don't contain fenv(which is needed by tf), so the cross compilation 
with hisiv300 is more difficult. I recommand not to use hisiv300 toolchain to cross-compile tensorflow. However, there is also a .whl for hisiv300 toolchain.

* [tensorflow-1.9.0(hisiv300)](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v0.9.0/tensorflow-1.9.0-cp35-none-linux_armv7l.whl)

### Other cross-compiled python packages.
Those cross-compiled python packages(for Hi3536 soc and hisiv400) are also uploaded:

* [numpy-1.6.0(with OpenBLAS support)](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/numpy-1.16.0.dev0+511787d-cp36-cp36m-linux_armv7l.whl)
* [opencv-python-3.4.2(with TBB support)](https://github.com/zhewang95/tensorflow-on-hisilicon/releases/download/v1.0.0/opencv_python-3.4.2+5b36c37-cp36-cp36m-linux_armv7l.whl)

### Cross compile tf(for Hi3536 with hisiv400) by yourself
1. before the cross compilation, you need:
    1. hisiv400 toolchain installed.
    2. bazel-1.15.0 or higher installed.
    3. python3 installed on both your host machine and target machine(which also need cross compilation), same python version will be better.
    4. for 32-bit soc(armv7l), do cross compilation on 32-bit host machine will be good choice.

2. cd to tensorflow repo, do these following things:
    1. run `git checkout remotes/origin/r1.9` under tensorflow repo.
    2. run `./configure`, set python location, set optimization flags to `-march=armv7-a` and set all other options to `n`.

3. apply the patch file:
```
cp tensorflow-on-hisi/hisiv400.patch tensorflow/
cd tensorflow
git apply hisiv400.patch
```
the patch do these things mainly:
* define the cross-compile toolchain.
* replace `lib64` with `lib` in all files cause Hi3536 is 32bit armv7l.
* repair missed `-ltensorflow_framework` dependency for some bazel targets.
* correct some errors caused by not properly inclueded `stdint.h`(But I still don't know why this header file can't be included correctly).
* disable intel vector instruction features(SSE, AVX .etc) check.
* remove tensorboard dependency.

4. bazel build
```
bazel build -c opt --copt=-DARM_NON_MOBILE \
    --copt="-fPIC" \
    --copt="-march=armv7-a" \
    --copt="-mfloat-abi=softfp" \
    --copt="-mfpu=neon-vfpv4" \
    --copt="-Wno-unused-function" \
    --copt="-Wno-sign-compare" \
    --copt="-ftree-vectorize" \
    --copt="-fomit-frame-pointer" \
    --copt="-mno-unaligned-access" \
    --copt="-fno-aggressive-loop-optimizations" \
    --cxxopt="-Wno-maybe-uninitialized" \
    --cxxopt="-Wno-narrowing" \
    --cxxopt="-Wno-unused" \
    --cxxopt="-Wno-comment" \
    --cxxopt="-Wno-unused-function" \
    --cxxopt="-Wno-sign-compare" \
    --cxxopt="-mfloat-abi=softfp" \
    --cxxopt="-mfpu=neon-vfpv4" \
    --cxxopt="-mno-unaligned-access" \
    --cxxopt="-fno-aggressive-loop-optimizations" \
    --linkopt="-mfloat-abi=softfp" \
    --linkopt="-mfpu=neon-vfpv4" \
    --linkopt="-mno-unaligned-access" \
    --linkopt="-fno-aggressive-loop-optimizations" \
    --verbose_failures \
    --conlyopt="-std=c99" \
    --crosstool_top=//local_arm_compiler:toolchain \
    --cpu=armv7-a \
    --config=opt \
    //tensorflow/tools/pip_package:build_pip_package
```

However, some external packages(grpc,boringssl) may still throw error during compilation. But they are easy to repair.  
And above steps don't gurantee that you can get everything done with one try. You still need basic cross compilation experiences.

### About dynamic library(.so) load error
When I try to cross-compile some library(like python, OpenBLAS, opencv-python, tensorflow), the most disgusting thing is not compile error but dynamic library load error. 
Through some struggle, I have summarized several countermeasures when ambiguous .so load errors are thrown:
1. "libxx.so: cannot open shared object file: No such file or directory": First you should check whether the library exists. If it surely does, this error may mean 
"you have wrong .so file generated". For example, you may try to load a x86 .so file on armv7l platform. You can use `readelf -h libxx.so` to check your .so file.
2. "symbol lookup error: xx.so: undefined symbol: yyy": This error means that the library(.so or .a) contains symbol "yyy" is not linked to when you compile library libxx.so. 
You should add "-lxx -L/directory/contain/libxx.so" to your compile flags. You can usually infer the missed library's name by "yyy". And `/lib/ld-xx.so --list libxx.so` will show 
you whether the required library is properly linked to.
3. "cannot import name xx" when import python package(suppose we name it yy): This error don't give any information about what's going wrong. But you can go to python's
 site-packages/yy directory and run `find . -name "*.so"` to list all .so files in package yy. You may find that the file name is complicated like `xx.cpython-36m-i386-linux-gnu.so`, 
which contain the python version and platfrom information. Rename it to `xx.so` and the problem will be solved. I guess that python will check the filename to ensure 
right .so library will be loaded. But for cross-compile you may generate wrong file name by chance. 

### Reference links
* [海思3536：osdrv编译过程中报错及解决方法](https://blog.csdn.net/u010168781/article/details/65637105)
* [andorid编译报错serve_image.c:32:18: error: storage size of ‘hints’ isn’t known](https://blog.csdn.net/mtbiao/article/details/77052659)
* [hi3559v100 sdk 编译错误](https://blog.csdn.net/ternence_hsu/article/details/71194893)
* [ARM40移植Python3.6.4](https://blog.csdn.net/jzzy_hony/article/details/79745136)
* [Cross Compiling Python Extensions](http://whatschrisdoing.com/blog/2009/10/16/cross-compiling-python-extensions/)
* [Opencv cross compilation for ARM based Linux systems](https://docs.opencv.org/2.4/doc/tutorials/introduction/crosscompilation/arm_crosscompile_with_cmake.html)
* [TensorFlow on 32-bit Linux?](https://stackoverflow.com/questions/33634525/tensorflow-on-32-bit-linux)
* [Configuring CROSSTOOL](https://docs.bazel.build/versions/master/tutorial/crosstool.html)
* [在Ubuntu 16.04上使用bazel交叉编译tensorflow](https://www.cnblogs.com/jojodru/p/7744630.html)
* [Raspbian 9 (Stretch): Failed to load native TensorFlow runtime](https://github.com/tensorflow/tensorflow/issues/17790)
* [Using the GNU Compiler Collection (GCC)](https://gcc.gnu.org/onlinedocs/gcc-4.8.0/gcc/)
