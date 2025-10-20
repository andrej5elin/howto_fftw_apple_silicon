# howto_fftw_apple_silicon

These are instructions how to install fftw and pyfftw on apple silicon computers. Note that it is possible to install fftw with brew. It is also possible to pip install pyfftw. But, if you want to get the best performance, fftw has to be compiled with SIMD instruction sets and, more importantly, using --enable-armv8-cntvct-el0 option, which the packages shipped with brew and pip are not (as of today - October 2025). 

Tested on **M1** (2020 Mac mini with 8-core CPU) and **M4 Pro** (2024 Macbook Pro with 14‑core CPU).

## Installing FFTW 

Download FFTW source code (tested using version 3.3.10). If you want to apply NEON optimization for doubple precision, you must copy fftw-3-3-10-configure-diff.txt file from this repository to fftw source directory and patch:

```sh
curl -L https://www.fftw.org/fftw-3.3.10.tar.gz | tar xvf -
cd fftw-3.3.10
curl -LO https://raw.githubusercontent.com/andrej5elin/howto_fftw_apple_silicon/refs/heads/main/fftw-3-3-10-configure-diff.txt | patch configure -
```
I did not find a better way other than patching. Scroll down to section **Finding optimal compilation settings** for details. 

### Using pthreads
Note that we also compile long double (we need that for pyffftw)
```sh
./configure --enable-armv8-cntvct-el0 --enable-threads --enable-long-double 
make clean
make
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-threads
make clean
make
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-threads --enable-float
make clean
make
sudo make install
```
Below we test 2D FFT (complex and real) using 1, 4 and 8 threads with patient planning on M4 pro and M1. 

#### M4 Pro threads benchmarks (single precision)
```sh
tests/bench -opatient -onthreads=1 c512x512
tests/bench -opatient -onthreads=4 c512x512
tests/bench -opatient -onthreads=8 c512x512
tests/bench -opatient -onthreads=1 r512x512
tests/bench -opatient -onthreads=4 r512x512
tests/bench -opatient -onthreads=8 r512x512
```
```console
Problem: c512x512, setup: 1.06 s, time: 374.88 us, ``mflops'': 62935.539
Problem: c512x512, setup: 4.56 s, time: 135.15 us, ``mflops'': 174570.72
Problem: c512x512, setup: 6.05 s, time: 123.25 us, ``mflops'': 191423.61
Problem: r512x512, setup: 483.02 ms, time: 214.41 us, ``mflops'': 55019.292
Problem: r512x512, setup: 3.12 s, time: 88.27 us, ``mflops'': 133635.67
Problem: r512x512, setup: 4.45 s, time: 105.38 us, ``mflops'': 111939.32
```
So, the threading works, but it is not efficient for the given problem size (512x512). No significant improvement beyond -onthreads=4, even though we have 10 performance cores.

#### M1 threads benchmarks (single precision)
```sh
tests/bench -opatient -onthreads=1 c512x512
tests/bench -opatient -onthreads=4 c512x512
tests/bench -opatient -onthreads=8 c512x512
tests/bench -opatient -onthreads=1 r512x512
tests/bench -opatient -onthreads=4 r512x512
tests/bench -opatient -onthreads=8 r512x512
```
```console
Problem: c512x512, setup: 3.62 s, time: 562.25 us, ``mflops'': 41961.69
Problem: c512x512, setup: 16.16 s, time: 233.50 us, ``mflops'': 101040.51
Problem: c512x512, setup: 20.36 s, time: 218.78 us, ``mflops'': 107838.13
Problem: r512x512, setup: 2.64 s, time: 304.31 us, ``mflops'': 38764.362
Problem: r512x512, setup: 13.25 s, time: 136.19 us, ``mflops'': 86619.403
Problem: r512x512, setup: 17.07 s, time: 151.23 us, ``mflops'': 78005.344
```
Note that when running the computation on 8 cores on M1 it activates efficiency cores. 

## Installing FFTW with openmp
Installing with openmp should speed up multi-threaded calculation. Compiling with apple's clang is possible, see https://iscinumpy.gitlab.io/post/omp-on-high-sierra/ Ideally, we should be able to first install libomp using brew using:
```sh
brew install libomp
export CPPFLAGS="-I$(brew --prefix libomp)/include -Xpreprocessor -fopenmp"
export LDFLAGS="-L$(brew --prefix libomp)/lib -lomp"
```
Although this approach worked back in 2020 when the first M1 came out, today the version shipped with brew does not work optimally on any of the apple silicon computers I own (M1 and M4 Pro). Scroll down to **the openmp issue** to learn about this. If you want the best threading performance you must install the version from the 2020/2021 era. I found that the last version that we can find in brew, which gives good results is 14.0.6, while from version 15 onwards, I see performance decrease on M1 and M4 pro. I followed https://blog.sandipb.net/2021/09/02/installing-a-specific-version-of-a-homebrew-formula/ There may be other ways, but I have settled with the following recipe:
```sh
brew tap-new $USER/libomp
brew tap homebrew/core --force
brew extract --version=14.0.6 libomp $USER/libomp
HOMEBREW_NO_AUTO_UPDATE=1 brew install $USER/libomp/libomp@14.0.6
export CPPFLAGS="-I$(brew --prefix libomp@14.0.6)/include -Xpreprocessor -fopenmp"
export LDFLAGS="-L$(brew --prefix libomp@14.0.6)/lib -lomp"
```
Now we are able to compile with the old libomp version. I found it easier to compile pyfftw with shared fftw library, so we use *enable-shared* option:
```sh
./configure --enable-armv8-cntvct-el0 --enable-openmp --enable-shared --enable-long-double 
make clean
make
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-openmp --enable-shared 
make clean
make
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-openmp --enable-shared --enable-float
make clean
make
sudo make install
```
#### M4 Pro openmp benchmarks (single precision)
```sh
tests/bench -opatient -onthreads=1 c512x512
tests/bench -opatient -onthreads=4 c512x512
tests/bench -opatient -onthreads=8 c512x512
tests/bench -opatient -onthreads=1 r512x512
tests/bench -opatient -onthreads=4 r512x512
tests/bench -opatient -onthreads=8 r512x512
```
```console
Problem: c512x512, setup: 1.08 s, time: 372.47 us, ``mflops'': 63342.119
Problem: c512x512, setup: 3.54 s, time: 117.63 us, ``mflops'': 200564.45
Problem: c512x512, setup: 3.91 s, time: 74.95 us, ``mflops'': 314802.34
Problem: r512x512, setup: 503.24 ms, time: 224.55 us, ``mflops'': 52534.599
Problem: r512x512, setup: 1.96 s, time: 61.27 us, ``mflops'': 192546.47
Problem: r512x512, setup: 2.42 s, time: 45.09 us, ``mflops'': 261599
```
Openmp does improve the computation speed of multithreaded runs. We get a noticable improvement; M4 Pro's 8-core results are 2x better than pthread's version.
#### M1 openmp benchmarks (single precision)
```sh
tests/bench -opatient -onthreads=1 c512x512
tests/bench -opatient -onthreads=4 c512x512
tests/bench -opatient -onthreads=8 c512x512
tests/bench -opatient -onthreads=1 r512x512
tests/bench -opatient -onthreads=4 r512x512
tests/bench -opatient -onthreads=8 r512x512
```
```console
Problem: c512x512, setup: 3.62 s, time: 556.47 us, ``mflops'': 42397.637
Problem: c512x512, setup: 16.73 s, time: 157.09 us, ``mflops'': 150183.95
Problem: c512x512, setup: 21.94 s, time: 313.09 us, ``mflops'': 75354.299
Problem: r512x512, setup: 2.64 s, time: 304.47 us, ``mflops'': 38744.469
Problem: r512x512, setup: 13.53 s, time: 85.52 us, ``mflops'': 137945.32
Problem: r512x512, setup: 17.84 s, time: 192.70 us, ``mflops'': 61215.821
```
These results are interesting. When running with four threads (which equals the number of performance cores on M1) we get notable improvement. When running with 8 threads, the efficiency cores are utilized, and we observe a strong slowdown. Here, the openmp results are worse compared to pthreads results. With pthreads, OS balances work better between efficiency and performance cores. See additional comments in the **The openmp issue** section below. Main takeaway message: 

> **When using openp set OMP_NUM_THREADS <= number of performance cores**

## Installing pyFFTW

We first create and prepare a new environment using conda. I have tested this using python version 3.13.

```sh
conda create --name fftw python=3.13
conda activate fftw
conda install cython llvm-openmp numpy scipy ipython 
```

If you start with a fresh console, there will be no LDFLAGS and CPPFLAGS set... but I assume we still have these from previous compiles, so unset these. We will use openmp runtime libs installed by conda instead of brew. 

```sh
unset LDFLAGS
unset CPPFLAGS
```
Next, download and extract pyfftw (version 0.15.0) and apply the patch and install
```sh
curl -L https://github.com/pyFFTW/pyFFTW/archive/refs/tags/v0.15.0.tar.gz | tar xvf -
cd pyFFTW-0.15.0
curl -LO https://raw.githubusercontent.com/andrej5elin/howto_fftw_apple_silicon/refs/heads/main/pyfftw-0-15-0-setup-diff.txt
patch setup.py pyfftw-0-15-0-setup-diff.txt
pip install -e . -v
```
If pyfftw finds omp version of the fftw libraries it should install that version and the install script prints this out. Look for: 
```console
  INFO:__main__:Build pyFFTW with support for FFTW with
  INFO:__main__:double precision + openMP
  INFO:__main__:single precision + openMP
  INFO:__main__:long precision + openMP
```
If you want to use or test pthreads instead of openmp, do
```sh
PYFFTW_USE_PTHREADS=1 pip install -e . -v
```
```console
  INFO:__main__:Build pyFFTW with support for FFTW with
  INFO:__main__:double precision + pthreads
  INFO:__main__:single precision + pthreads
  INFO:__main__:long precision + pthreads
```

## pyFFTW benchmarks

```sh
ipython
```
First, we must set the OMP_NUM_THREADS environment variable, otherwise pyfftw takes all available cores for planning. We do not want our code to be running on
efficiency cores while performing plans. Setting this variable to the number of performance cores does the trick. 

```python
>>> import os
>>> os.environ["OMP_NUM_THREADS"] = "10"
```
It is important that the environment variable is set prior to importing pyfftw. Now we can plan the transforms:
```python
>>> import pyfftw
>>> import numpy as np
>>> pyfftw.config.PLANNER_EFFORT = 'FFTW_PATIENT'
>>> c = pyfftw.empty_aligned((512,512), dtype='complex64')
>>> r = pyfftw.empty_aligned((512,512), dtype='float32')
>>> c512x512_1 = pyfftw.builders.fft2(c, threads = 1)
>>> c512x512_4 = pyfftw.builders.fft2(c, threads = 4)
>>> c512x512_8 = pyfftw.builders.fft2(c, threads = 8)
>>> r512x512_1 = pyfftw.builders.rfft2(r, threads = 1)
>>> r512x512_4 = pyfftw.builders.rfft2(r, threads = 4)
>>> r512x512_8 = pyfftw.builders.rfft2(r, threads = 8)
```
timeit tests reveal identical performance as test/bench (M4 pro):
```python
>>> timeit c512x512_1(c)
393 μs ± 6.01 μs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit c512x512_4(c)
129 μs ± 2.3 μs per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
>>> timeit c512x512_8(c)
70 μs ± 345 ns per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
>>> timeit r512x512_1(r)
228 μs ± 726 ns per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit r512x512_4(r)
77.6 μs ± 5.17 μs per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
>>> timeit r512x512_8(r)
45.4 μs ± 686 ns per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
```
Nice, but was it worth it? What if we just use whatever conda gives in scipy

```python
>>> import scipy as sp
>>> import numpy as np
>>> c = np.empty((512,512), dtype='complex64')
>>> r = np.empty((512,512), dtype='float32')
>>> timeit sp.fft.fft2(c, workers = 1, overwrite_x = True)
821 μs ± 2.26 μs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit sp.fft.fft2(c, workers = 4, overwrite_x = True)
282 μs ± 480 ns per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit sp.fft.fft2(c, workers = 8, overwrite_x = True)
186 μs ± 158 ns per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
>>> timeit sp.fft.rfft2(r, workers = 1, overwrite_x = True)
284 μs ± 516 ns per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
>>> timeit sp.fft.rfft2(r, workers = 4, overwrite_x = True)
110 μs ± 313 ns per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
>>> timeit sp.fft.rfft2(r, workers = 8, overwrite_x = True)
106 μs ± 560 ns per loop (mean ± std. dev. of 7 runs, 10,000 loops each)
```
Ok, so the real transforms are actually quite good in scipy, but complex transforms are more than two times slower in scipy. Also, here the comparison is not fair because we use overwrite_x = True which implies an in-place transform, whereas in pyfftw we used out-of-place transform. In-place transforms in pyfftw are actually slightly slower. If you really need to increase FFT computation speed in python, then a factor 2x speedup (compared to scipy) is possible using pyfftw. However, to really benefit anything, you need to apply all the tricks explained here.

### Finding optimal fftw configuration

Here I document my testing of fftw under different compilation settings to get best single-core performance. The benchmarks below are for the M4 Pro. The first issue arises because fftw cannot find cycle counter when running configuration:
```sh
./configure
make clean
make
tests/bench c512x512
tests/bench r512x512
```
```console
Problem: c512x512, setup: 6.83 ms, time: 5.39 ms, ``mflops'': 4376.7665
Problem: r512x512, setup: 2.18 ms, time: 497.62 us, ``mflops'': 23705.561
```
The results are strange. The real transform appears to work well, while for the complex input, it is way too slow. The configure script actualy print a warning, saying the code won't work well:
```console
***************************************************************
WARNING: No cycle counter found.  FFTW will use ESTIMATE mode  
         for all plans.  See the manual for more information.
***************************************************************
```
so the issue may be in bad FFT planning for complex transform. To help find cycle counter, we must explixitly specify it as
```sh
./configure --enable-armv8-cntvct-el0
make clean
make
tests/bench -opatient c512x512
tests/bench -opatient r512x512
```
```console
Problem: c512x512, setup: 959.19 ms, time: 1.01 ms, ``mflops'': 23311.762
Problem: r512x512, setup: 491.46 ms, time: 475.41 us, ``mflops'': 24813.473
```
Problem solved. However, for single precision I get:
```sh
./configure --enable-armv8-cntvct-el0 --enable-float
make clean
make
tests/bench -opatient c512x512
tests/bench -opatient r512x512
```
```console
Problem: c512x512, setup: 936.43 ms, time: 944.75 us, ``mflops'': 24972.702
Problem: r512x512, setup: 489.33 ms, time: 454.84 us, ``mflops'': 25935.236
```
So, not much faster than double precision, indicating we need to apply more optimization. For floats we are allowed to enable neon as:
```sh
./configure --enable-armv8-cntvct-el0 --enable-float --enable-neon
make clean
make
tests/bench -opatient c512x512
tests/bench -opatient r512x512
```
```console
Problem: c512x512, setup: 1.08 s, time: 381.50 us, ``mflops'': 61842.621
Problem: r512x512, setup: 501.09 ms, time: 212.67 us, ``mflops'': 55467.983
```
Ok, this appears to be fine now. We can also enable neon to double precison, however, we must hint fftw that we are on aarch64, otherwise it complains. Soo we add --host=aarch64
```sh
./configure --enable-armv8-cntvct-el0 --enable-neon --host=aarch64
make clean
make
tests/bench -opatient c512x512
tests/bench -opatient r512x512
```
```console
Problem: c512x512, setup: 1.61 s, time: 779.44 us, ``mflops'': 30269.213
Problem: r512x512, setup: 590.10 ms, time: 335.38 us, ``mflops'': 35173.999
```
and we see a slight imrovement. The results are consistent with the single precision results, being about 2x slower than single precision tests. Have all possible optimization been applied? I am not an expert and I don't know. NEON is 128bit, so working with double precision and complex transform already consumes 128bit, so we cannot expect much improvement in double precision... 

Later I found that I cannot create shared libraries if using --host=aarch64, that is why I patched the config. I am sure this couls be resolved with some special compile arguments, but I just did not find a better solution other than patching.

### The openmp thread utilization issue

Ok let us test the current version of libomp (21.1.3 as of October 2025) we get:

```sh
brew install libomp
export CPPFLAGS="-I$(brew --prefix libomp)/include -Xpreprocessor -fopenmp"
export LDFLAGS="-L$(brew --prefix libomp)/lib -lomp"
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-openmp --enable-float
make clean
make
tests/bench -opatient -onthreads=8 c512x512
```
```console
Problem: c512x512, setup: 10.20 s, time: 136.80 us, ``mflops'': 172457.25
```
What? This is worse than pthreads. Repeating the tests with OMP_WAIT_POLICY=ACTIVE reveals the problem
```sh
OMP_WAIT_POLICY=ACTIVE tests/bench -opatient -onthreads=8 c512x512
```
```console
Problem: c512x512, setup: 3.91 s, time: 74.95 us, ``mflops'': 314802.34
```
When observing thread utilization using with apple's activity monitor, I can see poor utilization when OMP_WAIT_POLICY=PASSIVE (default value), and almost half of the processing time appears to be wasted for some reason, so it makes sense that computation time is almost two times longer compared to OMP_WAIT_POLICY=ACTIVE. Using version libomop 14.0.6 (and prior to this version) gives a similar result with both OMP_WAIT_POLICY=PASSIVE and OMP_WAIT_POLICY=ACTIVE. I cannot use active omp wait policy because I am using python and pyfftw. Using OMP_WAIT_POLICY=ACTIVE inside python just spins all cores at 100%, making the environment useless. Therefore, I choose libomop 14.0.6 for my work...





