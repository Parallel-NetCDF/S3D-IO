# PnetCDF I/O Benchmark using S3D Application I/O Kernel

This software benchmarks the performance of [ADIOS](https://www.olcf.ornl.gov/center-projects/adios/) writing and
[PnetCDF](https://parallel-netcdf.github.io/) reading method implementing the I/O
kernel of [S3D combustion simulation code](http://exactcodesign.org).
The evaluation method is [weak scaling](https://en.wikipedia.org/wiki/Scalability#Weak_versus_strong_scaling).

S3D is a continuum scale first principles direct numerical simulation code
which solves the compressible governing equations of mass continuity, momenta,
energy and mass fractions of chemical species including chemical reactions.
Readers are referred to the published paper below.
* J. Chen, A. Choudhary, B. de Supinski, M. DeVries, E. Hawkes, S. Klasky,
  W. Liao, K. Ma, J. Crummey, N. Podhorszki, R. Sankaran, S. Shende, and
  C. Yoo. Teras-cale Direct Numerical Simulations of Turbulent Combustion
  Using S3D. In Computational Science and Discovery Volume 2, January 2009.

## I/O pattern:
A data checkpoint is performed at regular time intervals, and its data consist
of three- and four-dimensional array variables of type `double`. At each
checkpoint, four global arrays, representing mass, velocity, pressure, and
temperature, respectively, are written to a newly created file in the canonical
order. Mass and velocity are four-dimensional arrays while pressure and
temperature are three-dimensional arrays. All four arrays share the same size
of the lowest three spatial dimensions X, Y, and Z, which are partitioned among
MPI processes in a **block-block-block** fashion. For the mass and velocity
arrays, the length of the fourth dimension is 11 and 3, respectively. The
fourth dimension, the most significant one, is not partitioned. As the number
of MPI processes increases, the aggregate I/O amount proportionally increases
as well.

For detailed description of the data partitioning and I/O patterns, please
refer to the following paper.
* W. Liao and A. Choudhary. Dynamically Adapting File Domain Partitioning
  Methods for Collective I/O Based on Underlying Parallel File System
  Locking Protocols. In the Proceedings of International Conference for
  High Performance Computing, Networking, Storage and Analysis, Austin,
  Texas, November 2008.

## To compile:
Edit file `Makefile` and adjust the following variables:
```
    MPIF90         - MPI Fortran 90 compiler
    FCFLAGS        - compile flags
    PNETCDF_DIR    - the path of PnetCDF library (1.12.0 and higher with ADIOS interoperability is required)
    ADIOS_DIR      - the path of ADIOS library (1.12.0 and higher is required)
```
For example:
```
    MPIF90      = /usr/bin/mpif90
    FCFLAGS     = -O2
    PNETCDF_DIR = ${HOME}/PnetCDF
    ADIOS_DIR = ${HOME}/ADIOS
```
## To run:
The command-line arguments are shown below, which can also be obtained by command
`./s3d_io.x -h`.
```
Usage: s3d_io.x nx_g ny_g nz_g npx npy npz method restart dir_path
```
There are 9 command-line arguments:
```
       nx_g     - GLOBAL grid size along X dimension
       ny_g     - GLOBAL grid size along Y dimension
       nz_g     - GLOBAL grid size along Z dimension
       npx      - number of MPI processes along X dimension
       npy      - number of MPI processes along Y dimension
       npz      - number of MPI processes along Z dimension
       method   - 0: using PnetCDF blocking APIs, 1: non-blocking APIs ( since PnetCDF does not support non-blocking API on ADIOS files, this option is simply disregarded by the benchmark program. It is left in the benchmark for future extension )
       restart  - restart from reading a previous written file (True/False)
       dir_path - the directory name to store the output files
```
To change the number of checkpoint dumps (default is set to 5), edit
file `param_m.f90` and set a different value for `i_time_end`:
```
       i_time_end = 5   ! number of checkpoints (also number of output files)
```
The contents of all variables written to BP files are randomly generated
numbers. This setting can be disabled by commenting out the line below in file
`solve_driver.f90`. Commenting it out can reduce the benchmark execution time.
```
       call random_set
```

### Example run command:
For a test run with small data size and a short return time, here is an
example command for running on 4 MPI processes.
```
   mpiexec -n 4 ./s3d_io.x 10 10 10 2 2 1 1 F .
```

The example command below runs a job on 4096 MPI processes with the global
array of size `800 x 800 x 800` and local arrays of size `50 x 50 x 50`, output
directory `/scratch1/scratchdirs/wkliao/FS_1M_96` using nonblocking APIs, and
without restart.
```
   mpiexec -l -n 512 ./s3d_io.x 800 800 800 16 16 16 1 F /scratch1/scratchdirs/wkliao/FS_1M_96
```

## Example output from screen:
```
++++ I/O is done through ADIOS/PnetCDF ++++
I/O method          : blocking APIs
Run with restart    : True
No. MPI processes   :       8
Global array size   :     100 x    100 x    100
output file path    : ./test
file striping count :       0
file striping size  :       0 bytes
-----------------------------------------------
Time for open       :        0.84 sec
Time for read       :        0.43 sec
Time for write      :        0.83 sec
Time for close      :        1.35 sec
no. read  calls     :        4    per process
no. write calls     :       20    per process
total read  amount  :        0.12 GiB
total write amount  :        0.60 GiB
read  bandwidth     :      285.29 MiB/s
write bandwidth     :      738.54 MiB/s
-----------------------------------------------
total I/O   amount  :        0.72 GiB
total I/O   time    :        2.03 sec
I/O   bandwidth     :      361.61 MiB/s
```

## Example metadata of the output file.
```
% ncdump -h pressure_wave_test.0.000E+00.field.nc

netcdf pressure_wave_test.0.000E+00.field {
// file format: ADIOS BP Ver. 3
dimensions:
  x = 100 ;
  y = 100 ;
  z = 100 ;
  lx = 50 ;
  ly = 50 ;
  lz = 50 ;
  sx = 1 ;
  sy = 1 ;
  sz = 1 ;
  nsc = 11 ;
  u_3 = 3 ;
  one = 1 ;
variables:
  double yspecies(nsc, z, y, x) ;
  double u(u_3, z, y, x) ;
  double pressure(z, y, x) ;
  double temp(z, y, x) ;
  double time_var(one) ;
  double tstep_var(one) ;
  double time_save_var(one) ;

// global attributes:
        :time = 0. ;
        :tstep = 0. ;
        :time_save = 100000. ;
}
```

## Questions/Comments:
email: khl7265@eecs.northwestern.edu

Copyright (C) 2019, Northwestern University

See COPYRIGHT notice in top-level directory.

