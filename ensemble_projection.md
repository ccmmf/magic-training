# Today: From hindcast inventories to projections

## Plan

First, what components need to change?

* Start and end dates, obviously
* Retrieve new met drivers, update configured paths to them
* Retrieve projected event files, update configured paths

If we only do these steps, we'll have a run that reports success but starts its
run in 2024 using initial conditions computed for 2016, and all ensemble
parameters will be freshly sampled each time so there's no way to meaningfully
connect a specific projection run to a specific hindcast run.
To align them, we'll need a few additional components:

* Save model state ("restart.out" files) from each matching hindcast run
* Use the same parameter samples for matched ensemble members in hindcast and
	forecast phases.

## Run setup

```sh
export AWS_PROFILE=magic
export CONDA_DIR=/project/60007/cblack/.conda/envs
conda activate ${CONDA_DIR}/pecan-all-1.17

export DEMO_DIR=/project/60007/cblack/ensemble_demo_20260909
export INVY_DIR=${DEMO_DIR}/workflows/run-inventory
export PROJN_DIR=${DEMO_DIR/workflows/run-projections


mkdir -p "$DEMO_DIR" && cd "$DEMO_DIR"
git clone https://github.com/ccmmf/workflows.git
cd workflows
# git checkout main
git fetch  && git checkout carb-demo-20260909
mkdir -p "$INVY_DIR"
mkdir -p "$PROJN_DIR"
```

## Fetch demo data

```sh
cd 
aws s3 sync s3://carb/management/crops/ data_raw/management/crops/
aws s3 sync s3://carb/management/harvest/ data_raw/management/harvest/
aws s3 sync s3://carb/management/planting/ data_raw/management/planting/
aws s3 sync s3://carb/management/phenology/ data_raw/management/phenology/
aws s3 sync s3://carb/management/tillage/ data_raw/management/tillage/
aws s3 sync s3://carb/management/irrigation/ data_raw/management/irrigation/
aws s3 sync s3://carb/management/fertilization/ data_raw/management/fertilization/
aws s3 sync s3://carb/management/ncc/ data_raw/management/ncc/
aws s3 sync s3://carb/met/ data_raw/met
aws s3 sync s3://carb/IC/ data_raw/IC/
aws s3 sync s3://carb/data/workflows/phase_3/proj_mgmt_20260909/ data_raw/proj_mgmt_20260909/
aws s3 sync s3://carb/data/workflows/phase_3/demo_data_20260909/ demo_data_20260909/

cp data_raw/demo_data_20260909/inventory-config.yaml ${INVY_DIR}/inventory-config.yaml
```

## Demo dir layout

let's take a sec to understand the layout we just created. Our working directory is `${DEMO_DIR}/workflows/`. We are inside the cloned repo of `magic-ensemble` run scripts and have added four paths of interest inside it:

```sh
pwd
ls -l
git status
```

* `data_raw/` contains all the demo data pulled from s3.
	Data will be symlinked from there into other directories without duplicating the large files.
* `demo_data_20260909/` contains the site info, PFT files, initial conditions for these sites, and examples of user configs for the runs to be done today.
* `inventory/` contains just a config file but will soon contain ensemble
	inventory simulations like those demoed last week.
* `projections/` is empty now but will contain ensemble
	projections by the time we're finished.


## Run the inventory demo

```sh
cd workflows
./magic-ensemble prepare-example-3 \
	--verbose --config ${INVY_DIR}/inventory-config.yaml
./magic-ensemble run-ensembles \
	--verbose --config ${INVY_DIR}/inventory-config.yaml 
./tools/plot_ts.R \
	--model_dir ${INVY_DIR}/output \
	--plot_dir ${INVY_DIR} \
	--varname AGB
./tools/plot_ts.R \
	--model_dir ${INVY_DIR}/output \
	--plot_dir ${INVY_DIR} \
	--varname TotSoilCarb
./tools/plot_ts.R \
	--model_dir ${INVY_DIR}/output \
	--plot_dir ${INVY_DIR} \
	--varname N2O_flux
```

## Set up projection inputs

```sh
cp ${INVY_DIR}/inventory-config.yaml ${PROJN_DIR}/projection_config.yaml
vim ${PROJN_DIR}/projection_config.yaml
```

edit:

* run_dir `run-inventory` -> `run-projections`
* start_date `2026-01-01` -> `2024-01-01`
* end_date `2023-12-31` -> `2051-12-31`
* Leave run_LAI_date in 2016 for now
* All management paths `management` -> `proj_mgmt_20260909`
	(Vim tip: `:.,+7s|management|proj_mgmt_20260909|`)
* delete external ncc_info_path, set paths:ncc_info_path to "/fake/path"
	(The draft projections has one combined NCC and fert file;
	passing a fake path lets the cleaning script skip its ncc argument cleanly)


### Drivers

I've already downloaded WRF met for the entire state grid from CalAdapt by using 
```sh
srun --mem=0 --time=4320 ./workflows/tools/caladapt_download_grid.R \
	--parcel_geom_file data_raw/management/crops/v4.1.2/parcels-consolidated.gpkg \
	--output_dir wrf_45km \
	--start_year 2024 \
	--end_year 2051 \
	--models "CESM2,CNRM-ESM2-1,EC-Earth3,EC-Earth3-Veg,FGOALS-g3,MIROC6,MPI-ESM1-2-HR,TaiESM1" \
	--scenario "ssp370" \
	--resolution "d01"
```
Note: uses the `caladaptaer` R packages, which is not yet added to MAGiC env 1.17.
It will be added to the next release.

Download took on the order of 36 hours, but could likely be sped up considerably
by fetching years/models in parallel.
Note: 13 GB compressed, 17 GB expanded

Now we can convert the relevant sites to Sipnet clim files as part of the projection prepare step:

```sh
./magic-ensemble prepare-projections \
	--verbose --config ${PROJN_DIR}/projection-config.yaml
```

This has also grabbed and cleaned the projected management events, so now we're ready to set up run directories.

```sh
./magic-ensemble run-projections --verbose --config ${PROJN_DIR}/projection-config.yaml
```

This will:

1. Go fetch the sampling design and end-of-run model state from all ensemble members
	of the paired inventory run
2. Set up the projection runs using the sample ensemble parameters and initializing each model from
	the inventory run's end-of-run state
3. Set up segmented runs as appropriate
4. run the model.










================ scratch area






* As always, start a new config file `projections-config.yaml`
* Set outdir
* start_date = 2024-01-01
* end_date = 2051-12-31
* LAI date not yet changed
* 







Steps we skipped earlier:
* LAI date (expect can go once IC turned off)


still need to create:
carb/data/workflows/phase_3/proj_mgmt_20260909/
carb/data/workflows/phase_3/demo_data_20260909/ dir
	. pfts
	inventory-config.yaml
	projection-config.yaml
	. site_info.csv with grid cells
		 
git workflows carb-demo-20260909 branch

# Setting up demo data from my machine 
```sh
mkdir demo_data_20260909
# PFTs are treated as pure artifact for now => copy around
cp -R demo_data_20260903/pfts demo_data_20260909/pfts
./tools/make_crop2pft.R --output_file demo_data_20260909/crop2pft.csv

./tools/build_site_info.R --out_file demo_data_20260909/site_info.csv --location_file demo_data_20260903/site_info_6sites.csv
# ^ That created duplicate site.pft.x and site.pft.y cols.
# Edited by hand to remove, it's only 6 rows

# Gonna just copy all 6 sites' IC means
cp -R demo-run-20260909/data/IC_prep demo_data_20260909/IC_prep

```
