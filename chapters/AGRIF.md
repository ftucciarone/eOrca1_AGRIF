<a href="xxx.html">Back</a>|<a href="xxx.html">Next</a>



# AGRIF

#### Prequel: Folder setup
```shell
export WORKDIR=/home/ftucciarone/tethys/nemo-AGRIF
export NEMODIR=/home/ftucciarone/tethys/nemo-AGRIF/nemo-5.0.1
export TOOLDIR=/home/ftucciarone/tethys/nemo-AGRIF/nemo-5.0.1/tools
```
#### Safety measures:
**Always copy `domain_cfg.nc` and bathymerty file into `DOMAINcfg` tool folder**, that is 
```shell
cp $WORKDIR/input-eOrca1/input_fields/domain_cfg.nc $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO/
cp $WORKDIR/input-AGRIF/GEBCO_2020.nc $TOOLDIR/DOMAINcfg/cfgs/AGRIF_DEMO/
```
#### AGRIF Grids setup
For this example, the `AGRIF_FixedGrids.in` file will read
```shell
1
108 208  65 165 1 1 1
0
```

## Creating the domain configuration files using `DOMAINcfg` tool:

Once the `AGRIF_FixedGrids.in` is ready, one has to create a consistent set of meshes for the whole nested system. This step ensures that cell volumes agree at the grid interfaces. Volume matching, as well as child bathymetry interpolation from an external database is ensured by the DOMAINcfg tool located in `/tools/DOMAINcfg/`.


To compile the DOMAINcfg tool for AGRIF zooms, you need to add the `key_agrif` to your cpp file:
```
/path/to/nemo-5.0.1/tools/DOMAINcfg/cpp_DOMAINcfg.fcm
```
After that you can compile the DOMAINcfg tool using this command:
```
./maketools -m [<your_machine>] -n DOMAINcfg
```
you can access the configuration folder with
```
cd /path/to/nemo-5.0.1/tools/tools/DOMAINcfg/cfgs/AGRIF_DEMO/
```
in this folder, you shall **copy** the `domain_cfg.nc` of your configuration and the external bathymetry file if needed (in this example, `GEBCO_2020.nc`).
> :warning:	**WARNING** 
   DOMAINcfg will **overwrite** the original `domain_cgf.nc` file, so you shall **NEVER link the original** into the folder of DOMAINcfg but rather **make a hard copy**.

#### Namelist setup
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
## Templates for deciding the positioning of the zoom
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

# Pacific Ocean refinement

# Atlantic Ocean refinement

# Atlantic-Mediterranean refinement
Set up the `DOMAINcfg` configuration as
```shell
cp -r ./tools/DOMAINcfg/cfgs/AGRIF_DEMO ./tools/DOMAINcfg/cfgs/AtlaMed
```
copy the `domain_cfg.nc` file inside the configuration folder, link the `GEBCO_2020.nc` file in 
```shell
cp $WORKDIR/input-eOrca1/input_fields/domain_cfg.nc $TOOLDIR/DOMAINcfg/cfgs/AtlaMed/
ln -sf $WORKDIR/input-AGRIF/GEBCO_2020.nc $TOOLDIR/DOMAINcfg/cfgs/AtlaMed/
```
and define the `AGRIF_FixedGrids.in` as
```shell
1
185 330 208 265 4 4 4
1
360 567 98 195 3 3 3
0
```
This setup produces the following bathymetry files.
<p align="center">
  <img src="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/figures/output_AtlaMed.png" />
</p>

