# WRF
How to compile WRF-SFIRE-Chem in Vega and Poznan // Meteogrid Project
# Description

This repository contains the tests, configurations and scripts developed to run WRF model at Poznan HPC. It contains a directory with the collection of tests that have been carried out to ensure that the installation of the model at the HPC is correct and has all the modules needed and also some tests to study configuration and paralelization options and parameters.

A script to set up the environment and also create separate folders for each test is available. In this version, it is needed to configure this script to the folders and namelists in use but it is planned to improve this version when a stable configuration for both the HPC and the WRF model has been reached.

## Installation

### WRF Requirements

#### Libraries

* <https://www.zlib.net/zlib-1.2.13.tar.gz>
* <https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.13.2/src/hdf5-1.13.2.tar.gz>
* <https://downloads.unidata.ucar.edu/netcdf-c/4.9.0/netcdf-c-4.9.0.tar.gz>
* <https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.0/netcdf-fortran-4.6.0.tar.gz>
* <https://www.mpich.org/static/downloads/4.0.3/mpich-4.0.3.tar.gz>
* <https://download.sourceforge.net/libpng/libpng-1.6.37.tar.gz>
* <https://www.ece.uvic.ca/~frodo/jasper/software/jasper-1.900.1.zip>

#### WRF-SFIRE version 4.4

* git clone <https://github.com/openwfm/WRF-SFIRE>
* WPS version 4.3 git clone <https://github.com/openwfm/WPS> or version 4.4  <https://github.com/wrf-model/WPS/releases>

Note that WRF-SFIRE is only needed if the module SFIRE is going to be used, if not just install WRF:

* <https://github.com/wrf-model/WRF/releases>

### WRF-SFIRE INSTALLATION INSTRUCTION @Altair

#### Versions of the software used

* openmpi 4.1.0 (available as a module)
* zlib-1.2.7 (already installed in the system)
* libpng-3.50.0 (already installed in the system)
* jasper-1.900.1
* hdf5-1.14.0
* netcdf-c-4.9.2
* netcdf-fortran-4.6.0
* WRF-SFIRE 4.4
* WPS 4.4

#### Installation

These steps will cover the installation of WRF with SFIRE and CHEM.

1. Allocate a lot of memory for your interactive task:

    ```bash
    srun -n 1 -p altair --mem=30G --pty /bin/bash
    ```

2. Load a module with openmpi

    ```bash
    module load openmpi/4.1.0_gcc620
    ```

3. Create a general software directory

    ```bash
    export SOFTWARE_DIR=/mnt/storage_2/project_data/grant_627/Software
    ```

4. Download WRF and WPS

    ```bash
    cd ${SOFTWARE_DIR}
    git clone https://github.com/openwfm/WRF-SFIRE
    wget https://github.com/wrf-model/WPS/archive/refs/tags/v4.4.tar.gz
    tar -zxvf v4.4.tar.gz
    ```

5. Create a temporary directory for dependencies sources:

    ```bash
    export DEP_SOURCES_DIR=${SOFTWARE}/build
    mkdir -p ${DEP_SOURCES_DIR}
    ```

6. Download all the necessary sources

    ```bash
    cd ${DEP_SOURCES_DIR}
    wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
    wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.0/netcdf-fortran-4.6.0.tar.gz
    wget https://www.ece.uvic.ca/~frodo/jasper/software/jasper-1.900.1.zip
    wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.14/hdf5-1.14.0/src/hdf5-1.14.0.tar.gz
    ```

7. Create a directory for all the dependencies installation

    ```bash
    export DEP_DIR=${SOFTWARE_DIR}/dependencies
    mkdir -p ${DEP_DIR}
    ```

8. Install library dependencies (jasper, HDF5, NETCDF-C, NETCDF-FORTRAN):

    ```bash
    
    # Install Jasper
    cd ${DEP_SOURCES_DIR}
    unzip jasper-1.900.1.zip
    cd ${DEP_SOURCES_DIR}/jasper-1.900.1
    ./configure --prefix=${DEP_DIR}
    make
    make install

    # Install HDF5:
    cd ${DEP_SOURCES_DIR}
    tar -zxvf hdf5-1.14.0.tar.gz
    cd ${DEP_SOURCES_DIR}/hdf5-1.14.0
    CC=mpicc ./configure --with-zlib=/usr/ --prefix=${DEP_DIR} --enable-parallel --enable-fortran --enable-hl
    make
    make install

    # Set PATH and LD_LIBRARY_PATH for further steps
    export PATH=${DEP_DIR}/bin:${PATH}
    export LD_LIBRARY_PATH=${DEP_DIR}/lib:${LD_LIBRARY_PATH}

    # Install NETCDF-C:
    cd ${DEP_SOURCES_DIR}
    tar -zxvf netcdf-c-4.9.2.tar.gz
    cd ${DEP_SOURCES_DIR}/netcdf-c-4.9.2
    CC=mpicc CFLAGS=-fPIC CPPFLAGS=-I${DEP_DIR}/include LDFLAGS="-L${DEP_DIR}/lib -L/usr/lib64/" ./configure --enable-shared --disable-dap --enable-parallel-tests --enable-netcdf4 --enable-netcdf-4 --prefix=${DEP_DIR}
    make
    make install

    # Install NETCDF-FORTRAN:
    cd ${DEP_SOURCES_DIR}
    tar -zxvf netcdf-fortran-4.6.0.tar.gz
    cd ${DEP_SOURCES_DIR}/netcdf-fortran-4.6.0
    CC=mpicc CPPFLAGS=-I${DEP_DIR}/include CFLAGS=$(nc-config --cflags) LDFLAGS=$(nc-config --libs --static) ./configure --prefix=${DEP_DIR} --enable-shared
    make
    make install
    ```

9. Install WRF-SFIRE:

    ```bash
    cd ${SOFTWARE_DIR}/WRF-SFIRE
    # in arch/Config.pl file set $I_really_want_to_output_grib2_from_WRF = "TRUE"
    export NETCDF=${DEP_DIR}
    export HDF5=${DEP_DIR}
    export PHDF5=${DEP_DIR}
    export JASPERLIB=${DEP_DIR}/lib
    export JASPERINC=${DEP_DIR}/include
    ulimit -s unlimited
    export WRF_CHEM=1
    ./clean -a
    ./configure chem
    # Choose option 34
    # Choose option 1
    ./compile -j 1 em_real >& compile.log
    ```

    Note that in order for CHEM to work properly it might be needed to make this change:

    in chem/depend.chem change line number 232 from:

    > module_mosaic_addemiss.o: module_data_mosaic_asect.o module_data_sorgam.o

    to:
    > module_mosaic_addemiss.o: module_data_mosaic_asect.o module_data_sorgam.o module_gocart_dust.o

    continue with the compilation
  
    ```bash
    ./compile -j 1 em_real >& compile.log2
    ```

10. Create link for WRF-SFIRE

    ```bash
    cd ${SOFTWARE_DIR}
    ln -s WRF-SFIRE WRF
    ```

11. Install WPS

    ```bash
    cd ${SOFTWARE_DIR}
    tar -zxvf v4.4.tar.gz
    cd ${SOFTWARE_DIR}/WPS-4.4
    ./clean
    ./configure
    # Choose option 3
    ./compile >& log.compile
    ```

Once the installation is finished each time the user will run the model, it is needed to set up this environment parameters. This can be done using the script that will be used to submit the jobs or using a dedicated set up script created for this purpose.

```bash
module load openmpi/4.1.0_gcc620
export DEP_DIR=/path/to/userhome/grant_XXX/project_data/grant_627/Software/dependencies/
export PATH=${DEP_DIR}/bin:${PATH}
export LD_LIBRARY_PATH=${DEP_DIR}/lib:${LD_LIBRARY_PATH}
export LD_LIBRARY_PATH=${DEP_DIR}/lib64:${LD_LIBRARY_PATH}
ulimit -s unlimited
```

### WRF-SFIRE INSTALLATION INSTRUCTION @Vega

#### Versions of the software used

* openmpi 4.1.4 (available as a module)
* zlib-1.2.12 (available as a module)
* jasper-2.0.33
* libpng-1.6.37
* hdf5-1.14.0
* netcdf-c-4.9.2
* WRF-SFIRE 4.4
* WPS 4.4

1. Set environment variables and modules

```bash
export SOFTWARE_DIR=/ceph/hpc/data/d2023d08-038-users/Software
export DEP_SOURCES_DIR=${SOFTWARE_DIR}/build
mkdir -p ${DEP_SOURCES_DIR}
export DEP_DIR=${SOFTWARE_DIR}/dependencies
export PATH=${DEP_DIR}/bin:${PATH}
export LD_LIBRARY_PATH=${DEP_DIR}/lib:${LD_LIBRARY_PATH}

module load OpenMPI/4.1.4-GCC-12.2.0
module load zlib/1.2.12-GCCcore-12.2.0
```

2. Download software

* Libraries

```bash
cd ${DEP_SOURCES_DIR}
wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.0/netcdf-fortran-4.6.0.tar.gz
wget https://www.ece.uvic.ca/~frodo/jasper/software/jasper-1.900.1.zip
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.14/hdf5-1.14.0/src/hdf5-1.14.0.tar.gz
```

* WRF & WPS

```bash
cd ${SOFTWARE_DIR}
git clone https://github.com/openwfm/WRF-SFIRE
wget https://github.com/wrf-model/WPS/archive/refs/tags/v4.4.tar.gz
tar -zxvf v4.4.tar.gz
```

3. Install library dependencies (jasper, HDF5, NETCDF-C, NETCDF-FORTRAN):

* Install Jasper

```bash
cd ${DEP_SOURCES_DIR}
unzip jasper-1.900.1.zip
cd ${DEP_SOURCES_DIR}/jasper-1.900.1
./configure --prefix=${DEP_DIR}
make
make install
```

* Install libpng

```bash
cd ${DEP_SOURCES_DIR}
tar xvzf libpng-1.6.37.tar.gz
cd ${DEP_SOURCES_DIR}/libpng-1.6.37
./configure --prefix=${DEP_DIR}
make
make install

* Install HDF5
```bash
cd ${DEP_SOURCES_DIR}
tar -zxvf hdf5-1.14.0.tar.gz
cd ${DEP_SOURCES_DIR}/hdf5-1.14.0
CC=mpicc ./configure --with-zlib=/usr/ --prefix=${DEP_DIR} --enable-parallel --enable-fortran --enable-hl
make
make install
```

* Install NETCDF-C

```bash
module load cURL
cd ${DEP_SOURCES_DIR}
tar -zxvf netcdf-c-4.9.2.tar.gz
cd ${DEP_SOURCES_DIR}/netcdf-c-4.9.2
CC=mpicc CFLAGS=-fPIC CPPFLAGS=-I${DEP_DIR}/include LDFLAGS="-L${DEP_DIR}/lib -L/usr/lib64/" ./configure --enable-shared --disable-dap --enable-parallel-tests --enable-netcdf4 --enable-netcdf-4 --prefix=${DEP_DIR}
make
make install
```

* Install NETCDF-FORTRAN

```bash
cd ${DEP_SOURCES_DIR}
tar -zxvf netcdf-fortran-4.6.0.tar.gz
cd ${DEP_SOURCES_DIR}/netcdf-fortran-4.6.0
CC=mpicc CPPFLAGS=-I${DEP_DIR}/include CFLAGS=$(nc-config --cflags) LDFLAGS=$(nc-config --libs --static) ./configure --prefix=${DEP_DIR} --enable-shared
make
make install
```

* Install WRF-SFIRE

```bash
cd ${SOFTWARE_DIR}/WRF-SFIRE
# in arch/Config.pl file set $I_really_want_to_output_grib2_from_WRF = "TRUE"
export NETCDF=${DEP_DIR}
export HDF5=${DEP_DIR}
export PHDF5=${DEP_DIR}
export JASPERLIB=${DEP_DIR}/lib
export JASPERINC=${DEP_DIR}/include
ulimit -s unlimited
export WRF_CHEM=1
./clean -a
./configure chem
# Choose option 34
# Choose option 1
```

Note that in order for CHEM to work properly it might be needed to make this change:

in chem/depend.chem change line number 232 from:

> module_mosaic_addemiss.o: module_data_mosaic_asect.o module_data_sorgam.o

to:
> module_mosaic_addemiss.o: module_data_mosaic_asect.o module_data_sorgam.o module_gocart_dust.o

continue with the compilation

Failed compilation:

Tried:

* switching to bash
* adding flag -DLANDREAD_STUB CFLAGS in configure.wrf
* compiling with 4 & 1 core

What worked:

* Before compiling wrf edit the compile.wrf file and at line 139 change (<https://forum.mmm.ucar.edu/threads/compiling-wrf-4-0-2-test-em_b_wave-failed.8991/>):

```bash
FC = time $(DM_FC)
```

to

```bash
FC = $(DM_FC)
```

```bash
./compile -j 1 em_real | tee compile.log
```

Note: it might be interesting to test with "-j 4 "

Create link for WRF-SFIRE:


### WRF Requirements

* Install WPS

```bash
cd ${SOFTWARE_DIR}
tar -zxvf v4.4.tar.gz
cd ${SOFTWARE_DIR}/WPS-4.4
./clean
./configure
# Choose option 3
./compile >& log.compile
```
