# howto_fftw_apple_silicon

Note that it is possible to install fftw with brew. But, if you want to get best performance, fftw has to be compiled with SIMD instruction sets. 
Here is a how to install pyfftw and fftw on apple silicon computers (M1).

## Installing FFTW 

Download FFTW source code (version 3.3.9) and move the fftw-3-3-9-configure-diff.txt file from this repository to fftw source directory and configure:

```console
$ patch configure fftw-3-3-9-configure-diff.txt
$ ./configure --enable-threads --enable-neon --enable-armv8-cntvct-el0 --enable-float
$ make
$ sudo make install
$ msake clean
$ ./configure --enable-threads --enable-armv8-cntvct-el0 --enable-neon 
$ make
$ sudo make install
$ make clean
```

The patch file is needed so that we can compile with neon for double precision. The enable-armv8-cntvct-el0 allows fftw to use timers, which appear to be working OK because planning with FFTW_PATIENT does improve the calculation speed compared to FFTW_MEASURE. Threading appears to be working OK. On Mac Mini (2020 M1 8GB) I get:

```console
$ tests/bench -s c512x512
Problem: c512x512, setup: 299.56 ms, time: 1.13 ms, ``mflops'': 20906.478
$ tests/bench -onthreads=4 -s c512x512
Problem: c512x512, setup: 322.40 ms, time: 462.38 us, ``mflops'': 51025.596
$ tests/bench -onthreads=4 -opatient -s c512x512
Problem: c512x512, setup: 18.84 s, time: 349.69 us, ``mflops'': 67468.697
```

Compilation for long double is optional, but it appears that pyfftw by default needs it to operate flawlessly, so we install it. No optimizations here, because we wont be needing it anyway:

```console
$ ./configure --enable-threads --enable-armv8-cntvct-el0 --enable-long-double
$ make
$ sudo make install
$ make clean
```

## Installing pyFFTW

Download pyfftw source and install with

```console
$ CFLAGS="-Wno-implicit-function-declaration" python setup.py install
```

The setup.py script tries to detect how we compiled fftw library by compiling some auto-generated c program using some specific function calls. On apple, gcc is linked to clang, which prohibits compiling without function declarations. The compiler option above allows it to find the libraries and appears to compile into a working pyfftw package. On mini-forge python distribution with python 3.9 running natively:

```console
$ ipython
```

```python
>>> import pyfftw
>>> import numpy as np
>>> pyfftw.interfaces.cache.enable()
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a, planner_effort = "FFTW_PATIENT", workers = 1)
1.24 ms ± 2.24 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a,  planner_effort = "FFTW_PATIENT", workers = 4)
539 µs ± 53.1 µs per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

## TODO

Python multithreaded benchmarks are not convincing. Installing with openmp might speed up multi-threaded calculation in python. Compiling with apple's clang appears to be possible according to https://iscinumpy.gitlab.io/post/omp-on-high-sierra/ 

```console
$ brew install libomp
$ ./configure CPPFLAGS="-Xpreprocessor -fopenmp" LDFLAGS="-lomp" --enable-openmp --enable-neon --enable-armv8-cntvct-el0 --enable-float
$ make clean
$ make
$ sudo make install
```
Above appears to be compiling and installing correctly, but I could not figure out how to convince pyfftw to use openmp instead of pthreads.









