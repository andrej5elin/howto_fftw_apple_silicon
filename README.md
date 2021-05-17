# howto_fftw_apple_silicon

Note that it is possible to install fftw with brew, but the package there is not yet properly compiled.
Until things get sorted out, here are the instructions on installing pyfftw and fftw on apple silicon computers.

## Installing FFTW 

Download FFTW source code and the fftw_3_3_9_configure_diff.txt file from the repository and configure:

$ 
$ ./configure --enable-threads --enable-neon --enable-float
$ make
$ sudo make install

To compile for double precision, you have to apply a patch to 



