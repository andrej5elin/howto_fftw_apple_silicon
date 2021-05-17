# howto_fftw_apple_silicon

Note that it is possible to install fftw with brew, but the package there is not yet properly compiled.
Until things get sorted out, here are the instructions on installing pyfftw and fftw on apple silicon computers.

## Installing FFTW 

Download FFTW source code and the fftw-3-3-9-configure-diff.txt file from the repository and configure:

```console
$ patch configure fftw-3-3-9-configure-diff.txt
$ ./configure --enable-threads --enable-neon --enable-armv8-cntvct-el0 --enable-float
$ make
$ sudo make install
$ ./configure --enable-threads --enable-armv8-cntvct-el0 --enable-neon 
$ make
$ sudo make install
$ ./configure --enable-threads --enable-armv8-cntvct-el0 --enable-long-double
$ make
$ sudo make install
```





