# background

## project overview

Currently, NLDAS-3 wind is only bilinearly interpolated MERRA-2 that
has been terrain-corrected using the MicroMet procedure from
(Liston & Elder, 2006) with MERIT curvature data.

Ultimately, outside of terrain correction, NLDAS-3 is effectively
just interpolated MERRA-2 data.

The new product needs to be hourly but most important for
validation is daily min/max/mean

**current correction inputs**

`/discover/nobackup/projects/eis_nldas3/dmocko/nldas3/winds/`
`lis_input.nldas3.noahmp401.1km.land_and_inland_water.curvature.nc`
`lis_input.nldas3.noahmp401.1km.land_and_inland_water.m2interp.nc`

## proposed strategy

1. Acquire ground station data from a variety of regional contexts
   (ie mountainous, plains, coastal, tropical, sub-arctic)
2. For each station, develop error covariance wrt interpolated
   background fields using multiple interpolation schemes
   (ie nearest, bilinear, linearized, MicroMet, WindMapper) in a
   variety of flow directions.
3. For stations that appear to capture meaningful

## possible background fields

| model  | res   | domain    | freq     | start | notes       |
| ------ | ----- | --------- | -------- | ----- | ----------- |
| NAM    | 12km  | N America | 4x daily | 2006- | retire soon |
| HRDPS  | 2.5km | US/Canada | 4x daily | 2015- |             |
| RRFS   | 3km   | N America | hourly   | 2026- | in dev      |
| NARR   | 32km  | N America | 8x daily | 1979- |             |
| MERRA2 | ~50km | Global    | hourly   | 1980- |             |
| CaPA   | 10km  | N America | 4x daily | 2002- | pcp only    |

## merra-2

variables grouped into files by use case, with repetition.

Instantaneous collections are wrt synoptic times.

Time-averaged are timestamped with the central time of their period.

For all info see:
https://gmao.gsfc.nasa.gov/media/publications/zbly36ziNFDFbmYmvhQeVqPhUo/Bosilovich785.pdf

### grid domain

The model is run on a cubed-sphere domain natively. see vis:
https://geoschem.github.io/cube-sphere-step.html

This is interpolated onto a equirectangular grid with
5/8 degree longitude and 1/2 degree latitude resolution
(~50 x 55.5 km at 40 latitude)

Final character in the file name refers to:

- x: horizontal data only
- p: vertical pressure coordinates
- v: model layer centers
- e: model layer edges

pressure coordinates are always on 42 levels, while the model-level
data is natively at 72 layers with 15mb thickness.

### assimilated meteorological fields

- **inst1_2d_asm_Nx**: (lon:576,lat:361,t:24)
  - 2,10,50m wind; skin,2,10m temp; 2,10m humidity, pressure
- **inst3_3d_asm_Np**: (lon:576,lat:361,lvl:42,t:8)
  - (all layers): pot vort, humidity, temp, wind
- **inst3_3d_asm_Nv**: (lon:576,lat:361,lvl:72,t:8)
  - same as Np but more layers plus cloud fraction

### analyzed meteorological fields

- **inst6_3d_ana_Np**: (lon:576,lat:361,lvl:42,t:4)
  - height, humidity, temperature, wind
- **inst6_3d_ana_Nv**: (lon:576,lat:361,lvl:72,t:4)
  - pressure, humidity, temperature, wind

### other

- **tavg1_2d_flx_Nx** (lon:576,lat:361,t:24)
  - buoyancy scale, exchange coeff, drag coeff, fluxes, ground heat,
    PBL height, partitioned precip, humidity, bulk richardson number,
    surface wind speed, sfc wind gravity wave stress, general wind
    stress, skin temp, air temp, surface roughness.

- **tavg1_2d_slv_Nx** (lon:576,lat:361,t:24)
  - 850,500,250mb height,humidity,temp,wind; 500mb omega;
    2,10,50m wind; LCL, skin temp, 2,10m temp.

- **tavg3_3d_udt_Np** (lon:576,lat:361,lvl:42,t:8)
  - wind tendency, and components due to moisture, turbulence,
    dynamics, and gravity wave drag.

## rtma

- 2.5km resolution

- Background field is a blend of HRRR & NAM, weighted according to
  the forecast distance.

- Observations are +/- 30 min from analysis time, or closest to
  analysis time if there are multiple observations in the period.

- Background grid is interpolated to obs points, and differences
  are computed.

- Background error covariance is used to calculate "increments" of
  contribution from obs to surrounding grid points.

- After version 2.8, all station obs are adjusted to 10m
  - Background wind is adjusted using similarity theory

- 2d variational analysis is conducted using *Gridpoint Statistical
  Interpolation* (GSI), seeking the minimum of a cost function.

- When obs vs background is very large, other conditional corrective
  processes are invoked.

### RTMA-reported wind obs limitations

- Due to poor ground station siting, ~60% of mesonet wind
  observations pass quality check.
  - Sometimes wind acceptance is conditional on wind direction due
    to the location of a barrier or other surrounding feature.

## WindNinja (Forthofer et al., 2014)

- mass-conserving and mass+momentum-conserving models

- assumes neutral stability

- intended for ~100m grid sizeo

- captures ridge top acceleration and valley channeling

## Windmapper (Marsh  et al., 2023)

### classical approaches from intro

- fluid dynamics or LES based approaches are computationally
  expensive and numerically unstable for downscaling NWP outputs
  to ~200m, especially in complex terrain

- Some diagnostic models preserve mass conservation but neglect
  momentum conservation, which decreases sensitivity to surface
  roughness / boundary layer condition, and are faster, but lee
  slope winds suffer due to failure to capture recirculation and
  flow separation.

- Linearized models are applicable for low-slope terrain but don't
  generalize well to complex terrain

- MicroMet is widely-adopted, but requires calibration, and
  pertubations wrt background field are limited to 22.5 degrees.

- Terrain sheltering indexes can be effective, but only provide
  speed, not wind direction.

- Some approaches pre-compute terrain impacts on quasi-stationary
  time-averaged wind flow using high-resolution models, then use a
  lookup table

- **windmapper** extends the LUT approach of (Essery, 1999) and
  (Barcons, 2018) by using mass-conserving windflow model
  WindNinja (Wagenbrenner, 2016)

### model development

target is downscaling ~2.5km NWP models to microscale (~50m)

1. Choose N directions for windfield library
2. Run WindNinja to produce windfields for each direction

## micromet (Liston & Elder,  2005)

Model to produce 30m-1km gridded forcings using nearby ground station
data, and data-fills using procedures including ARIMA.

- Single missing timestep is assumed to be average of before/after
  - Many missing timesteps uses ARIMA.

- Barnes gaussian-weighted interpolation used for spatial imputation.

- Wind adjusted with (Liston & Sturm, 1998) according to topographic
  slope and curvature relationships.
  - Not strictly divergence-free

## chatgpt

### overall methods

- **Mass-consistent diagnostic models** interpolate to a fine grid
  and use high-res terrain to solve a constrained optimization
  problem. Basically redistributes flow according to terrain & da.
  - includes CALMET, WindNinja
  - enforces continuity and straightforward data assimilation
  - doesn't simulate mountain waves, lee circulation, fronts, eddys.

- **Variational objective analysis** used by WRF-DA, GSI. Treats the
  coarse field as a background, builds covariance structures based
  on the terrain, assimilates station winds, constrains continuity.
  - Includes 3DVAR / 4DVAR

- **Machine learning** including CNNs, U-Nets, Graph Neural Networks
  can reproduce terrain acceleration, but don't conserve mass or
  momentum, and may hallucinate flow features.
  - predictors: background, elevation, slope, curvature, exposure,
    land cover, upstream blocking indices.

- **Hybrid statistical-physical downscaling** may use a statistical
  diagnostic method to start, then predict residuals with ML.

### assimilation approaches

- **Optimal interpolation** determines an analysis value with minimal
  error variance by assigning weights based on error covariance
  matrices for a background field and observation points.

- **Ensemble Kalman Filter**

- **4DVAR** Useful for temporal consistency (?)

### high-resolution phenomena (fronts, terrain)

- Identifying front location and movement will require dense station
  data assimilation or other higher-resolution fields

- Mountain waves and mesoscale winds in many cases depend on
  atmospheric stability and vertical wind shear information.

### general recommendations

1. interpolate to 1km
2. assimilate station winds with 3DVAR or EnKF
3. apply mass adjustment to enforce continuity
4. train residual model for terrain-induced perturbations.
5. "reproject to a divergence-constrained space"

divergence-constrained reprojection: some grid points may show
unrealistic acceleration that doesn't correspond to any discernable
process. You can solve poisson equation or do constrained
optimization as in WindNinja to adjust the ML outputs.

## chatgsfc (downscaling to resolve fronts)

A model that resolves fronts must incorporate a temporal dimension,
and spatial covariance capturing how point observations influence the
surrounding grid over time.

## deep learning approach

- use something like a ConvLSTM or video vision transformer.

- provide the features at high resolution, with the ground stations
  as the only non-zero pixels, alongside a binary mask, the
  interpolated background field, and covariates like slope/elevation.

- loss function for mass conservation (divergence-free flow)

- **WindNinja** and **CALMET** are meant to assimilate ground
  station data in downscaling a background field
  - *but* fronts are likely to have bullseyes unless something like
    spatiotemporal kriging with external drifts is used.
  - such approaches should capture covariance in space and time,
    and should better capture frontal movement between stations.

- Diffusion models may better capture sharp frontal gradients
  - NVIDIA CorrDiff (25km -> 2km)
  - Diffusion for Spatiotemporal Kriging/Inpainting
  - Condition the diffusion process with high-res interpolated
    background field, covariates, and masked station data
  - Ground station data would act as an *inpainting mask*. After
    each iteration, the station data pixels would be re-replaced
    with thte source values
  - 3D U-Net would be needed as the backbone of the model in order
    to force temporal coherence.
  - high-resolution target data would still be needed as truth basis.


## concepts

### processes affecting wind

- Terrain interactions have multiple overlapping scales
  - Wind accelerates over high ridges, sheltered on lee side
- mechanical turbul

- From ChatGSFC: sfc roughness, vegetation/lulc drag, channeling and
  orographic effects from terrain, mixing/decoupling from stability,
  katabatic effects (density-driven), differential heating
  (UHI/sea breeze), synoptic forcing, boundary layer depth.

### similarity theory

1. determine variables relevant to the process
2. organize variables into dimensionless groups, introducing
   exponents where necessary
3. fit a regression to observational data

If there are missing variables, correlation between groups will not
be discernable, or groups will have high variance.

Buckingham Pi theory can be used to identify groups

Similarity relationships usually apply to processes at equilibrium,
so highly time-coupled processes like boundary layer depth make it
difficult to develop a similarity representation.

Once defined, similarity relationships can be used to diagnose the
equilibrium states of unknown values.

## literature

### windmapper

**(Marsh et al., 2023)**

https://doi.org/10.1029/2022WR032683

Downscales NWP then terrain-corrects interpolation outputs.

### math model for diagnosis of surface wind in mountains

**(Ryan, 1977)**

https://doi.org/10.1175/1520-0450(1977)016%3C0571:AMMFDA%3E2.0.CO;2

Names processes:

- synoptic flow, monsoon flow, sea breeze, slope wind, sheltering.
- generally applicable functions for valley wind, slope wind,
  sheltering, and diverting.
