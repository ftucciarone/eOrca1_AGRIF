# Building an eORCA 1$`^{\circ}`$ configuration with AGRIF zooms
<p align="center">
  <img src="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/figures/med-Orca_global.png" width="400" height="400"/>
  <img src="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/figures/med-Orca_local.png" width="400" height="400"/>
</p>

[AGRIF](https://agrif.imag.fr) (Adaptive Grid Refinement In Fortran) is a library that allows the seamless space and time refinement over rectangular regions in NEMO. Refinement factors can be odd or even (usually lower than 5 to maintain stability). Interaction between grids is “two-way” in the sense that the parent grid feeds the child grid open boundaries and the child grid provides volume/area weighted averages of prognostic variables once a given number of time steps are completed. This page provide guidelines for how to use AGRIF in NEMO. For a more technical description of the library itself, please refer to the [User's guide](https://agrif.imag.fr/agrifusersguide.html) [(pdf)](https://agrif.imag.fr/_downloads/agrifdoc_usersguide.pdf) or the [Reference manual](https://agrif.imag.fr/DoxygenGeneratedDoc/html/index.html) [(pdf)](https://agrif.imag.fr/_downloads/refman.pdf).

**Prerequisites:**
- [Install dependencies](https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/00_Install_dependencies.md)
- [Install NEMO v5.0](https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/01_Install_NEMO.md)
- [eORCA1 configuration setup](https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/02_eOrca1_base)

> [!WARNING]
> In the following, we will make systematically use of some folder location. To ease the process, you are invited to define the main folders are environment variables, like
> ```shell
> ROOT=$HOME
> # Source, installation, xios and work directories
> export SRCSDIR=$ROOT/nemo-deps/sources
> export INSTDIR=$ROOT/nemo-deps/installs
> export XIOSDIR=$ROOT/nemo-deps/XIOS/xios-trunk
> export WORKDIR=$ROOT/nemo-AGRIF
> ```
> so that we can refer to the same directory structure. The directory tree looks like
> ```
> .
> └── $ROOT/                    # Root folder for the project
>     ├── nemo-deps/            # Dependencies for XIOS/NEMO
>     │   ├── sources/          # Sources tarballs ($SRCSDIR)
>     │   ├── installs/         # Installation points ($INSTDIR)
>     │   └── XIOS/             # XIOS base folder (if multiple versions are needed)
>     │       ├── xios-trunk/   # target XIOS dir ($XIOSDIR)
>     │       └── ...           # other XIOS versions to target
>     └── $WORKDIR/         # Work directory for this project
>         ├── input-AGRIF/      # Inputs for AGRIF
>         │   └── ...
>         ├── input-eORCA1/     # Input for eORCA1
>         │   └── ...
>         └── nemo-5.01/        # NEMO 5.0.1 run folder ($NEMODIR)
>             ├── arch/         # Architecture folder
>             ├── cfgs/         # Configurations folder
>             ├── ...
>             ├── tools/        # Tools folder ($TOOLDIR)
>             └── ...
> ```
> The content of each folder explained in the previous (in the case of `nemo-deps` and subfolders) or in the next sections.


# AGRIF
We will refer to the direcory tree illustrated before. In particular, we will make use of the following folders:
```shell
export NEMODIR=$WORKDIR/nemo-5.0.1
export TOOLDIR=$WORKDIR/nemo-5.0.1/tools
```
Moreover, the content of the work direcroty `$WORKDIR` has to be populated with the input needed for the AGRIF tool. In particular, it will look like this 

```
.
└── $ROOT/                # Root folder for the project
    ├── ...             
    ├── $WORKDIR/         # Work directory for this project
    │   └── input-AGRIF/      # Inputs for AGRIF
    │       ├── DOMAINcfg/      # Contains the specific files for the DOMAINcfg tool
    │       ├── eORCA1/         # Contains the namelists to run eORCA with AGRIF
    │       └── restarts/       # Contains the specific files to compute the restarts
    └── ...
```
where `DOMAINcfg` will contain all the necessary files to use the NEMO tool `DOMAINcfg`, `eORCA1` will contain namelists updated to run NEMO eORCA1 with the zooms, while finally `restarts` contains those things needed to create the restart files.
> [!WARNING]
> In reproducing this experience, you might be tempted to skip this last folder, as you probably a;ready have your restart file. This is unfortunately wrong, as AGRIF will not only create new grids, but actually update also the parent grid. For this reason, there might be slight changes in your `domain_cfg.nc` file that renders the new domain incompatible with the new AGRIF simulation, even if you don't switch on the zoom. I don't know how to match old restarts to work with zooms, so the strategy adopted here is to re-do the spinup. Bummer, I know.

## Creating the domain configuration files using `DOMAINcfg` tool:
To compile the DOMAINcfg tool for AGRIF zooms, you need to add the `key_agrif` to your cpp file:
```
$TOOLDIR/DOMAINcfg/cpp_DOMAINcfg.fcm
```
After that you can compile the DOMAINcfg tool using this command:
```
cd $TOOLDIR
./maketools -m [<your_machine>] -n DOMAINcfg
```
you can duplicate the configuration folder in order to make different configuration, in this case we will make three different configurations:
```
cp -r $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO Pacific
cp -r $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO Atlantic_Medsea
cp -r $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO Medsea
```
the first one being a zoom over the Pacific ocean, the second being a double zoom over the Atlantic and Mediterranean sea, the third being almost the same as before but "with less Atlantic" covered. 
### `DOMAINcfg` folder
Inside this folder we will make a copy od the original `domain_cfg.nc` file and we will donwload our reference bathymetry. We will work in this example with [GEBCO 2020 bathymetry](https://www.gebco.net/data-products/gridded-bathymetry-data/gebco-2020#compilations). 
> [!WARNING]
> As stressed multiple times, it is always better to refresh the `domain_cfg.nc` file before running the DOMAINcfg tool. Hence, a bash file called `make_zooms.sh` is created to pipeline the creation of the zooms and avoid mistakes:
> ```shell
> cp $WORKDIR/input-eOrca1/input_fields/domain_cfg.nc .
> ./make_namelist.py
> ./make_domain_cfg.exe
> ```
> then make it executable as `chmod +x make_zooms.sh` and run it as `./make_zoom.sh` when you need to create the AGRIF zooms.



### Define the AGRIF zoom positioning
To run AGRIF you need a configuration file that will define the hierarchy of all the subdomains: `AGRIF_FixedGrids.in`. This file is necessary either to run your model, but also if you need to create the `domain_cfg.nc` files for your models, as will be explained later.
Bellow is how `AGRIF_FixedGrids.in` looks like in the test case example for AGRIF_DEMO:
```txt
2
45 85 52 94 1 1 1
121 146 113 133 4 4 4
0
1
20 60 27 60 3 3 3
0
```
The first line indicates the number of zooms in the parent larger domain (2). The following lines indicate the position of these nested domains in the parent grid (e.g. imin=45, imax=85, jmin=52, jmax=94). The last three values in these lines indicate the horizontal (rx and ry) and time (rt) refinement of each nested model (e.g. rx=4, ry=4, rt=4 for zoom 2). Please note that in this example zoom 1 has the same resolution as the parent, so in this case the refinement is always equals to 1. The following lines indicate the subsequent multiple nesting configuration, if there is. In the zoom 1 there is no embedded zoom domain, so it is 0. Inside zoom 2 there is a nested zoom 3, so we indicate 1 and the following lines follow the same rule as explained above.
The following Python script plots the positioning of a first-level zoom over the parent grid.
```python3
#!/usr/bin/env python3
import os
import numpy as np
import matplotlib.pyplot as plt
import netCDF4 as netcdf
#
# Open domain_cfg.nc eOrca1 (original grid)
eOrca100_gridfile = "/home/ftucciarone/tethys/nemo-AGRIF/nemo-5.0.1/tools/DOMAINcfg/cfgs/AtlaMed/domain_cfg.nc"
eOrca100_grid = netcdf.Dataset(eOrca100_gridfile, "r", format="NETCDF4")

imin = 185
imax = 325
jmin = 208
jmax = 265

# Read bathymetry for a visual check
eOrca100_bathy = eOrca100_grid.variables["bathy_metry"][0]
cropped = eOrca100_bathy[jmin:jmax,imin:imax]
# Visual check
plt.rcParams['figure.dpi'] = 250  
fig, axes = plt.subplots(1, 2)
axes[0].imshow(eOrca100_bathy[::-1,:], cmap="BrBG", interpolation=None)
axes[1].imshow(cropped[::-1,:], cmap="BrBG", interpolation=None)
plt.tight_layout()
plt.show()
```
Once the `AGRIF_FixedGrids.in` is ready, one has to create a consistent set of meshes for the whole nested system. This step ensures that cell volumes agree at the grid interfaces. Volume matching, as well as child bathymetry interpolation from an external database is ensured by the DOMAINcfg tool located in `/tools/DOMAINcfg/`.

### Namelist setup
Parent domain will be defined based on the specifications of the namelist, either by reading a configuration file (ln_read_cfg = .true.) or by defining manually in your namelist (e.g. ppglam0, ppgphi0). Child domain will be defined based on `AGRIF_FixedGrids.in`, with respect to parent grid information.

The first step is thus to modify the `namelist_cfg` file to adapt it to our needs. In this list, `cn_domcfg` is the name of the input/output grid file, `cn_topo` is the external topography file, `cn_bath` is the name of the bathymetry variable in the netCDF file, with `cn_lat` and `cn_lon` the names of the latitude and longitude variables int he netCDF file. 
```fortran
!-----------------------------------------------------------------------
&namdom        !   space and time domain (bathymetry, mesh, timestep)
!-----------------------------------------------------------------------
   ln_read_cfg = .true.
   nn_bathy    =    1      ! = 0 compute analyticaly
                           ! = 1 read the bathymetry file
                           ! = 2 compute from external bathymetry
                           ! = 3 compute from parent (if "key_agrif")
   nn_interp   =    1              ! type of interpolation (nn_bathy =2)
   cn_domcfg   =  'domain_cfg.nc'  ! external grid file         
   cn_topo     =  'GEBCO_2020.nc'  ! external topo file (nn_bathy =2)
   cn_bath     =  'elevation'      ! topo name in file  (nn_bathy =2)
   cn_lon      =  'lon'            ! lon  name in file  (nn_bathy =2)
   cn_lat      =  'lat'            ! lat  name in file  (nn_bathy =2)
   rn_scale    = 1
   rn_bathy    =    0.     !  value of the bathymetry. if (=0) bottom flat at jpkm1
   jphgr_msh   =       0               !  type of horizontal mesh
   ppglam0     =  999999.0             !  longitude of first raw and column T-point (jphgr_msh = 1)
   ppgphi0     =  999999.0             ! latitude  of first raw and column T-point (jphgr_msh = 1)
   ppe1_deg    =  999999.0             !  zonal      grid-spacing (degrees)
   ppe2_deg    =  999999.0             !  meridional grid-spacing (degrees)
   ppe1_m      =  999999.0             !  zonal      grid-spacing (degrees)
   ppe2_m      =  999999.0             !  meridional grid-spacing (degrees)
   ppsur       =   -3958.951371276829  !  ORCA r4, r2 and r05 coefficients
   ppa0        =     103.9530096000000 ! (default coefficients)
   ppa1        =       2.415951269000000   !
   ppkth       =      15.35101370000000    !
   ppacr       =       7.0             !
   ppdzmin     =  999999.0             !  Minimum vertical spacing
   pphmax      =  999999.0             !  Maximum depth
   ldbletanh   =   .TRUE.              !  Use/do not use double tanf function for vertical coordinates
   ppa2        =     100.7609285000000 !  Double tanh function parameters
   ppkth2      =      48.02989372000000    !
   ppacr2      =      13.00000000000   !
/
```
The bathymetry for each of the grids will be computed based on the information you put in `nn_bathy`:
```fortran
nn_bathy = 1  ! = 0 compute analyticaly
              ! = 1 read the bathymetry file
              ! = 2 compute from external bathymetry
              ! = 3 compute from parent (if "key_agrif")
```

* For the option `nn_bathy = 1` the bathymetry needs to be already interpolated to the model grid. For the parent grid, if you provide a coordinates file (`ln_read_cfg = .true.`), longitude and latitude definitions will follow this file. If you set `ln_read_cfg = .false.`, the coordinates of the output `domain_cfg.nc` will follow the definitions in the `namelist_cfg`, where you define, for example, ilon and ilat (`ppglam0, ppgphi0`),  grid spacing (`ppe1_deg, ppe2_deg`), etc.
Be aware that when creating the child bathymetry, if you set `nn_bathy = 1`, the bathymetry you are reading need to match the exact domain size and position defined in the `AGRIF_FixedGrids.in`. The tool in this case will not consider the latitude and longitude of the bathymetry provided, but instead it will just overlap that region provided from the file to the domain specifications defined in `AGRIF_FixedGrids.in`.

* The choice `nn_bathy = 2` will read a bathymetry file and interpolate it to the model grid. 

* With `nn_bathy = 3` the tool will simply compute the bathymetry from the parent one, in the region defined in `AGRIF_FixedGrids.in`, without reading any external file. In this procedure, however, no interpolation is performed, and the parent bathymetry is just reshaped with more grid points.

The parameters from `jphgr_msh` to `ppacr2` can be chosen from those listed in `README_configs_namcfg_namdom`

```fortran
!-----------------------------------------------------------------------
&namcfg        !   parameters of the configuration
!-----------------------------------------------------------------------
   !
   ln_e3_dep   = .true.    ! =T : e3=dk[depth] in discret sens.
   !                       !      ===>>> will become the only possibility in v4.0
   !                       ! =F : e3 analytical derivative of depth function
   !                       !      only there for backward compatibility test with v3.6
      !                      ! if ln_e3_dep = T
      ln_dept_mid = .true.   ! =T : set T points in the middle of cells
   !                       !
   cp_cfg      =  "orca"   !  name of the configuration
   jp_cfg      =       1   !  resolution of the configuration
   jpidta      =     360   !  1st lateral dimension ( >= jpi )
   jpjdta      =     331   !  2nd    "         "    ( >= jpj )
   jpkdta      =      75   !  number of levels      ( >= jpk )
   Ni0glo      =     360   !  1st dimension of global domain --> i =jpidta
   Nj0glo      =     331   !  2nd    -                  -    --> j  =jpjdta
   jpkglo      =      75
   jperio      =       4   !  lateral cond. type (between 0 and 6)
   ln_domclo = .false.     ! computation of closed sea masks (see namclo)
/
!-----------------------------------------------------------------------
&namzgr        !   vertical coordinate                                  (default: NO selection)
!-----------------------------------------------------------------------
!-----------------------------------------------------------------------
   ln_zco      = .false.   !  z-coordinate - full    steps
   ln_zps      = .true.   !  z-coordinate - partial steps
   ln_sco      = .false.   !  s- or hybrid z-s-coordinate
   ln_isfcav   = .false.   !  ice shelf cavity             (T: see namzgr_isf)
/
```
Once done, make the child `namelist_cfg` with
```shell
./make_namelist.py
```
ans then run `DOMAINcfg` with 
```shell
./make_domain_cfg.exe
```
Finally, copy all the important files into the `EXP00` folder of your configuration
```shell
cp $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO/domain_cfg.nc .
cp $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO/AGRIF_FixedGrids.in .
```

# [Pacific Ocean refinement](https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/input-AGRIF/zoom_Pacific/Pacific.md)
# [Atlantic-Mediterranean refinement](https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/input-AGRIF/zoom_MedAtlantic/MedAtlantic.md)
# [Mediterranean Sea refinement](https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/input-AGRIF/zoom_MedORCA/MedORCA.md)
