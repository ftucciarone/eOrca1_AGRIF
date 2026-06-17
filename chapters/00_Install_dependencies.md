<a href="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/chapters/01_Install_NEMO.md" class="next">Next &raquo;</a>

# Nemo dependencies
> [!WARNING]
> *Lasciate ogne speranza, voi ch'intrate*

> [!IMPORTANT]
> These instruction are not going to be bulletproof. We built them upon the work of previous wise men. Among them, in alphabetical order, honorable mentions should be addressed to 
> * [Romain Caneill](https://romaincaneill.fr), (role unknown) wrote a great [apptainer](https://github.com/rcaneill/xnemogcm_test_data) with all the dependencies;
> * [Julian Mak](https://julianmak.github.io), NEMO System Team member, wrote incredible notes and [made them available](https://nemo-related.readthedocs.io/en/latest/index.html);
> * [Sebastian Masson](https://forge.nemo-ocean.eu/users/smasson/snippets), NEMO System Team member;
> * [Yann Meurdesoif](chapters/Install_dependencies.md), main developer and maintainer of XIOS. Wrote (I can imagine alongside his collaborators) the [XIOS wiki](https://lmdz-forge.lmd.jussieu.fr/mediawiki/Planets/index.php/The_XIOS_Library), containing a lot of tips and insights, and this [bash template](https://web.lmd.jussieu.fr/~lmdz/pub/script_install/install_netcdf4_hdf5.bash) for the prerequisites.


Nemo requires a set of fairly complicated dependencies. Among these we have 
* C/C++ compiler (we will install gcc and g++);
* Fortran compiler (we will install gfortran);
* MPI, Message Passing Interface (we will install both mpich and openmpi);
* NetCDF-C;
* NetCDF-Fortran;
* HDF5;
* XIOS (Optional but highly recommended).
Other important libraries and tools to be installed are svn, wget, git, make 

> [!WARNING]
> We will install also libcurl4-openssl-dev, m4, liburi-perl, libxml2-dev, but frankly I don't know what they are or why we are doing this. The original tutorial by Romain Caneill included them, and I have no reasons to doubt him.

### Step 1: Define the installation parameters
This procedure will create directories, download tarballs and sources, install libraries. In particular, it will create folders to manage easily the installation points. The final structure will be as the following tree:
```
.
└── $ROOT/                # Root folder for the project
    ├── nemo-deps/          # Dependencies for XIOS/NEMO
    │   ├── sources/          # Sources tarballs ($SRCSDIR)
    │   ├── installs/         # Installation points ($INSTDIR)
    │   │   ├── bin/
    │   │   ├── include/
    │   │   ├── lib/
    │   │   └── share/
    │   └── XIOS/             # XIOS base folder (if multiple versions are needed)
    │       ├── xios-trunk/     # target XIOS dir ($XIOSDIR)
    │       └── ...             # other XIOS versions to target
    └── $WORKDIR
```
The basic idea is that `nemo-deps` will contain all the dependencies and it is separated from `$WORKDIR` where we can base the work. In this way you can have multiple versions of NEMO based on the same dependencies. `XIOS` lives in its own directory because different versions of it are available (and not all of them are compatible with some specific version of NEMO) and thus it's safer to have it this way. 

The only thing that should be modified by the user is the `ROOT` variable, that specifies the root folder where everything will be done. 

The following snippet contains the parameters to set up the directories tree and the compilers to use. This section can be copy-pasted in a file called `install_params.sh` and then sourced. 
```shell
# Let's base ourselves in the base directory
ROOT=$HOME
# Echo where we work
echo $ROOT

# Installation directories
export SRCSDIR=$ROOT/nemo-deps/sources
export INSTDIR=$ROOT/nemo-deps/installs
mkdir -p $SRCSDIR
mkdir -p $INSTDIR
mkdir -p $ROOT/nemo-deps/XIOS

# compilers
export CC=/usr/bin/mpicc
export CXX=/usr/bin/mpicxx
export FC=/usr/bin/mpif90
export F77=/usr/bin/mpif77

# compiler flags (except for libraries)
export CFLAGS="-O3 -fPIC"
export CXXFLAGS="-O3 -fPIC"
export F90FLAGS="-O3 -fPIC"
export FCFLAGS="-O3 -fPIC"
export FFLAGS="-O3 -fPIC"
export LDFLAGS="-O3 -fPIC "

# FLAGS FOR F90  TEST-EXAMPLES
export FCFLAGS_f90="-O3 -fPIC "
```

### Step 2: Install the packages
This part should be adapted, depending on the machine you are using. In this example, an Ubuntu machine is used and thus the package manager is `apt`. 
```shell
# Install the packages
sudo apt-get -y update
sudo apt-get install -y openmpi-bin libmpich-dev libopenmpi-dev gcc g++ gfortran subversion libcurl4-openssl-dev wget make m4 git liburi-perl libxml2-dev
```

### Step 3: Install Zlib
According to [zlib](https://www.zlib.net)'s documentation, **zlib** is designed to be a free, general-purpose, legally unencumbered -- that is, not covered by any patents -- lossless data-compression library for use on virtually any computer hardware and operating system. The zlib data format is itself portable across platforms. It is a prerequisite for NetCDF (if I understood correctly). Check the website for the latest version and manually set this variable. Then, the installation procedure is the following.
```shell
cd $SRCSDIR
LIB_VERSION="zlib-1.3.1"
wget https://www.zlib.net/${LIB_VERSION}.tar.gz
tar xvfz ${LIB_VERSION}.tar.gz
cd $LIB_VERSION
./configure --prefix=$INSTDIR
make -j1
##make check
make install
echo " " 
```
> [!NOTE]
> We have just started populating the `$ROOT/nemo-deps/installs` directory. This will be indicated by the variable `$INSTDIR`. You can check with `echo $INSTDIR` that this is the correct folder if you do this in different moments and not right after step 1.

### Step 4: Install HDF5
Hierarchical Data Format (HDF) is a set of file formats (HDF4, HDF5) designed to store and organize large amounts of data. It is supported by The [HDF Group](https://www.hdfgroup.org), a non-profit corporation whose mission is to ensure continued development of HDF5 technologies and the continued accessibility of data stored in HDF.

You should check at the [download](https://support.hdfgroup.org/downloads/hdf5/hdf5_1_14_6.html) page which is the version you want to install, but download it from their [GitHb Releases](https://github.com/HDFGroup/hdf5/releases) page.

Once the version has been chosen, manually change the version (and check the link) and run the following:
```shell
cd $SRCSDIR
LIB_VERSION="hdf5_1.14.6"
wget https://github.com/HDFGroup/hdf5/archive/refs/tags/${LIB_VERSION}.tar.gz
tar xvfz ${LIB_VERSION}.tar.gz
cd hdf5-$LIB_VERSION
export HDF5_Make_Ignore=yes
# Configure
./configure --prefix=$INSTDIR --enable-fortran  --enable-parallel --enable-hl --enable-shared --with-zlib=$INSTDIR
# Make and install
make -j1
##make check
make install
echo " " 
```

### Step 5: Install NetCDF-C and NetCDF-F
As for the HDF5, you can control the version of NetCDF on the github page, update the variable `LIB_VERSION` and run the following.
> [!IMPORTANT]
> Order matters: NetCDF-C before, NetCDF-Fortran later. 

```shell
# install netcdf-c
cd $SRCSDIR
LIB_VERSION="4.9.3"
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v${LIB_VERSION}.tar.gz
tar xvfz v${LIB_VERSION}.tar.gz
cd netcdf-c-${LIB_VERSION}
export CPPFLAGS="-I$INSTDIR/include -DpgiFortran"
export LDFLAGS="-Wl,-rpath,$INSTDIR/lib -L$INSTDIR/lib -lhdf5_hl -lhdf5"
export LIBS="-lmpi"
./configure --prefix=$INSTDIR --enable-netcdf-4 --enable-shared --enable-parallel-tests
make -j1
##make check
make install
echo " " 

# Install netcdf-fortran
cd $SRCSDIR
LIB_VERSION="4.6.2"
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v${LIB_VERSION}.tar.gz
tar xvfz v${LIB_VERSION}.tar.gz
cd netcdf-fortran-${LIB_VERSION}
export LD_LIBRARY_PATH=${NCDIR}/lib:${LD_LIBRARY_PATH}
export CPPFLAGS="-I$INSTDIR/include -DpgiFortran"
export LDFLAGS="-Wl,-rpath,$INSTDIR/lib -L$INSTDIR/lib -lnetcdf -lhdf5_hl -lhdf5 -lz -lcurl"
export LIBS="-lmpi"
./configure --prefix=$INSTDIR --enable-shared --enable-parallel-tests --enable-parallel
make -j1
##make check
make install
echo " " 
```

## Install XIOS
```shell
mkdir -p $ROOT/nemo-deps/XIOS
cd $ROOT/nemo-deps/XIOS
svn co -r 2701 http://forge.ipsl.fr/ioserver/svn/XIOS/trunk xios-trunk
rm -rf $(find . -iname .svn)
cd ..
```
The following steps are crucial. Inside the `xios-trunk/arch/` folder we need to set three different files with environment specifications, linking specifications and compiler specifications. To do so, we need to create an environment file `arch-GCC_LINUX_local.env` containing the following:
```shell
# ATTENTION INSTDIR must be defined before

export HDF5_INC_DIR=$INSTDIR/include
export HDF5_LIB_DIR=$INSTDIR/lib

export NETCDF_INC_DIR=$INSTDIR/include
export NETCDF_LIB_DIR=$INSTDIR/lib
```
Then we need to create a fcm file: `arch-GCC_LINUX_local.fcm` containing the following:
```shell
################################################################################
###################                Projet XIOS               ###################
################################################################################

%CCOMPILER      mpicc
%FCOMPILER      mpif90
%LINKER         mpif90

%BASE_CFLAGS    -std=c++11
%PROD_CFLAGS    -O3 -DBOOST_DISABLE_ASSERTS
%DEV_CFLAGS     -g -O2 
%DEBUG_CFLAGS   -g 

%BASE_FFLAGS    -D__NONE__ 
%PROD_FFLAGS    -O3
%DEV_FFLAGS     -g -O2
%DEBUG_FFLAGS   -g 

%BASE_INC       -D__NONE__
%BASE_LD        -lstdc++

%CPP            cpp
%FPP            cpp -P
%MAKE           make
```
And finally create a path file `arch-GCC_LINUX_local.path` containing:
```shell
NETCDF_INCDIR="-I$NETCDF_INC_DIR"
NETCDF_LIBDIR="-Wl,-rpath,$NETCDF_LIB_DIR -L$NETCDF_LIB_DIR"
NETCDF_LIB="-lnetcdff -lnetcdf"

MPI_INCDIR=""
MPI_LIBDIR=""
MPI_LIB="-lcurl"

HDF5_INCDIR="-I$HDF5_INC_DIR"
HDF5_LIBDIR="-Wl,-rpath,$HDF5_LIB_DIR -L$HDF5_LIB_DIR"
HDF5_LIB="-lhdf5_hl -lhdf5 -lhdf5 -lz"
```
Once done, one can compile XIOS as
```shell
./make_xios --arch GCC_LINUX_local --job 16
```
Notice that the name of the architecture chosen, `GCC_LINUX_local` is the same as prescribed by the `.env`, `.fcm` and `.path` files.
## Install ncview (optional but nice to have)
[UDUNITS](https://docs.unidata.ucar.edu/udunits/current/#Unix) must be installed locally, as it is needed each time you open ncview. [Xaw](https://manpages.ubuntu.com/manpages/questing/man3/Xaw.3.html) (also known as Athena Widgets) is not necessary all the time, just at installation time.
```
sudo-g5k apt-get install libxaw7-dev libudunits2-dev
cd $SRCSDIR
wget https://downloads.unidata.ucar.edu/udunits/2.2.28/udunits-2.2.28.tar.gz
tar -zxvf udunits-2.2.28.tar.gz
cd udunits-2.2.28
./configure --prefix=$INSTDIR
make
make install
```
and add the following line to your `$HOME/.bashrc` file
```
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/nemo/installs/lib/"
```
and now ncview
```
wget https://cirrus.ucsd.edu/~pierce/ncview/ncview-2.1.11.tar.gz
tar -zxvf ncview-2.1.11.tar.gz
cd ncview-2.1.11/
./configure --prefix=$INSTDIR --with-nc-config=$INSTDIR/bin/nc-config
make -j1
##make check
make install
```
<p align="right"> <b>Go to the next step: </b></p>
<p align="right"> <a href="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/chapters/01_Install_NEMO.md">Install NEMO v5.0</a> </p>
