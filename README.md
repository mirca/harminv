# Harminv
[![Build Status](https://travis-ci.org/stevengj/harminv.png)](https://travis-ci.org/stevengj/harminv)

Harmonic Inversion of Time Signals by the Filter Diagonalization Method (FDM),
implemented by [Steven G. Johnson](http://math.mit.edu/~stevenj/), Massachusetts Institute of Technology.  See also the

* [Harminv Home Page](http://ab-initio.mit.edu/harminv)

## Introduction

Harminv is a free program (and accompanying library) to solve the
problem of "harmonic inversion."  Given a discrete, finite-length
signal that consists of a sum of finitely-many sinusoids (possibly
exponentially decaying), it determines the frequencies, decay
constants, amplitudes, and phases of those sinusoids.

It can, in principle, provide much better accuracy than
straightforward FFT based methods, essentially because it assumes a
specific form for the signal.  (Fourier transforms, in contrast,
attempt to represent *any* data as a sum of sinusoidal components.)

We use a low-storage "filter diagonalization method" (FDM) for
finding the sinusoids near a given frequency interval, described in:

* V. A. Mandelshtam and H. S. Taylor, "Harmonic inversion of time
  signals," J. Chem. Phys., vol. 107, no. 17, p. 6756-6769 (Nov. 1
  1997).  See also erratum, ibid, vol. 109, no. 10, p. 4128 (Sep. 8
  1998).

This kind of spectral analysis has wide applications in many areas of
physics and engineering, as well as other fields.  For example, it
could be used to extract the vibrational or "eigen" modes of a system
from its response to some stimulus, and also their rates of decay in
dissipative systems.  FDM has been applied to analyze, e.g., NMR
experimental data and numerical simulations of quantum mechanics.

See also:

* Rongqing Chen and Hua Guo, "Efficient calculation of matrix
  elements in low storate filter diagonalization," J. Chem. Phys.,
  vol. 111, no. 2, p. 464-471(Jul. 8 1999).

* Michael R. Wall and Daniel Neuhauser, "Extraction, through
  filter-diagonalization, of general quantum eigenvalues or classical
  normal mode frequencies from a small number of residues or a
  short-time segment of a signal. I. Theory and application to a
  quantum-dynamics model," J. Chem. Phys., 102, no. 20, p. 8011-8022
  (May 22 1995).

* V. A. Mandelshtam, "On harmonic inversion of cross-correlation
  functions by the filter diagonalization method," J. Theoretical and
  Computational Chemistry 2 (4), 497-505 (2003).

Essentially, FDM works by considering the time-series to be the
autocorrelation function of a fictitious dynamical system, such that
the problem of finding the frequencies and decay constants is
re-expressed as the problem of finding the eigenvalues of the
complex-symmetric time-evolution operator of this system.  The key
point is that, if you are only interested in frequencies within a
known band-limited region, the matrix elements of this operator can be
expressed purely in terms of Fourier transforms (or, really, z
transforms) of your time-series.  Then, one can simply diagonalize a
small matrix (size proportional to the bandwidth and the number of
frequencies) to find the desired result.

In general, for M data points and J frequencies, the time required is
O(M J + J^3).  The main point of the algorithm is not so much speed,
however, but the effective solution of a very ill-conditioned fitting
problem.  (Even closely-spaced frequencies and/or weak decay rates can
be resolved much more reliably by FDM than by straightforward fits of
the data or its spectrum.)

## Program Usage

The usage of the harminv program is described by [the harminv man page](http://ab-initio.mit.edu/harminv/harminv-man.html) (`man
harminv`), included in the installation below.  To briefly summarize,
it takes a sequence of numbers (real or complex) from standard input
and a range of frequencies to search and outputs the frequencies it
finds.

## Test Cases/Examples

The input for harminv should just be a list of numbers (real or
complex), one per line, as described in the harminv man page.

You can use the program `sines`, in the harminv source directory, to
test harminv and to generate example inputs.  The sines program
generates a signal consisting of a sum of decaying sinuoids with
specified complex frequencies.  For example,

    ./sines 0.1+0.01i 0.08+0.001i 

generates 10000 data points consisting of a signal with complex
frequencies 0.1+0.01i and 0.08+0.001i, with amplitudes 1 and 2
respectively, sampled at time intervals dt=1.0.  If we input this data
into harminv, it should be able to extract these frequencies, decay
rates, and amplitudes.

    ./sines 0.1+0.01i 0.08+0.001i | harminv 0.05-0.15

The output should be something like:

    frequency, decay constant, Q, amplitude, phase, error
    0.08, 1.000000e-03, 251.327, 2, 3.14159, 1.064964e-16
    0.1, 1.000000e-02, 31.4159, 1, -4.31228e-15, 2.265265e-15

as expected.  Note that we have to pass harminv a range of frequencies
to search, here 0.05-0.15, which shouldn't be too large and should
normally not include 0.  In most cases, one would also specify the
sampling interval to harminv via `harminv -t <dt>`, but in this case we
don't need to because `-t 1.0` is the default.

Run `./sines -h` to get more options.

## Library Usage

The usage of the library `-lharminv` is analogous to the program.  In C
or C++, you first `#include <harminv.h>`, then specify the data and the
frequency range by calling `harminv_data_create`, returning a
`harminv_data` data structure:

    harminv_data harminv_data_create(int n, 
                                     const harminv_complex *signal,
                                     double fmin, double fmax, int nf);

`signal` is a pointer to an array of `n` complex numbers.  In C++,
`harminv_complex` is `std::complex<double>`.  In C, `harminv_complex` is a
`double[2]` with the real parts in `signal[i][0]` and the imaginary parts
in `signal[i][1]`.  (For a real signal, set the imaginary parts to
zero.)  `fmin` and `fmax` are the frequency range to search, and `nf` is the
number of spectral basis functions (see below).  Frequencies are in
units corresponding to a sampling interval of 1 time unit; if your
actual sampling interval is dt, then you should rescale your
frequencies by multiplying them by dt.

A good default for `nf` is `min(300, (fmax - fmin) * n * 1.1)`,
corresponding to a spectral "density" of at most 1.1 (see also the `-d`
option of the command-line tool).  That is, this uses a number of
initial basis functions corresponding to the Fourier resolution of
`1/n`.  This does *not* determine the frequency resolution of the
outputs, which can be much greater than the Fourier resolution.  It
sets an upper bound on the number of modes to search for, and in some
sense is the density with which the bandwidth is initially "searched"
for modes.  Spectral densities much larger than 1 are not recommended,
as they lead to large and singular matrices and unstable results.
Note also that the computation time goes as O(n * nf) + O(nf^3).

Then, you solve for the frequencies by calling:

    void harminv_solve(harminv_data d);

Then, the frequencies and other data can be extracted from `d` by the
following routines.  The number N of frequencies found is returned by:

    int harminv_get_num_freqs(harminv_data d);

Then, for each index 0 <= k < N, the corresponding frequency and decay
constant (as defined in `man harminv`) are returned by:

    double harminv_get_freq(harminv_data d, int k);
    double harminv_get_decay(harminv_data d, int k);

Alternative, you can get the complex angular frequency (omega =
2*pi*freq - i*decay) by:

    void harminv_get_omega(harminv_complex *omega, harminv_data d, int k);

You can get the "quality factor" Q (pi |freq| / decay) by:

    double harminv_get_Q(harminv_data d, int k);

The complex amplitude (|amp| * exp(-I phase)) for each k is returned by:

    void harminv_get_amplitude(harminv_complex *amplitude, harminv_data d, int k);

A crude estimate of the relative error in the (complex) frequency is:

    double harminv_get_freq_error(harminv_data d, int k);

As described in `man harminv`, this is not really an error bar, and
should be treated more as a figure of merit (smaller is better).

### Linking

To link to the library, you need to not only link to `-lharminv`, but
also to the math library, the BLAS and LAPACK libraries (see below),
and any libraries that are required to link C with Fortran code (like
LAPACK).  If you have the `pkg-config` program installed (standard on most
GNU/Linux systems), you can simply do:

    pkg-config --cflags harminv
    pkg-config --libs harminv

to get the flags for compiling and linking, respectively.  You may
need to tell `pkg-config` where to find `harminv.pc` if `harminv` was
installed under `/usr/local` (the default)...in this case, you would
specify `/usr/local/lib/pkgconfig/harminv.pc` instead of `harminv`
above.

There is an additional wrinkle.  If you configured harminv with
`--with-cxx`, or if your C compiler did not support C99 complex numbers
and the configure script automatically switched to C++, then you will
need to link to harminv with the C++ linker, even if your program is
written in C, in order to link the C++ libraries.

## Installation

Harminv is designed to run on any Unix-like system (GNU/Linux is
fine).  (It may also be possible to compile it on other systems, as it
is mainly ANSI C with the exception of one or two POSIX functions like
getopt.)

However, you do need a couple of prerequisites:

* [BLAS](http://www.netlib.org/blas/)

Basic Linear Algebra Subroutines (matrix-multiplications, etcetera),
following a standard interface, for use by LAPACK (see below).  There
are many optimized versions of BLAS available, e.g. a free library
called [ATLAS](http://www.netlib.org/atlas/) or [OpenBLAS](http://www.openblas.net/).

* [LAPACK](http://www.netlib.org/lapack/)

A standard, free, linear-algebra package.  Note that the default
configuration script looks for LAPACK by linking with `-llapack`.  This
means that the library must be called `liblapack.a` and be installed in
a standard directory like `/usr/local/lib` (alternatively, you can
specify another directory via the `LDFLAGS` environment variable; see
below).

See also [our BLAS installation notes](http://ab-initio.mit.edu/wiki/index.php/Template:Installing_BLAS_and_LAPACK) for
links to other information on installing BLAS and LAPACK.

Once you have those installed, you can compile and install harminv.
harminv comes with a GNU-style `configure` script, so on Unix systems
compilation is ideally just a matter of:

    ./configure && make

and then switching to root and running:

    make install

In order to compile, harminv requires either:

* An ANSI C compiler supporting complex numbers, as defined in
  the ANSI C99 standard (or a reasonable approximation thereof).
  For example, gcc-2.95 with GNU libc is fine.

* A C++ compiler supporting the `complex<double>` standard template class.

The `configure` script looks for a C compiler with complex numbers first,
and then for a C++ compiler.  You can force it to use C++ by passing
`--with-cxx` to `configure`.

If you need to, you can have further control over the configure
script's behavior by setting enviroment variables.  This can be useful
especially if you have libraries installed in nonstandard locations
(e.g. in your home directory, if you are not root), to tell the
compiler where to look.  The most common variables to set are:

* `CC`: the name of the C compiler

* `CPPFLAGS`: `-I<dir>` flags to tell the C compiler where to look for header files

* `CFLAGS`: C compiler flags

* `F77`: the name of the Fortran compiler

* `FFLAGS`: Fortran compiler flags

* `LDFLAGS`: linker flags (e.g. `-L<dir>` to look for libraries in `<dir>`)

