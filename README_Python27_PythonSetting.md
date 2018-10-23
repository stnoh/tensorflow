# Non-pure packages of Python

- Maya-Python in Maya 2018 is 2.7.11, built with MSC v.1900 (Visual Studio 2015, aka vc14).  
- You can get the similar setting from [p-nand-q.com](http://p-nand-q.com/python/building-python-27-with-visual_studio.html).  
  We use this our testbed for installation.  
  Download Python 2.7.10 64-bit / Visual Studio 2015, and extract where you want to.  


## 1. Uninstall the initial packages

These bother the installation process.  
Please remove these at first: [Cython],[numpy],**Markdown**,**reportlab**


## 2. Install and upgrade pip

- Download [get-pip.py](https://bootstrap.pypa.io/get-pip.py) and run.  
  ```sh
  > python get-pip.py
  ```
- upgrade **pip** and **setuptools**.  
  ```sh
  > pip install --upgrade pip setuptools
  ```


## 3. Install custom-build wheels [CAUTION !]

This process should be done **before** the installtation of other packages from remote.  
The package list are follow: [Cython],[numpy],[h5py],[scipy]  

If you want to build the wheels by yourself, you can refer the following commands.  

```sh
> path=%PATH%;C:\Python27;C:\Python27\Scripts
> python setup.py build
> python setup.py sdist          # create zip file that contains .py sources, can be skipped
> python setup.py bdist_wheel    # create wheel for installer
> cd dist
> pip install {package_name}.whl # install from wheel
```

[Cython]:https://github.com/cython/cython
[numpy]:https://github.com/numpy/numpy
[h5py]:https://github.com/h5py/h5py
[scipy]:https://github.com/scipy/scipy

### [Cython]

My custom wheel: **Cython-0.29-cp27-cp27m-win_amd64.whl**  

- It is stand-alone, so there is no need to include DLLs inside.


### [numpy]

My custom wheel: **numpy-1.14.5+mkl-cp27-cp27-win_amd64.whl**  

- Simple build is not so difficult, but build with [mkl] is slightly tricky.  
  Anyway, it needs **icl.exe** (Intel C++ Compiler). Install [Intel Parallel Studio XE](https://software.intel.com/en-us/parallel-studio-xe).  
- Before build the [numpy], we need to modify the code slightly.  
  Open **numpy/\_\_init\_\_.py** and add the following code after importing the packages.  
  ```Python
  # !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  # Hack to package mkl dll's with numpy
  import os
  mkl_dll_path   = r'%s\mkl'%os.path.dirname(os.path.abspath(__file__))
  if 'path' in os.environ:
    if not mkl_dll_path.lower() in [os.path.abspath(x).lower() for x in os.environ['path'].split(os.pathsep)]:
      os.environ['path'] = '%s;%s'%(mkl_dll_path,os.environ['path'])
  else:
    os.environ['path'] = mkl_dll_path
  # end of hack
  # !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  ```
- Also, copy **site.cfg.example** as **site.cfg** and activate the commented out lines like:  
  ```sh
  [mkl]
  include_dirs = C:\Program Files (x86)\IntelSWTools\compilers_and_libraries_2018\windows\mkl\include
  library_dirs = C:\Program Files (x86)\IntelSWTools\compilers_and_libraries_2018\windows\mkl\lib\intel64
  mkl_libs = mkl_rt
  lapack_libs =
  ```
- Activate **icl.exe** from the Startup menu (e.g. Intel Compiler 18.0 Update 5 Intel(R) 64 Visual Studio 2015). Then, run the next command to build.  
  ```sh
  > python setup.py config --compiler=intelemw build_clib --compiler=intelemw --fcompiler=intelvem build_ext --compiler=intelemw --fcompiler=intelvem build
  ```
- Copy the DLLs of redistributable package (under **redist**/intel64_win/{compiler,mkl,tbb} ) into **build/lib.win-amd64-2.7/numpy/mkl** to create a stand-alone wheel.  
  Now we can build a stand-alone wheel.  
  ```sh
  > python setup.py bdist_wheel
  > pip install dist\numpy-1.14.5-cp27-cp27-win_amd64.whl
  ```

[mkl]:https://software.intel.com/en-us/mkl


### [h5py]

My custom wheel: **h5py-2.8.0-cp27-cp27m-win_amd64.whl**  

- To compile h5py, we also need HDF5's include/lib.  
  Copy those files from its installed directory:  
  + **C:\Python27\include** <- "C:\Program Files\HDF_Group\HDF5\1.8.21\include\*"  
  + **C:\Python27\libs**    <- "C:\Program Files\HDF_Group\HDF5\1.8.21\lib\*"  
- We need to set some options to build h5py.  
  For the detail, you can refer this site: http://docs.h5py.org/en/latest/build.html  
  ```sh
  > set HDF5_DIR=C:/Program Files/HDF_Group/HDF5
  > set HDF5_VERSION=1.8.21
  > python setup.py configure --reset
  > python setup.py build
  ```
- Copy the DLLs of HDF5 to create a stand-alone wheel.  
  hdf5.dll and hdf5_hl.dll to **build/lib.win-amd64-2.7/h5py**  
  After this, run to create a wheel.  
  ```sh
  > python setup.py bdist_wheel
  > pip install dist\h5py-2.8.0-cp27-cp27-win_amd64.whl
  ```


### [scipy]

My custom wheel: **scipy-1.1.0+mkl-cp27-cp27m-win_amd64.whl**

- Before build the [scipy], we need to modify the code slightly.  
  Open **scipy/\_lib/\_fpumode.c** and add the code below.  
  ```C
  ...
  #ifdef _MSC_VER
  #pragma float_control(precise, on) // add this line
  #pragma fenv_access (on)
  #endif
  ...
  ```
  To know the detail, read this pull request: https://github.com/scipy/scipy/pull/8918  
- Compared to numpy, scipy is easy to compile with **mkl**.  
  First, activate **icl.exe** from the Startup menu like numpy. Then, run the next command to build.  
  ```sh
  > python setup.py config --compiler=intelemw --fcompiler=intelvem build_clib --compiler=intelemw --fcompiler=intelvem build_ext --compiler=intelemw --fcompiler=intelvem build
  ```  
- It generates **static linked binary**, so you don't have to include DLLs in the wheel.  
  You can just create a wheel and install like:  
  ```sh
  > python setup.py bdist_wheel
  > pip install dist\scipy-1.1.0-cp27-cp27m-win_amd64.whl
  ```



## Install other requirements by pip from remote repository

- Install **enum34**, **cryptography**, and **mock**.  
  If **enum** exists, uninstall it at first.  
  ```sh
  > pip uninstall enum
  > pip install enum34 cryptography mock
  ```
- **Keras** subpackages for tensorflow.  
  ```sh
  > pip install keras_applications==1.0.5 --no-deps
  > pip install keras_preprocessing==1.0.3 --no-deps
  ```
- These are additional requirements for **GPU version** of tensorflow.  
  ```sh
  > python -m pip install absl-py protobuf
  ```


## Build tensorflow

To build the tensorflow, you want to read [another markdown document](./README_Python27_HowToBuild.md).  


## Update keras

- keras is already installed I guess, but we need to update this due to the subpackages' version.  
  ```sh
  > pip install keras
  ```
- Now you can use **keras** with **tensorflow** in **Python 2.7** !
