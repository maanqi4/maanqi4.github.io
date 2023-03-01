---
author: "Anqi Ma"
title: "[Linux] Generate Short/Long PPE with CESM"
date: "2022-10-13"
weight: 2
tag: ["Linux","CESM"]
---
## 1. Main Settings
**Experiment name:** `f.e12.FAMIPCN.f19_f19`  
- `f` f compset;  
	“F” compsets use CAM,CLM, CICE(prescribed-thermo), DOCN(prescribed-SST).  
- `e12` cesm1.2.2 (model version)  
- `FAMIPCN` AMIP run for CMIP5 protocol with CLM/CN:  
	`AMIP_CAM4_CLM40%CN_CICE%PRES_DOCN%DOM_RTM_SGLC_SWAV`  
	`AMIP`: time, AMIP runs  
	`CAM4`: atmosphere model CAM4  
	`CLM40`: land model, clm4.0  
	`CN` (carbon-nitrogen) model version: a biogeochemistry model that simulates the carbon and nitrogen cycles  
	`CICE%PRES`: prescribed sea ice  
	`DOCN%DOM`: DOCN data ocean mode  
	`RTM`: river transport model, land river runoff  
	`SGLC`: Land Ice, Stub glacier (land ice) component  
	`SWAV`: Wave, Stub wave component  
- `f19_f19` 1.9x2.5_1.9x2.5  
	atm_grid,lnd_grid,ice_grid,ocn_grid: 1.9x2.5  
	atm_grid_type,ocn_grid_type: finite volume
	![](/images/FV_grid.png)
### 1.1 output frequency

### 1.2 enable -COSP option
In `user_nl_cam`
```shell
&cospsimulator_nl
docosp         = .true.
cosp_amwg      = .true.
cosp_cfmip_da = .true. #daily simulator 
```

## 2. Long Simulations

## 3. Short Simulations
### 3.1 Initial Conditions
#### 3.1.1 generate initial conditions
**Base:** `f.e12.FAMIPCN.f19_f19.short.Parent`
Modify `env_run.xml`
```python
./xmlchange RUN_STARTDATE = '1978-10-01'
./xmlchange STOP_OPTIONS = 'nmonths' 
./xmlchange STOP_N = '3'
./xmlchange RESUBMIT = '3'
```
Modify `user_nl_cam`
```python
inithist = 'MONTHLY'
inithist_all = .true.
```
Also remember to set relevant perturbed parameters in `user_nl_cam`
```python
cldfrc_rhminl = 0.91
cldopt_rliqocean = 14
hkconv_cmftau = 1800
cldfrc_rhminh = 0.80
zmconv_tau = 3600
```
After 4 runs consecutively, in `$ARCHIVE/rest`, there will be 4 directories, containing initial files for `1979-01-01`, `1979-04-01`, `1979-07-01`, `1979-10-01`.
Example file list in `1979-01-01`:
```shell
f.e12.FAMIPCN.f19_f19.short.Parent.cam.h0.1978-12.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cam.h1.1978-10-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cam.h2.1978-12-30-10800.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cam.i.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cam.r.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cam.rh0.1979-10-31-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cam.rs.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cice.r.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.clm2.h0.1978-12.nc
f.e12.FAMIPCN.f19_f19.short.Parent.clm2.r.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.clm2.rh0.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.cpl.r.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.docn.rs1.1979-01-01-00000.bin
f.e12.FAMIPCN.f19_f19.short.Parent.rtm.h0.1978-12.nc
f.e12.FAMIPCN.f19_f19.short.Parent.rtm.r.1979-01-01-00000.nc
f.e12.FAMIPCN.f19_f19.short.Parent.rtm.rh0.1979-01-01-00000.nc
rpointer.atm
rpointer.drv
rpointer.ice
rpointer.lnd
rpointer.ocn
rpointer.rof
```
**scripts:** `csh.init.diff.niagara` generate initial conditions for each parameter setting (modify and submit 50cases: `f.e12.FAMIPCN.f19_f19.init.diff.*`)

### 3.2 start initial runs (same initial condition for each parameter setting)
**scripts:**
1. bash ==init.short.parent.niagara== to generate parent run files [change `runstartdate`]
2. sbatch ==csh.init.short.child.niagara== to run 50 ensembles in given start date [change `runstartdate`] `f.e12.FAMIPCN.f19_f19.short.op.$i.$runstartdate`
3. munual run 4 parent runs in `f.e12.FAMIPCN.f19_f19.init.op.$runstartdate` genarated by ==init.short.parent.niagara== [munually modify parameters in `user_nl_cam`]

Modify initial files, in `user_nl_cam` (use the initial files generate by $3.1$)
```shell
ncdata = 'f.e12.FAMIPCN.f19_f19.short.Parent.cam.i.1979-01-01-00000.nc' #initial files for CAM
```
By default, this file should soft link to `$RUN`, setting `RUN_REFCASE` and `GET_REFCASE=TRUE` to make true of this.
One can also indicate specific path for initial file, but if initial conditions files are not in `$RUN` directory, make sure to also set other model's `.r.` files to avoid error. I prefer to ponit all initial files to `$RUN` for an easier operation.
Modify `env_run.xml`
```shell
./xmlchange RUN_TYPE=hybrid #default is hybrid for FAMIPCN, but make sure set it up anyway
./xmlchange RUN_STARTDATE=1979-01-01
./xmlchange RUN_REFCASE=f.e12.FAMIPCN.f19_f19.short.Parent #which REFCASE to look for in /inputdata/ccsm_init
./xmlchange RUN_REFDATE=1979-01-01 #which date/directory in REFCASE to use

./xmlchange GET_REFCASE=TRUE #soft link initial files to $RUN directory
```
One can have a case start from `1979-01-01` but using initial conditions in `1979-04-01` by setting `RUN_STARTDATE=1979-01-01` and `RUN_REFDATE=1979-04-01`. (If these is a `REFCASE` started in `1979-04-01` existed.)
Build a new case or rebuild the original case

### 3.3 initial runs with different initial conditions for different parameter setting
**scripts:** `csh.init.diff.short.child.parent`
