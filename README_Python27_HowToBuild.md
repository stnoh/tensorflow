# Build Tensorflow for Python27/Windows

- You can get the Python2.7 built with VS2015 from [p-nand-q.com](http://p-nand-q.com/python/building-python-27-with-visual_studio.html).  
  Download Python 2.7.10 64-bit / Visual Studio 2015, and extract the Python 2.7.10 in C:\Python27.  
  To prepare build, you want to read [another markdown document](./README_Python27_PythonSetting.md).  
- Use tensorflow > 1.10.x . Until 1.10.x, tensorflow has a severe compilation speed issue due to Eigen inlining.  
  From 1.11.x, the number of files to compile is quite reduced (8,000+ -> 5,000+),  
  and you can also use the EIGEN_STRONG_INLINE option.  
- **Do not** use [cmake] for your build anymore.  
  [bazel] is not only the officially supported system for tf, but also the faster than [cmake] !  

[bazel]:https://www.bazel.build/
[cmake]:https://cmake.org/



## Preparation of Build Environment in Windows 10

### Installation Environments

- **Account setting**: your account should *not* have special characters nor spaces [[link]](https://github.com/bazelbuild/bazel/issues/374). Just keep using ASCII characters.  
- To install bazel, follow the instruction [here](https://docs.bazel.build/versions/master/install-windows.html). My toolset is below.  
  + [bazel] : **0.15.0-windows-x86_64.exe** (C:\bazel)  
  + [MSYS2](https://www.msys2.org/): **msys2-x86_64-20180531.exe** (C:\MSYS64)  
  + [JDK8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html): **jdk-8u192-windows-x64.exe**  
- Set Python2.7.10 above as a build platform.  
  Add environment path to Windows (from: This PC > Property > Advanced system settings > Environment Variables...)  



## Build-CPU

- I succeeded to build **tensorflow-1.11.0** which is the latest stable release in 2018-Oct-22.  
- If you use bazel, the overall compilation is quite fast (but it still takes 40+ minutes).  
  You may feel the codes in **tensorflow/core/kernels** still takes a lot of time to compile.  

If you pull the tensorflow code from official page, you may encounter few issues in build.  
Here I show those errors and how I patched these issues...  

### Fix #1 for "AttributeError: 'file' object has no attribute 'readable' "

- The main problem is in **def_file_filter.py**, which is generated from **tensorflow/tools/def_file_filter/def_file_filter.py.tpl**  
  I added some subroutine and changed 1 line of code based on [Vek's method in this StackOverflow thread](
https://stackoverflow.com/questions/34447623/wrap-an-open-stream-with-io-textiowrapper) ...  
  ```Python
  # !!!!!!!!!!!!!!!!!!!!!
  # QUICK FIX for Python2
  import six  

  if six.PY2:
    class _ReadableWrapper(object):
      def __init__(self, raw):
          self._raw = raw

      def readable(self):
          return True

      def writable(self):
          return False

      def seekable(self):
          return True

      def __getattr__(self, name):
          return getattr(self._raw, name)

  def wrap_text(stream, *args, **kwargs):
    # Note: order important here, as 'file' doesn't exist in Python 3
    if six.PY2 and isinstance(stream, file):
        stream = io.BufferedReader(_ReadableWrapper(stream))

    return io.TextIOWrapper(stream)
  # QUICK FIX for Python2
  # !!!!!!!!!!!!!!!!!!!!!
  
  ...

    #for idx, line in enumerate(io.TextIOWrapper(proc.stdout, encoding="utf-8")):
    for idx, line in enumerate(wrap_text(proc.stdout, encoding="utf-8")): # QUICK FIX for Python2
  ```

### Fix #2 for "NotImplementedError: cannot determine number of cpus"

- The main reason is the not implemented feature in **multiprocessing** package ([Issue#17914](https://bugs.python.org/issue17914)).  
  If you want to know the detailed information, read this blog's article (Japanese): 
http://hhsprings.pinoko.jp/site-hhs/2016/03/python-3-3
- There are two ways to fix this issue.

#### Option 1. [Recommended] Quick fix on **multiprocessing** package.  

If you can access multiprocessing source code directly, it is worth to try this.  

1.  Remove bytecode of **multiprocessing**: **C:\Python27\Lib\multiprocessing\\_\_init\_\_.pyc**  
2.  Add the single line before "NotImplementedError" in: **C:\Python27\Lib\multiprocessing\\_\_init\_\_.py**  
```Python
def cpu_count():
    '''
    Returns the number of CPUs in the system
    '''
    if sys.platform == 'win32':
        try:
            num = int(os.environ['NUMBER_OF_PROCESSORS'])
        except (ValueError, KeyError):
            num = 0
    elif 'bsd' in sys.platform or sys.platform == 'darwin':
        comm = '/sbin/sysctl -n hw.ncpu'
        if sys.platform == 'darwin':
            comm = '/usr' + comm
        try:
            with os.popen(comm) as p:
                num = int(p.read())
        except ValueError:
            num = 0
    else:
        try:
            num = os.sysconf('SC_NPROCESSORS_ONLN')
        except (ValueError, OSError, AttributeError):
            num = 0

    if num >= 1:
        return num
    else:
        return 1 # [QUICK FIX]
        raise NotImplementedError('cannot determine number of cpus')
```
3.  Now you can run the build again. So simple!  
    After build is success, I recommend to turn it back of **multiprocessing** above.  

#### Option 2. Edit source code and build again.

I do not recommend this, because 
If you cannot edit the multiprocessing's source code, then you need to modify code to build.

1.  Modify: **tensorflow/tools/test/system_info_lib.py**  
    In tensorflow, **multiprocessing.cpu_count()** is used in this code only.  
    Open it and edit like this:  
    ```Python
    #cpu_info.num_cores = multiprocessing.cpu_count() # original code

    # Quick fix for "NotImplementedError" in Python2/Windows
    try:
      cpu_count = multiprocessing.cpu_count()
    except NotImplementedError:
      cpu_count = 1
    cpu_info.num_cores = cpu_count
    ```
2.  You also need to modify **scipy** package.  
    In scipy, **multiprocessing.cpu_count()** is used in: **scipy/spatial/ckdtree/ckdtree.pyx**  
    Open and edit it similar way:  
    ```Python
    #number_of_processors = cpu_count() # original 

    # Quick fix for "NotImplementedError" in Python2/Windows
    try:
      cpu_cnt = multiprocessing.cpu_count()
    except NotImplementedError:
      cpu_cnt = 1
    number_of_processors = cpu_cnt
    ```
3.  Build and install scipy if needed, and build tensorflow again.  



## Build-GPU

- Before jumping into GPU version, watch out on your compute capability setting through **configure.py**.  
  The default value is "3.5,**7.0**", which is not allowed in most PCs (e.g. GeForce 1080 GTX should be "6.1").  
  Full list can be found here: https://en.wikipedia.org/wiki/CUDA#GPUs_supported  
- In tensorflow-1.11.0, [eigen]-3.3 is used. However, this version has an issues in CUDA inline: http://eigen.tuxfamily.org/bz/show_bug.cgi?id=1526  
  This problem causes the error messeage in the compilation, like:  
  ```sh
  Error: "{filepath}/{kernalfile}_gpu.cu.compute_70.cpp1.ii"**
  ```
- This issue is also known to tensorflow community ([Pull request #21119](https://github.com/tensorflow/tensorflow/pull/21119)), but there is no bazel patch to automize this process yet (Oct-22-2018).  
  To solve this issue, I manually patched the **half.h** after the downloading.  
  + First, find it from bazel's working directory.  
    The path is usually like this:  
    C:\Users\\**{your_account}\\\_bazel\_{your_account}\\{hash_string}**\\external\\eigen_archive\\Eigen\\src\\Core\\arch\\CUDA\\half.h  
  + After Eigen is downloaded and checked, you need to change the file directly:  
    https://gist.github.com/bstriner/a7fb0a8da1f830900fa932652439ed44  
    ```C++
    EIGEN_STRONG_INLINE __device__ half operator + (const half& a, const half& b) {
      //return __hadd(a, b);
      return __hadd(::__half(a), ::__half(b));
    }

    ...

    EIGEN_STRONG_INLINE __device__ half operator / (const half& a, const half& b) {
      //float num = __half2float(a);
      //float denom = __half2float(b);
      //return __float2half(num / denom);
      return __hdiv(a, b);
    }
    ```
- I am quite sure that this can be done in **/tensorflow/workspace.bzl** with bazel command, but I do not know how to do this.  

[eigen]:http://eigen.tuxfamily.org
