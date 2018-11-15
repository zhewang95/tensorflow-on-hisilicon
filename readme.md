# cross compile tensorflow use hisilicon hisiv400 toolchain. 
# 使用海思hisiv400工具链交叉编译tensorflow

### Basic info
This repo contain patches and instructions for cross compile tensorflow-1.9(with python api) using hisiv400 toolchain.  
Tested on hisilicon Hi3536(armv7-a) soc.  
The cross compile procedure is painful, you can go to release for the .whl file(if you have same toolchain and soc) if you don't care how to do it.  

### About hisiv300
Hisiv300 toolchain use uClibc, which is lighter than glibc. But uClibc don't contain fenv(which is needed by tf), so the cross compilation 
with hisiv300 is more difficult. I recommand not to use hisiv300 toolchain to do cross compilation. However, there is also a .whl for hisiv300 toolchain.

### Other cross compiled python packages.
Those cross compiled python packages(for Hi3536 soc and cross compiled use hisiv400) are also uploaded:
* numpy(with OpenBLAS support)
* opencv-python(with TBB support)

### Cross compile tf(for Hi3536 with hisiv400) by yourself
1. before the cross compilation, you need:
    1. hisiv400 toolchain installed.
    2. bazel-1.15.0 or higher.
    3. python3 installed on both your host machine and target machine(which also need cross compilation), same python version will be better.
    4. for 32-bit soc(armv7l), do cross compilation on 32-bit host machine will be better choice.

2. cd to tensorflow repo, do these following things:
    1. run `git checkout remotes/origin/r1.10` under tensorflow repo.
    2. run `./configure`, set python location, set optimization flags to `-march=armv7-a` and set all other options to `n`.

3. apply the patch file:
```
cp tensorflow-on-hisi/hisiv400.diff tensorflow/
cd tensorflow
git apply hisiv400.diff
```
the patch do these things mainly:
    1. define the cross compile toolchain.
    2. replace `lib64` with `lib` in all files cause Hi3536 is 32bit armv7l.
    3. repair missed `-ltensorflow_framework` dependencies for some bazel target.
    4. correct some errors caused by not properly inclueded `stdint.h`(But I still don't know why this header file can't be included correctly).
    5. disable intel vector instruction features(SSE, AVX .etc) check.
    6. remove grpcio and tensorboard dependencies.

However, some external packages(grpc,boringssl) may still throw error during compilation. But they are easy to repair.  
And above steps don't gurantee that you can get everything done with one try. You may still need basic cross-compilation experiences.

### About dynamic library(.so) load error
When I try to cross compile some library(like python, OpenBLAS, opencv-python, tensorflow), the most disgusting thing is not compile error but dynamic library load error. 
Through some struggle, I have summarized some countermeasures when ambiguous .so load errors are thrown:
1. "libxx.so: cannot open shared object file: No such file or directory": First you should check whether the library exists. If it surely exists, this error may mean 
"you have wrong .so file generated, maybe it's not for this platform". For example, you may load a x86 .so file on armv7l platform.
2. "symbol lookup error: xx.so: undefined symbol: yyy": This error means that the library(.so or .a) contains symbol "yyy" is not linked when you compile library libxx.so. 
You should add "-lxx -L/path/to/libxx.so" to your compile flags. You can usually infer the missed library's name by "yyy".
3. "cannot import name xx" when import python package(suppose we name it yy): This error don't give any information about what's going wrong. But you can go to python's
 site-packages/yy directory and run `find . -name "*.so"` to list all .so file in package yy. You may find that the file name is complicated like `xx.cpython-36m-i386-linux-gnu.so', 
which contain the python version and platfrom information. You can rename it to `xx.so` and the problem will be solved. I guess that python will check the filename to ensure 
right .so library will be loaded. But for cross compile you may generate wrong file name sometimes. 

### Reference links
* ![海思3536：osdrv编译过程中报错及解决方法](https://blog.csdn.net/u010168781/article/details/65637105)
* ![ARM40移植Python3.6.4](https://blog.csdn.net/jzzy_hony/article/details/79745136)
* ![Cross Compiling Python Extensions](http://whatschrisdoing.com/blog/2009/10/16/cross-compiling-python-extensions/)
* ![Opencv cross compilation for ARM based Linux systems](https://docs.opencv.org/2.4/doc/tutorials/introduction/crosscompilation/arm_crosscompile_with_cmake.html)
* ![TensorFlow on 32-bit Linux?](https://stackoverflow.com/questions/33634525/tensorflow-on-32-bit-linux)
* ![Configuring CROSSTOOL](https://docs.bazel.build/versions/master/tutorial/crosstool.html)
* ![在Ubuntu 16.04上使用bazel交叉编译tensorflow](https://www.cnblogs.com/jojodru/p/7744630.html)
* ![Raspbian 9 (Stretch): Failed to load native TensorFlow runtime](https://github.com/tensorflow/tensorflow/issues/17790)
