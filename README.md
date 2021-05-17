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
$ make clean
$ ./configure --enable-threads --enable-armv8-cntvct-el0 --enable-neon 
$ make
$ sudo make install
$ make clean
```

The patch file is needed so that we can compile with neon for double precision. The enable-armv8-cntvct-el0 allows fftw to use timers, which appear to be working OK because planning with FFTW_PATIENT does improve the calculation speed compared to FFTW_MEASURE. Threading appears to be working OK. On Mac Mini (2020 M1 8GB) I get (double precision):

```console
$ tests/bench -s c512x512
Problem: c512x512, setup: 299.56 ms, time: 1.13 ms, ``mflops'': 20906.478
$ tests/bench -onthreads=4 -s c512x512
Problem: c512x512, setup: 322.40 ms, time: 462.38 us, ``mflops'': 51025.596
$ tests/bench -onthreads=4 -opatient -s c512x512
Problem: c512x512, setup: 18.84 s, time: 349.69 us, ``mflops'': 67468.697
```

Compilation for long double is optional, but it appears that pyfftw by default needs it to operate flawlessly, so we install it. No optimizations here because we wont be needing it anyway:

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

Without the compiler option, the setup.py script tries to detect how we compiled fftw library by compiling some auto-generated c program and linking with a proper library. This fails because of the default -Wimplicit-function-declaration compiler option. On apple, gcc is linked to clang, which prohibits compiling without function declarations. There may be some missing header files resulting in the problem, but the above compiler option seems to solve the problem. I did not dig further into this. The compiler option above allows it to find the libraries and appears to compile into a working pyfftw package.

## Benchmarks

On miniforge python distribution with python 3.9 running natively on Mac Mini (2020 M1 8GB):

```console
$ ipython
```

```python
>>> import pyfftw
>>> import numpy as np
>>> a = np.random.randn(512,512) + 1j #complex double precision data and transform
>>> pyfftw.interfaces.cache.enable()
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a, planner_effort = "FFTW_PATIENT", workers = 1)
1.24 ms ± 2.24 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a,  planner_effort = "FFTW_PATIENT", workers = 4)
578 µs ± 4.58 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> a = np.asarray(a,"complex64") #complex single precision data and transform
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a, planner_effort = "FFTW_PATIENT", workers = 1)
667 µs ± 892 ns per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a,  planner_effort = "FFTW_PATIENT", workers = 4)
315 µs ± 974 ns per loop (mean ± std. dev. of 7 runs, 1000 loops each)
```

Except for a somewhat slower computation speed, compared to the tests/bench results, it appears to be working OK. For comparison, anaconda running python 3.8 over roseta on Mac Mini (2020 M1 8GB) with intel's mkl_fft I get

```console
$ ipython
```

```python
>>> import mkl_fft, mkl
>>> import numpy as np
>>> a = np.random.randn(512,512) + 1j #complex128 data
>>> mkl.set_num_threads(1) #defaults to 8, better to use max 4 threads because we only have 4 high performance threads.
8
>>> timeit mkl_fft.fft2(a) #single core
1.8 ms ± 4.18 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> mkl.set_num_threads(4)
1
>>> timeit mkl_fft.fft2(a) #multi-core
365 µs ± 6.91 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> a = np.asarray(a,"complex64") # single precision
>>> mkl.set_num_threads(1)
4
>>> timeit mkl_fft.fft2(a) #single core single precision
665 µs ± 373 ns per loop (mean ± std. dev. of 7 runs, 1000 loops each)
>>> mkl.set_num_threads(4)
1
>>> timeit mkl_fft.fft2(a) #multi-core single precision
222 µs ± 34.4 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
```
So, single-core performance of fftw seems to be good, but Intels mkl_fft runs better multi-threaded.

## TODO - Install with openmp

Pyfftw multithreaded benchmarks on are not convincing. Installing with openmp might speed up multi-threaded calculation in python. Compiling with apple's clang appears to be possible according to https://iscinumpy.gitlab.io/post/omp-on-high-sierra/ 

```console
$ brew install libomp
$ ./configure CPPFLAGS="-Xpreprocessor -fopenmp" LDFLAGS="-lomp" --enable-openmp --enable-neon --enable-armv8-cntvct-el0 --enable-float
$ make clean
$ make
$ sudo make install
```
Above appears to be compiling and installing correctly, but I could not figure out how to convince pyfftw to use openmp instead of pthreads.









