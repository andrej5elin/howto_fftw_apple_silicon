# howto_fftw_apple_silicon

This is a how to install fftw and pyfftw on apple silicon computers. Note that it is possible to install fftw with brew. It is also posible to pip install pyfftw, but, if you want to get the best performance, fftw has to be compiled with SIMD instruction sets and, more importantly, using --enable-armv8-cntvct-el0 option, which the packages shipped with brew and pip are not (as of today - October 2025). 

Here is how to install pyfftw and fftw on apple silicon computers (tested on **M1** (2020 Mac mini with 8-core CPU) and **M4 Pro** (2024 Macbook Pro with 14‑core CPU) using pthreads and/or openmp as the threading library.

## Installing FFTW 

Download FFTW source code (tested using version 3.3.10).  If you want to apply NEON optimization for doubple precision, you must copy fftw-3-3-10-configure-diff.txt file from this repository to fftw source directory and patch:

```console
$ patch configure fftw-3-3-10-configure-diff.txt
```
The patch file is needed so that we can compile with neon for double precision. I did not find a better way of doing it. Scroll down to section **Finding optimal compialtion settings** for details. Below are instructions to compile fftw on apple silicon using pthreads and openmp. 

### Using pthreads
Note that we install also --enable-long-double. This is because pyfftw needs it to operate without problems.
```sh
./configure --enable-armv8-cntvct-el0 --enable-threads --enable-long-double 
make clean
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-threads
make clean
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-threads --enable-float
make clean
sudo make install
```
We test 2D FFT (complex and real) using 1, 4 and 8 threads with patient planning. 

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
The above results show that threading works, but it is not efficient for the given problem size (512x512), with no significant improvement beyond OMP_NUM_THREADS=4, even though we have 10 performance cores, and we are not under full power with OMP_NUM_THREADS=8.

#### M1 threads benchmarks (single precision)
We repeat the same tests on M1
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
Note that when runnin the computation on 8 cores on M1 it activates efficiency cores. Still, OS appears to balance the threads well between efficiency and performance cores because we can see some improvement when going from four to eight cores, provided that the computation problem is big enogh (see the c512x512 test results).

## Installing FFTW with openmp

Installing with openmp in should speed up multi-threaded calculation. Compiling with apple's clang is possbile, see https://iscinumpy.gitlab.io/post/omp-on-high-sierra/ Ideally, we should be able to first install libomp using brew using:
```sh
brew install libomp
export CPPFLAGS="-I/opt/homebrew/opt/libomp/include -Xpreprocessor -fopenmp"
export LDFLAGS="-L/opt/homebrew/opt/libomp/lib -lomp"
```
This approach worked back in 2020 when the first M1 came out, but currently, the version of libomp shipped with brew does not work optimally on any of the apple silicon computers I own (M1 and M4 Pro). Scroll down to learn about this. If you want the best threading performance you must install the version from the 2020/2021 era. I found that the last version that we can find in brew, which gives good results is 14.0.6, while from a version 15 onwards, I see performance decrease on M1 and M4 pro. It was a bit of a chalange to figure out how to install specific libomp version using brew. I followed https://blog.sandipb.net/2021/09/02/installing-a-specific-version-of-a-homebrew-formula/ There may be other ways to install, but I have settled with the following recipe:
```sh
brew tap-new $USER/libomp
brew tap homebrew/core --force
brew extract --version=14.0.6 libomp $USER/libomp
HOMEBREW_NO_AUTO_UPDATE=1 brew install $USER/libomp/libomp@14.0.6
export CPPFLAGS="-I/opt/homebrew/opt/libomp@14.0.6/include -Xpreprocessor -fopenmp"
export LDFLAGS="-L/opt/homebrew/opt/libomp@14.0.6/lib -lomp"
```
Now we should be able to compile with this old libomp version. I found it easier to compile pyfftw with shared fftw library, so we use *enable-shared* option. To compile and install do
```sh
./configure --enable-armv8-cntvct-el0 --enable-openmp --enable-shared --enable-long-double 
make clean
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-openmp --enable-shared 
make clean
sudo make install
./configure --enable-armv8-cntvct-el0 --enable-neon --enable-openmp --enable-shared --enable-float
make clean
sudo make install
```
Again, we perform tests to compare with the pthreads version
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
Therefore, openmp does improve the computation speed of multithreaded runs. We get a noticable improvement; M4 Pro's 8-core results are 2x better than pthread's version, so it was worth the hasle.
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

We first create and prepare a new environment using conda. I have tested this using python version 3.14.

```sh
conda create --name fftw3 python=3.14
conda activate fftw3
pip install numpy scipy cython ipython
```
I use pip instead of conda install because numpy comes with libomp and we mess up the environment when installing our custom build pyfftw compiled with a different libomp shipped with brew. I did not find a good solution for this, so for demonstration purposes, we use pip, which gives us unoptimized scipy and numpy packages.

Next, download and extract pyfftw (version 0.15.0), apply the patch

```sh
patch setup.py pyfftw-0-15-0-setup-diff.txt
```
pyfftw tries to compile some test code to determine which fftw libraries are present. The patch replaces the linker option "-fopenmp" with "-lomp" and then it is possible to compile with opemp. Also, we need to copy or link our libraries as:
```sh
ln -s /opt/homebrew/opt/libomp@14.0.6/lib/*.* /usr/local/lib/.
ln -s /opt/homebrew/opt/libomp@14.0.6/include/*.* /usr/local/include/.
```
I did not find a better solution to this, as $export LDFLAGS="-L/opt/homebrew/opt/libomp@14.0.6/lib -lomp" did not work. Currently, pyfftw ignores these kind of environment variables. Now install with:
```sh
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

On miniconda python distribution with python 3.14 running natively on Mac Mini (2020 M1 8GB):

```sh
ipython
```

First, I set the OMP_NUM_THREADS environment variable, otherwise pyfftw takes all available cores for planning. We do not want our code to be running on
efficiency cores when performing plans or later, when computing. Setting this variable to the number of performance cores does the trick. 

```python
>>> import os
>>> os.environ["OMP_NUM_THREADS"] = "10"
```
It is important that the enviroment variable is set prior to importing pyfftw. Now we can plan the transforms:
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
timeit tests reveal identical performance as test/bench:
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
Nice, but was it worth it? What if we just install the best option conda can provide. We make a test environment. I thought installing accelerate may accelerate things
```sh
conda create --name accelerate
conda activate accelerate
conda install accelerate
conda install scipy numpy ipython
ipython
```
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
Ok, so the real transforms are actually quite good in scipy, but complex transforms are more than two times slower in scipy. Also, here the comparison is not completeley fair because we use overwrite_x = True which implies an in-place transform, whereas in pyfftw we used out-of-place transform, so fftw could be even faster.

Does it matter? Not really.. Just use scipy when developing code. If at the end you really need to increase FFT computation speed, then you can try fftw. However, to really benefit anything, you really need to apply all tricks explained here...

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

### The openmp thread utilization issue

Ok let us test the current version of libomp (21.1.3 as of October 2025) we get:

```sh
brew install libomp
export CPPFLAGS="-I/opt/homebrew/opt/libomp/include -Xpreprocessor -fopenmp"
export LDFLAGS="-L/opt/homebrew/opt/libomp/lib -lomp"
./configure --enable-armv8-cntvct-el0 --host=aarch64 --enable-neon --enable-openmp --enable-float
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
When observing thread utilization using OMP_WAIT_POLICY=PASSIVE (default value) using apple's activity monitor, I can see poor utilization, and almost half of the processing time appears to be wasted for some reason, so it makes sense that computation time using OMP_WAIT_POLICY=PASSIVE is almost two times longer compared to OMP_WAIT_POLICY=ACTIVE. Using version libomop 14.0.6 and prior to this version give a similar result with both OMP_WAIT_POLICY=PASSIVE and OMP_WAIT_POLICY=ACTIVE. I cannot use active omp wait policy because I am using python and pyfftw. Using OMP_WAIT_POLICY=ACTIVE inside python just spins all cores at 100%, making the environment useless. Therefore, I choose libomop 14.0.6 for my work...






