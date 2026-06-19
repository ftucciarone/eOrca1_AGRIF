<a href="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/01_Install_NEMO.md" class="previous">&laquo; Previous</a>
<a href="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/03_AGRIF.md" class="next">Next &raquo;</a>

## Download the data repository (courtesy of Casimir de Lavergne [<img style="position:absolute; top:0px;" width="20px" src="https://orcid.org/assets/vectors/orcid.logo.icon.svg" />](https://orcid.org/0000-0001-9267-7390))
> [!WARNING] 
> The author of this repository, Francesco L. Tucciarone, was not involved in the development of the original configuration, thus he shall not be referenced. When using the configuration from this repository, only cite the original work done by Casimir de Lavergne and the other contributors. Minor adjustements were done to port the original configuration (running with NEMO 4.2.2) to NEMO 5.0.1, but they were almost trivial and not enough to grant F.L.T. authorship.

This work is based on the configuration that has been described in
> deLavergne, C., Rathore, S., Madec, G., Sallée, J.-B., Ethé, C., Nasser, A., et al. (2026). Effects of improved tidal mixing in NEMO one-degree global ocean model. Journal of Advances in Modeling Earth Systems, 18, e2024MS004824. 2 February 2026. DOI: [10.1029/2024MS004824](https://doi.org/10.1029/2024MS004824)

whose data can be found in the Zenodo repository:
> de Lavergne C., Rathore S., Madec G., Sallée J.-B., Ethe C., Nasser A., Millet B. and Vancoppenolle M.: _NEMO4.2 eORCA1 configuration files for stable millennial ocean simulations (1.0)_. 2024, [Data set]. Zenodo. https://doi.org/10.5281/zenodo.14041098

As a first step, download the repository with `wget` and unzip it:
```shell
cd $Root_dir
wget https://zenodo.org/records/14041098/files/data_repository.zip
unzip data_repository.zip
```
```
.
└── $Root_dir/
    ├── data_repository/
    │   ├── code/ # Original code for the paper, not of interest for us
    │   ├── initial_conditions/                # Initial conditions for Temperature and Salinity
    │   │   ├── woce_salt_monthly_init_4p2.nc 
    │   │   └── woce_temp_monthly_init_4p2.nc
    │   ├── input_fields/                              # Static files
    │   │   ├── domain_cfg.nc                          # Domain File
    │   │   ├── eddy_viscosity_3D.nc                   # Eddy viscosity (3D)
    │   │   ├── geothermal_heat_flux.nc                # Geothermal Heat Flux
    │   │   ├── merged_ESACCI_BIOMER4V1R1_CHL_REG05.nc # Chlorophill I guess
    │   │   ├── runoff-icb_DaiTrenberth_Depoorter.nc   # River run-off
    │   │   ├── sss_climatology_for_restoring.nc       # Climatology SSS restoring
    │   │   ├── weights_ghflux_bilinear.nc             # On-the-fly interpolation weights 
    │   │   ├── weights_reg05_bilinear.nc              # On-the-fly interpolation weights
    │   │   └── zdfiwm_forcing_*.nc                    # Internal waves mixing (only one needed)
    │   ├── namelists/ # Original namelists for NEMO 4.2.2, not of interest for us
    │   └── restart/ # Restart files 
    │       ├── TRA_10001231_restart_icemod.nc # Ice Model restart
    │       ├── TRA_10001231_restart_trc.nc    # Tracer Model restart
    │       └── TRA_10001231_restart.nc        # Ocean Model restart
    ├── data_repository.zip
    └── nemo-5.0.1/       # Source code for NEMO version X.Y.Z
```



## Compilation of the base cofiguration
0) Define a name for the experiment:
```shell
export Cfg_name=eOrca1_exp01
```
1) The eOrca1 configuration can be built starting from the shipped reference configuration `ORCA2_ICE_PISCES`. First, lets duplicate this configuration with the command
```shell
./makenemo -m 'auto' -r ORCA2_ICE_PISCES -n $Cfg_name -j 0;
```
where `-j 0` sets the number of processors for compilation to 0: with this peculiar choice the command `./makenemo` will only duplicate and rename the necessary files without compiling.

2) Modify the `cpp_*.fcm`: the file `$Root_dir/nemo-5.0.1/cfgs/$Cfg_name/cpp_$(Cfg_name).fcm` should contain the following line
```
bld::tool::fppkeys   key_si3 key_xios key_qco key_isf key_vco_1d3d key_RK3
```

3) Compile the code
```shell
./makenemo -m 'auto' -r ORCA2_ICE_PISCES -n $Cfg_name -j 32;
```

## Linking the static files
First, change the directory to the configuration experiment directory
```shell
cd $Root_dir/nemo-5.0.1/cfgs/$Cfg_name/EXP00/
```
then create a shell executable named `make_links.sh` with the following content:
```shell
#!/bin/bash
static_dir=/home/ftucciarone/ithaca/nemo-ensembleruns/input-eOrca1

#
# Input domain file
#
ln -sf $static_dir/input_fields/domain_cfg.nc .

#
# Input restart files
#
ln -sf $static_dir/restart/*.nc .

# Rivers run-off
ln -sf $static_dir/input_fields/runoff-icb_DaiTrenberth_Depoorter.nc .
# Internal waves mixing
ln -sf $static_dir/input_fields/zdfiwm_forcing_TRA.nc .
# Geothermal heat flux
ln -sf $static_dir/input_fields/geothermal_heat_flux.nc .
# Eddy viscosity (3D)
ln -sf $static_dir/input_fields/eddy_viscosity_3D.nc .
# Climatology SSS restoring
ln -sf $static_dir/input_fields/sss_climatology_for_restoring.nc .
# Chlorophill I guess
ln -sf $static_dir/input_fields/merged_ESACCI_BIOMER4V1R1_CHL_REG05.nc .
#
# On-the-fly interpolation weights
#
ln -sf $static_dir/input_fields/weights_ghflux_bilinear.nc .
ln -sf $static_dir/input_fields/weights_reg05_bilinear.nc .
#
# Initial conditions
#
ln -sf $static_dir/initial_conditions/woce_temp_monthly_init_4p2.nc .
ln -sf $static_dir/initial_conditions/woce_salt_monthly_init_4p2.nc .
#
# Forcings
#
ln -sf $static_dir/forcing_ORCA1/* .
```
make it executable as 
```shell
chmod +x make_links.sh
```
and finally execute it as `./make_links.sh`. This will create links to the static files inside the experiment folder, so that the namelist will find all the necessary data.

<p align="right"> <b>Go to the next step: </b></p>
<p align="right"> <a href="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/03_AGRIF.md">Setup AGRIF</a> </p>
