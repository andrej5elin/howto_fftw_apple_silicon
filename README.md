# howto_fftw_apple_silicon

Note that it is possible to install fftw with brew. But, if you want to get best performance, fftw has to be compiled with SIMD instruction sets. 
Here is a how to install pyfftw and fftw on apple silicon computers (M1).

## Installing FFTW 

Download FFTW source code (version 3.3.10) and move the fftw-3-3-10-configure-diff.txt file from this repository to fftw source directory and configure:

```console
$ patch configure fftw-3-3-10-configure-diff.txt
$ ./configure --enable-threads --enable-neon --enable-armv8-cntvct-el0
$ make
$ sudo make install
```

The patch file is needed so that we can compile with neon for double precision. The enable-armv8-cntvct-el0 allows fftw to use timers, which appear to be working OK because planning with FFTW_PATIENT does improve the calculation speed compared to FFTW_MEASURE. Threading appears to be working OK. On Mac Mini (2020 M1 8GB) I get (double precision):

```console
$ tests/bench c512x512
Problem: c512x512, setup: 295.95 ms, time: 1.11 ms, ``mflops'': 21179.788
$ tests/bench -onthreads=4 -s c512x512
Problem: c512x512, setup: 322.60 ms, time: 446.69 us, ``mflops'': 52817.596
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 17.97 s, time: 353.28 us, ``mflops'': 66782.372
```

Now we compile/install for single precision

```console
$ ./configure --enable-threads --enable-neon --enable-armv8-cntvct-el0 --enable-float
$ make clean
$ make
$ sudo make install
```

Computation is now faster, I get

```console
$ tests/bench c512x512
Problem: c512x512, setup: 265.03 ms, time: 697.94 us, ``mflops'': 33803.829
$ tests/bench -onthreads=4 -s c512x512
Problem: c512x512, setup: 289.50 ms, time: 282.66 us, ``mflops'': 83468.736
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 16.35 s, time: 210.22 us, ``mflops'': 112230.52
```

Compilation for long double is optional, but it appears that pyfftw by default needs it to operate flawlessly, so we install it. No optimizations here because we wont be needing it anyway:

```console
$ ./configure --enable-threads --enable-armv8-cntvct-el0 --enable-long-double
$ make clean
$ make
$ sudo make install
```

## Installing pyFFTW

Download pyfftw source and install with

```console
$ CFLAGS="-Wno-implicit-function-declaration" python setup.py install
```

Without the compiler option, the setup.py script tries to detect how we compiled fftw library by compiling some auto-generated c program and linking with a proper library. This fails because of the default -Wimplicit-function-declaration compiler option. On apple, gcc is linked to clang, which prohibits compiling without function declarations. There may be some missing header files resulting in the problem, but the above compiler option seems to solve the problem. I did not dig further into this.

## Benchmarks

On miniconda python distribution with python 3.10 running natively on Mac Mini (2020 M1 8GB):

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

## Install with openmp

Pyfftw multithreaded benchmarks on are not convincing. Installing with openmp might speed up multi-threaded calculation in python. Compiling with apple's clang appears to be possible according to https://iscinumpy.gitlab.io/post/omp-on-high-sierra/ 

```console
$ brew install libomp
$ ./configure CPPFLAGS="-Xpreprocessor -fopenmp" LDFLAGS="-lomp" --enable-openmp --enable-armv8-cntvct-el0 --enable-long-double
$ make clean
$ make
$ sudo make install
$ ./configure CPPFLAGS="-Xpreprocessor -fopenmp" LDFLAGS="-lomp" --enable-openmp --enable-neon --enable-armv8-cntvct-el0 
$ make clean
$ make
$ sudo make install
$ ./configure CPPFLAGS="-Xpreprocessor -fopenmp" LDFLAGS="-lomp" --enable-openmp --enable-neon --enable-armv8-cntvct-el0 --enable-float
$ make clean
$ make
$ sudo make install
```
Above appears to be compiling and installing correctly. We now get for single precision speed tests

```console
$ tests/bench c512x512
Problem: c512x512, setup: 264.07 ms, time: 738.38 us, ``mflops'': 31952.544
$ tests/bench -onthreads=4 -s c512x512
Problem: c512x512, setup: 293.12 ms, time: 223.69 us, ``mflops'': 105472.86
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 17.07 s, time: 161.14 us, ``mflops'': 146412.24
```
Therefore, openmp does improve the computation speed of multithreaded runs. Installing pyfftw 0.13 also works. The setup script finds the openmp version of FFTW and installs it. We get









