# howto_fftw_apple_silicon

This is a how to install fftw and pyfftw on apple silicon computers. Note that it is possible to install fftw with brew. It is also posible to pip install pyfftw, but, if you want to get the best performance, fftw has to be compiled with SIMD instruction sets, which the packages shipped with brew and pip are not (as of today - October 2025)

Here is how to install pyfftw and fftw on apple silicon computers (tested on M1 and M4 Pro) using pthreads and/or openmp as the threading library.

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

Computation is now faster, I get (M1)

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

## Installing FFTW with openmp

Installing with openmp in should speeds up multi-threaded calculation (but there are ceveats on Pro and Max machines). Compiling with apple's clang is possbile, see https://iscinumpy.gitlab.io/post/omp-on-high-sierra/ 

I am using --enable-shared option because we need it for pyfftw later on. 

```console
$ brew install libomp
$ export CPPFLAGS="-I/opt/homebrew/opt/libomp/include -Xpreprocessor -fopenmp"
$ export LDFLAGS="-L/opt/homebrew/opt/libomp/lib -lomp"
$ ./configure --enable-openmp --enable-armv8-cntvct-el0 --enable-long-double --enable-shared
$ make clean
$ make
$ sudo make install
$ ./configure --enable-openmp --enable-neon --enable-armv8-cntvct-el0 --enable-shared
$ make clean
$ make
$ sudo make install
$ ./configure --enable-openmp --enable-neon --enable-armv8-cntvct-el0 --enable-float --enable-shared
$ make clean
$ make
$ sudo make install
```
We now get for single precision speed tests on M1

```console
$ tests/bench -onthreads=1 -opatient c512x512
Problem: c512x512, setup: 3.66 s, time: 557.16 us, ``mflops'': 42345.321
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 17.07 s, time: 161.14 us, ``mflops'': 146412.24
```
Therefore, openmp does improve the computation speed of multithreaded runs. We get a noticable improvement; 160 us versus 210 us for patient planning.

For the M4 Pro (14 core variant) on the other hand (and most likely any M Pro or Max variant) one needs to first set OMP_WAIT_POLICY=ACTIVE, which allows openmp to fully utilize all cores. On M4 pro I get for the single precision tests:

```console
$ export OMP_WAIT_POLICY=ACTIVE
$ tests/bench -onthreads=1 -opatient c512x512
Problem: c512x512, setup: 1.05 s, time: 385.47 us, ``mflops'': 61205.895
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 3.46 s, time: 111.27 us, ``mflops'': 212026.88
$ tests/bench -onthreads=8 -opatient c512x512
Problem: c512x512, setup: 3.92 s, time: 76.96 us, ``mflops'': 306557.6
$ tests/bench -onthreads=10 -opatient c512x512
Problem: c512x512, setup: 4.94 s, time: 73.88 us, ``mflops'': 319363.25
```

Single core performance of M4 is about 40% faster than M1 in these tests. The computation is about 3x faster when using four cores, both on M1 and on M4 Pro. Unfortunately, openmp does not play well using the default policy OMP_WAIT_POLICY=PASSIVE. Here, I notice a noticable performance decrease when using 4 cores, and a dramatic performance decrease when using 8 cores:

```console
$ export OMP_WAIT_POLICY=PASSIVE
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 5.43 s, time: 138.53 us, ``mflops'': 170307.85
$ tests/bench -onthreads=8 -opatient c512x512
Problem: c512x512, setup: 10.13 s, time: 163.28 us, ``mflops'': 144492.77
```

Likeley this is caused by different thread handling on the OS level in Pro macs. See [inside m4 chips cpu core management](https://eclecticlight.co/2024/12/05/inside-m4-chips-cpu-core-management/). My guess is that on the non-pro machines with a simplified core management (having only one cluster of performance cores) libomp handles it well, whereas on Pros with a complicated thread mobility it makes libomp inefficient with poor thread utilization. The workaraount to force the threads to stay active using OMP_WAIT_POLICY=ACTIVE in order to achieve good thread utilization is unfortunately not a valid option when using pyfftw. So, I recommend using pthreads instead of openmp if you want to use pyfftw in M Pro and M Max orocessosrs. I will neverhteles document how to compile pyfftw with openmp. Using pthreads instead of openmp you should see

```console
$ tests/bench -onthreads=4 -opatient c512x512
Problem: c512x512, setup: 4.66 s, time: 136.15 us, ``mflops'': 173288.51
$ tests/bench -onthreads=8 -opatient c512x512
Problem: c512x512, setup: 6.12 s, time: 127.56 us, ``mflops'': 184952.16
```

So, not optimal, but at least we do not see a performance decrease when going from 4 to 8 cores. It is still a bummer we cannot achieve the full performance on M4 Pro without the OMP_WAIT_POLICY=ACTIVE and the pthreads 8-core results (127 us) are more than 50% slower than what the machine is capable of (77 us). Hopefully, the thread utilization issue will be resolved in future versions of openmp. 


## Installing pyFFTW

We first create and prepare a new environment using conda. I have tested this using python version 3.14.

```console
$ conda create --name fftw3 python=3.14
$ conda activate fftw3
$ pip install numpy scipy cython
```
I use pip instead of conda install because numpy comes with libomp and we mess up the environment when installing our custom build pyfftw compiled with a different libomp shipped with brew. I did not find a good solution for this, so for demonstration purposes, we use pip, which gives us unoptimized scipy and numpy packages.

Next, download and extract pyfftw (version 0.15.0), apply the patch and install

```console
$ patch setup.py pyfftw-0-15-0-setup-diff.txt
$ ln -s /opt/homebrew/opt/libomp/lib/*.* /usr/local/lib/.
$ ln -s /opt/homebrew/opt/libomp/include/*.* /usr/local/include/.
```
pyfftw tries to compile some test code to determine which fftw libraries are present. The patch replaces the linker option "-fopenmp" with "-lomp" and then it is possible to compile with opemp. I could not find a better solution, but symbolic linking homebrew-installed libomp allows pyfftw compile correctly. Now we can compile and install pyfftw

```console
$ pip install -e . -v
```

If you want to use pthreads instead of openmp, do

```console
$ export PYFFTW_USE_PTHREADS=1
$ pip install -e . -v
```

## pyFFTW benchmarks

On miniconda python distribution with python 3.14 running natively on Mac Mini (2020 M1 8GB):

```console
$ ipython
```

First, I set the OMP_NUM_THREADS environment variable, otherwise pyfftw takes 8 cores for planning. We do not want our code to be running on
efficiency cores. Setting this variable to the number of performance cores of M1 does the trick. There may be some other ways to force pyfftw to 
work properly, but for now, I settle with:

```python
>>> import os
>>> os.environ["OMP_NUM_THREADS"] = "4"
```

It is important that the enviroment variable is set prior to importing pyfftw. Now we can do benchamerks

```python
>>> import pyfftw
>>> import numpy as np
>>> a = np.random.randn(512,512) + 1j #complex double precision data and transform
>>> pyfftw.interfaces.cache.enable()
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a, planner_effort = "FFTW_PATIENT", workers = 1)
1.24 ms ± 2.59 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a,  planner_effort = "FFTW_PATIENT", workers = 4)
439 µs ± 10.8 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> a = np.asarray(a,"complex64") #complex single precision data and transform
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a, planner_effort = "FFTW_PATIENT", workers = 1)
669 µs ± 3.18 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit pyfftw.interfaces.scipy_fft.fft2(a,  planner_effort = "FFTW_PATIENT", workers = 4)
221 µs ± 15.4 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
```

Except for a somewhat slower computation speed (probably caused by python overhead), compared to the tests/bench results, it appears to be working OK. For comparison, anaconda running python 3.8 over roseta on Mac Mini (2020 M1 8GB) with intel's mkl_fft I get

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





