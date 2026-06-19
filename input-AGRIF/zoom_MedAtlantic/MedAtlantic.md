# Med12-Atlantic4 configuration

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
  <img src="https://github.com/ftucciarone/eOrca1_AGRIF/blob/main/docs/figures/output_AtlaMed.png" />
</p>
