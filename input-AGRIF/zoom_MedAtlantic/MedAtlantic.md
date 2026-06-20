# Med12-Atlantic4 configuration
<p align="center">
  <img src="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/figures/output_AtlaMed.png" />
</p>
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
This setup produces the bathymetry files shown at the top of the page.






### Some useful scripts
Visualization of `DOMAINcfg` products
```python
#!/usr/bin/env python3
import os
import cmocean
import numpy as np
import matplotlib as mpl
import matplotlib.pyplot as plt
import matplotlib.patches as patches
from matplotlib.colors import LogNorm
import netCDF4 as netcdf

# make a nice colormap for the ocean with masked land
ocean_depth = plt.colormaps["terrain"](np.linspace(0, 0.17, 256))
land_color = np.array([215/256, 211/256, 200/256, 0.75])
ocean_depth[128, :] = land_color
ocean_depth = mpl.colors.LinearSegmentedColormap.from_list('terrain_map', ocean_depth)
divnorm = mpl.colors.TwoSlopeNorm(vmin=-5000., vcenter=0, vmax=1)

base_dir = "/home/ftucciarone/tethys/nemo-AGRIF/nemo-5.0.1/tools/DOMAINcfg/cfgs/AtlaMed/"
titles = ["eORCA 1$^{\circ}$", "Atlantic 1/4$^{\circ}$", "Mediterranean 1/12$^{\circ}$"]

imin, imax, jmin, jmax = [], [], [], []
with open(base_dir + "AGRIF_FixedGrids.in") as fp:
    line = fp.readline()
    grid_count = 1
    while line:
        if len(line.split()) >=4 :
       	    line_split = line.split()[0:4]
            imin.append(int(line_split[0]))
            imax.append(int(line_split[1]))
            jmin.append(int(line_split[2]))
            jmax.append(int(line_split[3]))
            grid_count += 1
        line = fp.readline()

# Visual check
plt.rcParams['figure.dpi'] = 1000  
plt.rcParams['figure.autolayout'] = True

fig, axes = plt.subplot_mosaic([['left', 'upper right'],
                               ['left', 'lower right']],
                            #   figsize=(7.5, 3.5), layout="constrained")
                              figsize=(15, 7), layout="constrained")

for idx in range(grid_count):
    if idx == 0:
        grid = netcdf.Dataset(base_dir + "domain_cfg.nc", "r", format="NETCDF4")
    else:
        grid = netcdf.Dataset(base_dir + str(idx) + "_domain_cfg.nc", "r", format="NETCDF4")
    grid_nx = len(grid.dimensions["x"])
    grid_ny = len(grid.dimensions["y"])
    grid_bathy = grid.variables["bathy_metry"][0]
    im = fig.axes[idx].imshow(-grid_bathy[::-1,:], cmap=ocean_depth, norm=divnorm, interpolation=None)

    if idx < len(grid_files)-1:
    # Create a Rectangle patch
        rect = patches.Rectangle((imin[idx], grid_ny-jmax[idx]), imax[idx]-imin[idx], jmax[idx]-jmin[idx], linewidth=0.75, edgecolor='r', facecolor='none')
        # Add the patch to the Axes
        fig.axes[idx].add_patch(rect)

    #
    # Plot specs
    for ticks, ticks_width in zip(['major', 'minor'],['0.25', '0.125']):
        fig.axes[idx].tick_params(which=ticks, 
                                  label1On=False, 
                                  width=ticks_width,
                                  direction='in',
                                  top=False, bottom=False, left=False, right=False)
    for axis in ['top', 'bottom', 'left', 'right']:
        fig.axes[idx].spines[axis].set_linewidth(0.5)

    try:
        titles
    except NameError:
        None
    else:
        if titles: fig.axes[idx].set_title(titles[idx], fontsize=20)

plt.show()
```
