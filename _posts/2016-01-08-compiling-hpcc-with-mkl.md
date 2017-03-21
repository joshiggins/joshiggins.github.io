---
layout: post
comments: true
categories: 
    - hpc
    - howto
    - benchmarking
    - cluster
title: "Compiling HPC Challenge benchmark (hpcc) with the Intel MKL"
fulltitle: "Compiling HPC Challenge benchmark (hpcc) with the Intel MKL"
excerpt: ""
---

The [HPC Challenge](http://icl.cs.utk.edu/hpcc/) is a benchmark that combines HPL (Linpack) and a battery of other tests on memory, floating point and network performance that can give you a pretty well-rounded view on the performance characteristics of your cluster.

For Intel processors, the [Intel Math Kernel Library (MKL)](https://software.intel.com/en-us/intel-mkl) can be used instead of [CBLAS](http://www.netlib.org/blas/) or [ATLAS](http://math-atlas.sourceforge.net/) to get better performance on the HPL benchmark. So it seems preferential to compile the HPC Challenge (which includes an HPL) with the Intel MKL.

This [article](https://software.intel.com/en-us/articles/performance-tools-for-software-developers-use-of-intel-mkl-in-hpcc-benchmark) on the Intel Developer Zone is way out of date and doesn't offer any help if you are compiling without Intel's C and Fortran compilers.

## Step 0: Prerequisites

- Install the Intel MKL (you can get it for free for academia)
- Install the development packages you need
- Download and extract the HPCC source files

### My setup

I used the following

- Ubuntu 15.10
- MPICH2 3.1
- Intel MKL 11.3.0.109
- hpcc-1.5.0b
- gcc 5.2.1
- clang 3.6.2-1

This equates to 

```
$ apt-get install build-essential clang libmpich2-dev libiomp-dev libfftw-dev
```

## Step 1: Build MPI MKL FFTW library and C wrapper

```
$ cd /opt/intel/mkl/interfaces/fftw2x_cdft
```

```
$ make libintel64 PRECISION=MKL_DOUBLE interface=ilp64 mpi=mpich2 compiler=gnu
```

```
$ cd /opt/intel/mkl/interfaces/fftw2xc
```

```
$ make libintel64 PRECISION=MKL_DOUBLE compiler=gnu
```

This should create the files `libfftw2x_cdft_DOUBLE_ilp64.a` and `libfftw2xc_double_gnu.a` in `/opt/intel/mkl/lib/intel64`. If you are using a different MPI implementation, just type `make` to see a list of available options.


## Step 2: Create Makefile for HPL

```
$ cd (path to hpcc)/hpl
```

```
$ cp setup/Make.Linux_PII_CBLAS Make.intel64
```

Now you should edit this `Make.intel64` file to point to MPI and MKL libraries.

### Set the MPI library

For using MPICH2 on Ubuntu set MPdir, MPinc and MPlib like this

<pre>
MPdir        = /usr/lib/mpich
MPinc        = -I$(MPdir)/include
MPlib        = /usr/lib/x86_64-linux-gnu/libmpich.a
</pre>

### Set the MKL library

Set the LAdir and LAlib variables to point to the Intel MKL library

<pre>
LAdir        = /opt/intel/mkl/lib/intel64
LAinc        =
LAlib        = -Wl,--start-group $(LAdir)/libfftw2x_cdft_DOUBLE_ilp64.a \
            $(LAdir)/libfftw2xc_double_gnu.a \
            $(LAdir)/libmkl_intel_lp64.a \
            $(LAdir)/libmkl_intel_thread.a \
            $(LAdir)/libmkl_core.a \
            $(LAdir)/libmkl_blacs_intelmpi_lp64.a \
            $(LAdir)/libmkl_cdft_core.a \
            -Wl,--end-group
</pre>

### Set the compiler options

The compiler options are the same whether you want to use GCC or clang. clang seems to compile faster for me and looks pretty.

<pre>
CC           = /usr/bin/[gcc|clang]
CCNOOPT      = $(HPL_DEFS)
CCFLAGS      = $(HPL_DEFS) -fomit-frame-pointer -O3 -funroll-loops \
        -Wl,--no-as-needed -ldl -lmpi -liomp5 -lpthread -lm \
        -DUSING_FFTW -DMKL_INT=long -DLONG_IS_64BITS
</pre>

You shouldn't use `-lopenmp` because we want it to be linked against `libiomp5`.

You should also change the linker to use `gfortran`.

<pre>
LINKER       = /usr/bin/gfortran
LINKFLAGS    = $(CCFLAGS)
</pre>

## Step 3: Build HPCC

From the top level of the HPCC source tree, just run

```
$ make all arch=intel64
```

and you should get a nice shiny `hpcc` executable. Now to tune those HPL parameters...

## The Results

On my modest desktop with Intel(R) Core(TM) i7-4790 CPU @ 3.60GHz and 16GB of memory, the following results were obtained with a minimal amount of HPL.dat tweaking, using [this](http://www.clusterkit.co.th/cluster_cal.php) as a starting point.

<table>
<thead>
    <tr>
        <td><b>Compiler</b></td>
        <td><b>Gflops</b></td>
    </tr>
</thead>
<tbody>
    <tr>
        <td>GCC 5.2.1</td>
        <td>77.6</td>
    </tr>
    <tr>
        <td>clang 3.6.2-1</td>
        <td>76.4</td>
    </tr>
</tbody>
</table>

Not so much difference, I wonder how it compares with the Intel compilers. I was surprised clang is slower, but maybe it will be faster second time around.
