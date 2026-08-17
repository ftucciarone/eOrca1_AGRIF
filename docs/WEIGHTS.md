# WEIGHTS user guide

`WEIGHTS` is a standalone NEMO tool (`tools/WEIGHTS`) that generates
on-the-fly interpolation (IOF) weight files. These weight files let NEMO
read surface-forcing data (atmospheric fields, runoff, geothermal heat
flux, SSS-restoring climatologies, ...) on their own native grid and
interpolate it to the model grid at runtime, instead of requiring the
forcing data to be pre-interpolated offline. The tool is built on the
Spherical Coordinate Remapping and Interpolation Package (SCRIP, from Los
Alamos National Laboratory), vendored under `tools/WEIGHTS/SCRIP1.4`.

A weight file is specific to one *pair* of grids (one source forcing grid,
one target model grid) and one interpolation method (bilinear or bicubic).
For an AGRIF configuration, every grid (parent and each child nest) needs
its own weight file per interpolated field, dimensioned to that grid's own
size — this is why weight files, unlike the raw forcing data itself, get
the AGRIF nest-number prefix (`N_weights_*.nc`) and must be regenerated for
every new grid.

## Building the tool

No special preprocessor keys are needed — `WEIGHTS` doesn't need to know
about AGRIF nesting at compile time at all; it treats every grid (parent or
child) as just another target grid to generate weights for:

```shell
cd tools
./maketools -n WEIGHTS -m auto
```

This produces four executables in `tools/WEIGHTS/BLD/bin/`:
`scripgrid.exe`, `scrip.exe`, `scripshape.exe`, and `scripinterp.exe`.

## Setting up a run directory

The executables and the namelist templates need to be reachable from
whatever directory the generation is actually run in — they're not
automatically on `PATH`, so it's normal to symlink them in:

```shell
mkdir -p <weights_run_dir> && cd <weights_run_dir>
ln -sf /path/to/tools/WEIGHTS/scripgrid.exe .
ln -sf /path/to/tools/WEIGHTS/scrip.exe .
ln -sf /path/to/tools/WEIGHTS/scripshape.exe .
ln -sf /path/to/tools/WEIGHTS/namelist_bilin .
ln -sf /path/to/tools/WEIGHTS/namelist_bicub .
```

## Generating weights for your configuration

Producing weights field-by-field and grid-by-grid by hand doesn't scale
past a couple of fields. The shipped `tools/WEIGHTS/create_weights.py`
automates the pattern: for every `?_domain_cfg.nc` found in a configured
directory (i.e. every AGRIF grid present there, `?` matching the
single-digit nest number), and for every entry in a configured list of
forcing fields, it fills in a copy of the right template namelist and
drives the underlying tools through to a finished weight file for that
grid/field/method combination.

Its user-editable header is the actual thing to change to get weights for
your own configuration. The full script, shown here with the interpolation-
method condition fixed (see the warning below):
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import subprocess
import re
from netCDF4 import Dataset

# Root_dir is the same exported variable set at the top of this document
Root_dir = os.environ["Root_dir"]

#
# BEGIN USER MODIFICATIONS
#

# Directory with domccfg target file
DOMCFG_DIR=f"{Root_dir}/nemo-5.0.1/tools/DOMAINcfg/cfgs/Pacific"
# Suffix of domcfg files
RAD='domain_cfg.nc'
# Directory with original forcing on native grid
FORCING_DIR=f"{Root_dir}/input"
# Forcing file names, interpolation method (default bilin), and weigth file name (optional), lon(optional), lat(optional)  
FILES=[
['forcing_ORCA1/u_10.15JUNE2009_fill.nc'              , 'bicub', 'weights_coreII_2_eORCA1.4.2_bicubic.nc'  , '', ''],
['forcing_ORCA1/ncar_rad.15JUNE2009_fill.nc'          , 'bilin', 'weights_coreII_2_eORCA1.4.2_bilinear.nc' , '', ''],
['input_fields/eddy_viscosity_3D.nc'                  , 'bilin', 'weights_eddy_visc_bilinear.nc' ,'',''],
['input_fields/geothermal_heat_flux.nc'               , 'bilin', 'weights_ghflux_bilinear.nc'    ,'',''],
['input_fields/merged_ESACCI_BIOMER4V1R1_CHL_REG05.nc', 'bilin', 'weights_reg05_bilinear.nc'     ,'',''],
['input_fields/runoff-icb_DaiTrenberth_Depoorter.nc'  , 'bilin', 'weights_runoff-icb_bilinear.nc' ,'',''],
['input_fields/sss_climatology_for_restoring.nc'      , 'bilin', 'weights_sss_clima_bilinear.nc' ,'',''],
['input_fields/sss_climatology_for_restoring.nc'      , 'bicub', 'bicub_sss_climatology_for_restoring.nc' ,'',''],
['input_fields/zdfiwm_forcing_TRA.nc'                 , 'bilin', 'weights_zdfiwm_bilinear.nc' ,'',''],
['initial_conditions/woce_salt_monthly_init_4p2.nc'   , 'bilin', 'weights_woce_salt_bilinear.nc' ,'',''],
['initial_conditions/woce_temp_monthly_init_4p2.nc'   , 'bilin', 'weights_woce_temp_bilinear.nc' ,'','']
]

#
# END USER MODIFICATIONS
#
mycmd="ls "+DOMCFG_DIR+"/?_"+RAD
returned_output = subprocess.check_output(mycmd, shell=True)
listcfg = (returned_output.decode("utf-8")).split()



for i in range(len(listcfg)):
	print ('Computing weights for cfg file %s :' % listcfg[i])
	print()

	for myfile in FILES:
		print('Input file is %s with %s interpolation' % (myfile[0],  myfile[1]))
		if len(myfile[2]) == 0 :
			wfile=str(i+1)+'_'+myfile[1]+'_'+myfile[0]
		else:
			wfile=str(i+1)+'_'+myfile[2]
		print('   Performing weights computation ...')
		if myfile[1]=='bicub':
			namelist='namelist_bicub'
		else :
			namelist='namelist_bilin'

		myfilename=FORCING_DIR+'/'+myfile[0]
		dataset=Dataset(myfilename)
		
#		if len(myfile[3])==0: 
#			for key in dataset.variables:
#				if len(dataset.variables[key].dimensions) >=2:
#					myvar=key
#					break
#		else:
#			myvar=myfile[3]
#		print('   Interpolation based on variable %s ...' % myvar)		
		
		if len(myfile[3])==0 or len(myfile[4])==0: 
			for key in dataset.variables:
				if re.search('lat',key.lower()):
					mylat=key
				if re.search('lon',key.lower()):
					mylon=key
		else:
			mylon=myfile[3]
			mylat=myfile[4]
		print('   Interpolation based on longitude %s ...' % mylon)		
		print('   Interpolation based on latitude  %s ...' % mylat)		

		f2=open("namelist_new","w+")
		with open(namelist,"r") as f:
			for line in f:
				match = re.search('nemo_file',line)
				match2 = re.search('input_file',line)
				match3 = re.search('input_lon',line)
				match4 = re.search('input_lat',line)
				match5 = re.search('output_file',line)
				match6 = re.search('output_name',line)
				if match != None :
					line='nemo_file=''\''+listcfg[i]+'\''"\n"
				if match2 != None :
					line='input_file=''\''+myfilename+'\''"\n"
				if match3 != None :
					line='input_lon=''\''+mylon+'\''"\n"
				if match4 != None :
					line='input_lat=''\''+mylat+'\''"\n"
				if match5 != None :
					line='output_file=''\''+wfile+'\''"\n"
			#	if match5 != None :
		##			line='output_name=''\''+myvar+'\''"\n"
				f2.write(line)
			f.close()
			f2.close()
		mycheck=subprocess.check_output('./scripgrid.exe namelist_new',shell=True	)
		mycheck=subprocess.check_output('./scrip.exe namelist_new',shell=True	)
		mycheck=subprocess.check_output('./scripshape.exe namelist_new',shell=True	)
		print('   Success ...')
		print('   => weight file is %s' % wfile)

	print()
```
> [!WARNING]
> The condition above (`if myfile[1]=='bicub':`) is a local fix, not
> upstream. The version of `create_weights.py` as shipped with NEMO reads
> `if myfile[1]=='namelist_bicub':` — comparing the method tag against a
> namelist filename it can never equal, so it's always false and every
> field silently gets bilinear weights regardless of what `FILES`
> requests (see "Wrong interpolation method silently used" below). This
> is a genuine upstream bug, still present in the shipped script — file a
> bug report against it rather than assuming this local fix is already
> known.

Each `FILES` entry is `[input_filename, method, weight_filename,
lon_var, lat_var]`:
- `method` is `'bilin'` or `'bicub'`, selecting `namelist_bilin` or
  `namelist_bicub` as the template for that field.
- `weight_filename`, if left empty, defaults to
  `<nest_number>_<method>_<input_filename>` — but in practice this almost
  always needs to be set explicitly, to match whatever weight file name
  the target `namelist_cfg`'s `sn_*` forcing entry actually expects
  (`wgtname`). A mismatch here means NEMO reports the weight file as not
  found at runtime even though a weight file — just under a different
  name — was generated.
- `lon_var`/`lat_var`, if left empty, are auto-detected from the input
  file by scanning variable names for `lat`/`lon` substrings.

`DOMCFG_DIR` is the one setting that's genuinely specific to your
configuration — it must point at the directory containing the
`N_domain_cfg.nc` file(s) for every grid you want weights for (parent
included, if it also has one there). `FORCING_DIR` and `FILES` describe
the *parent* configuration's forcing requirements and stay the same
regardless of how many child grids you're generating weights for — the
script runs the same `FILES` list once per `?_domain_cfg.nc` it finds, so
adding a new grid to `DOMCFG_DIR` is enough to also get weights for it,
without touching `FILES`.

Beyond this header, the namelist templates (`namelist_bilin`/
`namelist_bicub`) themselves may also need a one-time check for your
configuration — in particular `ew_wrap` (east-west cyclicity of the
*source* forcing grid) and the mask settings, neither of which the script
fills in automatically. See "Namelist reference" below for what each field
controls.

## Common failure modes

### Missing `FILES` entry for a field the model actually needs

If a grid's `namelist_cfg` requests on-the-fly interpolation for a field
(via a `sn_*` entry with a weight file name set) but no matching entry
exists in `FILES`, no weight file gets generated for it. NEMO fails at
runtime with a `File ... not found` error for that specific weight file —
loud and immediate.

### Wrong interpolation method silently used

This is the dangerous one, because it does **not** crash: it produces a
weight file that opens and reads back fine, just computed with the wrong
method. Watch for a script that decides which template to use based on
a comparison against the literal string `'namelist_bicub'` rather than
against the method value actually stored in the `FILES` entry (`'bicub'`)
— the shipped `create_weights.py` has exactly this bug (`if
myfile[1]=='namelist_bicub':`, always false), so every field silently gets
bilinear weights regardless of what `FILES` requested.

The reliable way to catch this is to check the actual weight file's
content, not its filename: count the `src##`/`dst##`/`wgt##` variable
triples. A genuinely bilinear weight file has exactly 4 (`src01`..`src04`);
a genuinely bicubic one has 16 (`src01`..`src16`):
```python
from netCDF4 import Dataset
d = Dataset('some_weights_file.nc')
n = sum(1 for name in d.variables if name.startswith('src'))
print('interpolation sets:', n, '->', 'bilinear' if n == 4 else 'bicubic' if n == 16 else 'unexpected')
```

### Stale or wrong-size weight file ("start and count too big")

Weight files are dimensioned to the *target* (NEMO) grid, not the source
forcing data's grid. If a weight file left over from an earlier/smaller
grid gets reused for a different grid, NEMO fails at runtime with an
`iom_get_123d ... start and count too big` error — the file on disk is
smaller than what the currently active grid needs. Compare the weight
file's own dimensions against the target grid's `domain_cfg.nc`:
```python
from netCDF4 import Dataset
w = Dataset('some_weights_file.nc')
d = Dataset('target_domain_cfg.nc')
print('weight file lon/lat:', w.dimensions['lon'].size, w.dimensions['lat'].size)
print('target grid   x/y  :', d.dimensions['x'].size, d.dimensions['y'].size)
```

### Weight generation hangs on re-runs

The final generation stage prompts interactively when its output file
already exists, and blocks on stdin waiting for an answer. In an
automated/looped generation script, either delete existing output files
first, or feed the confirmation programmatically (e.g. piping `y`) —
otherwise a batch run over many fields silently stalls at the first field
that was already generated in a previous attempt. See "The SCRIP
pipeline, explained" below for exactly which stage this is and why.

### `nemo_lon`/`nemo_lat` not named `glam*`/`gphi*`

The grid-description stage derives the corner-point coordinate variables
it needs from the *names* of the center-point ones configured as
`nemo_lon`/`nemo_lat` — they must literally start with `glam`/`gphi` (e.g.
`glamt`/`gphit`), or the corner lookup fails. This is a naming convention
of the tool itself, not something to work around by renaming variables in
the grid file.

## After the weights file exists

A generated weight file is only useful once the target NEMO configuration
actually points at it: the relevant `sn_*` entry in that grid's
`namelist_cfg` (in whichever `SBC` namelist block covers that field) needs
its weight-file name set to match the actual output filename produced
here. For an AGRIF child grid, that file also needs to be linked into the
run directory under its nest-prefixed name (`N_<weight_filename>.nc`) —
the raw forcing data file itself stays unprefixed and shared across every
grid; only the weight file is per-grid.

## The SCRIP pipeline, explained

Everything above is enough to generate weights for a real configuration
without knowing how the tool works internally. This section explains what
`create_weights.py` (or a by-hand run) is actually driving underneath —
useful for making sense of an error that names one of these executables
directly, or for running a stage by hand.

Producing one weight file is a three-step pipeline, each stage consuming
the previous stage's output. All three read their configuration from the
same SCRIP-style namelist file (conventionally `namelist_bilin` or
`namelist_bicub`), each looking only at the block(s) relevant to it.

### 1. `scripgrid.exe`: describe both grids

Reads the `&grid_inputs` namelist block. It needs two things in the
current directory: `input_file` (a forcing-data file with longitude/
latitude coordinate variables) and `nemo_file` (a NEMO grid file —
typically a `domain_cfg.nc`, since it contains `glamt`/`gphit`; despite the
name, it does not need to be literally called `coordinates.nc`). It reads
the named center-point longitude/latitude variables from `nemo_file`
(`nemo_lon`/`nemo_lat`, conventionally `'glamt'`/`'gphit'`) and derives the
matching corner coordinates automatically — which is why those variable
names must start with the literal four characters `glam`/`gphi`: `scripgrid`
looks for the corresponding corner-point variables by construction from
that prefix (e.g. naming `glamt` also makes `glamf`/`gphif` discoverable).
It produces two remap grid description files (`datagrid_file`,
`nemogrid_file`), one for each grid, in the format `scrip.exe` expects.

This step is independent of the interpolation method — the same pair of
remap grid files works for both a bilinear and a bicubic weight file
between the same two grids, so it only needs to be run once per grid pair,
not once per method.

### 2. `scrip.exe`: compute the weights

Reads `&remap_inputs`. Takes the two remap grid files from step 1 and
computes the actual interpolation weights between them, writing the result
to `interp_file1` (and, if `num_maps=2`, the reverse mapping to
`interp_file2`). `map_method` selects `'bilinear'`, `'bicubic'`, or
`'conservative'` (the last one not tested/supported for NEMO grids).

### 3. `scripshape.exe`: reshape into NEMO's expected layout

Reads `&shape_inputs`. Takes `interp_file` from step 2 and rearranges its
source/destination indices and weights into a set of 2D fields spanning
the *target* (NEMO) grid — one `src##`/`dst##`/`wgt##` triple per
interpolation stencil point: **4 sets** (`src01`..`src04`) for a bilinear
mapping, **16 sets** (`src01`..`src16`) for a bicubic one. This is the
final file — `output_file` — that gets used as a weight file in NEMO's own
namelist. It also stamps an `ew_wrap` attribute recording the source
grid's east-west cyclicity (`-1` = not cyclic, `0` = cyclic with no
overlapping columns, `>0` = number of overlapping columns) — only actually
used by bicubic interpolation, which needs to know which extra columns to
read for gradient calculations.

This is the stage responsible for the interactive re-run prompt mentioned
above: if `output_file` already exists, it asks —
```
Output file: <name> exists
Ok to overwrite (y/n)?
```
— and blocks on stdin waiting for an answer.

(A fourth executable, `scripinterp.exe`, applies an already-computed
weight file to interpolate one field directly — a one-off diagnostic tool,
not part of the weight-generation pipeline itself.)

## Namelist reference

The shipped templates (`tools/WEIGHTS/namelist_bilin` /
`tools/WEIGHTS/namelist_bicub`) bundle all three blocks above in one file,
since in practice all three programs are run against the same namelist in
sequence:

```fortran
&grid_inputs
    input_file = 't_10.15JUNE2009_fill.nc'
    nemo_file = 'coordinates_nordic1.nc'
    datagrid_file = 'remap_core2_grid.nc'
    nemogrid_file = 'remap_nordic1_grid.nc'
    method = 'regular'
    input_lon = 'lon'
    input_lat = 'lat'
    nemo_lon = 'glamt'
    nemo_lat = 'gphit'
    nemo_mask = 'none'
    nemo_mask_value = 10
    input_mask = 'none'
    input_mask_value = 10
/
&remap_inputs
    num_maps = 1
    grid1_file = 'remap_core2_grid.nc'
    grid2_file = 'remap_nordic1_grid.nc'
    interp_file1 = 'core2_nordic1_bilin.nc'
    interp_file2 = 'nordic1_core2_bilin.nc'
    map1_name = 'orca2 to nordic1 bilin Mapping'
    map2_name = 'nordic1 to orca2 bilin Mapping'
    map_method = 'bilinear'
    normalize_opt = 'frac'
    output_opt = 'scrip'
    restrict_type = 'latitude'
    num_srch_bins = 90
    luse_grid1_area = .false.
    luse_grid2_area = .false.
/
&shape_inputs
    interp_file = 'core2_nordic1_bilin.nc'
    output_file = 'weights_core2_nordic1_bilin.nc'
    ew_wrap     = 0
/
```

The `namelist_bicub` template is identical in shape, with `map_method =
'bicubic'` and matching file names.

#### Options in `&grid_inputs`

- **`input_file`** — a source forcing-data file, used only for its
  longitude/latitude coordinate variables.
- **`nemo_file`** — the target NEMO grid file (a `domain_cfg.nc`, or
  historically `coordinates.nc`).
- **`datagrid_file`**, **`nemogrid_file`** — output remap grid description
  files, one per grid, consumed by `scrip.exe` next.
- **`method`** — only `'regular'` (cartesian/regular grid) is implemented.
- **`input_lon`**, **`input_lat`** — longitude/latitude variable names in
  `input_file`.
- **`nemo_lon`**, **`nemo_lat`** — longitude/latitude variable names in
  `nemo_file`; must start with `glam`/`gphi` respectively so the matching
  corner-point variables can be derived automatically.
- **`nemo_mask`**, **`input_mask`**, **`*_mask_value`** — optional masks;
  `'none'` disables masking. The tool assumes unmasked forcing data by
  default — if the source is masked, producing a properly filled/unmasked
  version on the source grid first is the user's responsibility, since
  masked land values would otherwise contaminate ocean values near
  coastlines after interpolation.

#### Options in `&remap_inputs`

- **`num_maps`** — `1` for a one-way mapping (source → target), `2` to
  also compute the reverse mapping.
- **`grid1_file`**, **`grid2_file`** — the two remap grid files produced by
  `scripgrid.exe`.
- **`interp_file1`**, **`interp_file2`** — output weight-map file(s), in
  SCRIP's own intermediate format (not yet NEMO's final layout).
- **`map1_name`**, **`map2_name`** — free-form descriptive labels.
- **`map_method`** — `'bilinear'`, `'bicubic'`, or `'conservative'` (the
  last untested for NEMO grids).
- **`normalize_opt`** — normalization of the computed weights; `'frac'`
  is the usual choice, otherwise scaling has to be done manually (this
  matters in particular for fields like fractional ice cover).
- **`output_opt`** — `'scrip'` or `'ncar-csm'` output layout for the
  intermediate file.
- **`restrict_type`**, **`num_srch_bins`** — restrict the neighbor search
  to `'latitude'` or `'latlon'` bins, for performance on large grids.
- **`luse_grid1_area`**, **`luse_grid2_area`** — override SCRIP's own area
  calculation with areas supplied in the input files, for cases where the
  model's own area computation would otherwise disagree slightly.

#### Options in `&shape_inputs`

- **`interp_file`** — the intermediate weight-map file from `scrip.exe`.
- **`output_file`** — the final, NEMO-ready weights file.
- **`ew_wrap`** — east-west cyclicity of the *source* grid: `-1` not
  cyclic, `0` cyclic with no overlap, `>0` number of overlapping columns.
  Only affects bicubic interpolation's gradient calculation.
