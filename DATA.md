# GRASS GIS data for Louisiana
NAD 1983 UTM 15N

## Statewide data

Import USGS data at 1/3 arc second / 30m resolution
```
g.extension extension=r.in.usgs operation=add
g.region vector=la_counties
r.in.usgs product=nlcd output_name=nlcd_2011_landcover output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs resampling_method=nearest
r.in.usgs product=ned output_name=ned_30m output_directory=usgs resampling_method=bilinear
```

Crop rasters to region
```
g.region vector=la_boundary res=30
v.to.rast in=la_boundary out=la_region use=val value=1
r.mapcalc expression="elevation_30m = if(la_region, ned_30m)"
r.mapcalc expression="landcover_2011 = if(la_region, nlcd_2011_landcover)"
```

## Amite watershed data
Set region
```
g.region vector=amite_basin save=amite
```

Import USGS NED data
```
g.region vector=amite_basin
r.in.usgs product=ned output_name=amite_ned_30m output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs resampling_method=bilinear
r.in.usgs product=ned output_name=amite_ned_10m output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs ned_dataset=ned13sec resampling_method=bilinear
r.in.usgs product=ned output_name=amite_ned_340cm output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs ned_dataset=ned19sec resampling_method=bilinear
r.in.usgs product=ned output_name=amite_ned_340cm output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs ned_dataset=ned19sec resampling_method=bilinear
```

Import NAIP Imagery
```
g.region vector=amite_basin
r.in.usgs product=naip output_name=amite_naip_1m output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs
```

Import NLCD
```
r.in.usgs product=nlcd output_name=nlcd_landcover_2011 output_directory=/home/baharmon/Downloads/Amite/gisdata/usgs resampling_method=nearest
```

Crop rasters to region at 30m resolution
```
g.region vector=amite_basin res=30
v.to.rast in=amite_basin out=amite_region_30m use=val value=1
r.mapcalc expression="amite_elevation_30m = if(amite_region_30m, amite_ned_30m)"
r.mapcalc expression="amite_landcover_30m = if(amite_region_30m, nlcd_landcover_2011)"
r.colors map=amite_landcover_30m raster=landcover_2011
g.remove -f type=raster name=amite_ned_30m,nlcd_landcover_2011
```

Crop rasters to region at 10 meter resolution
```
g.region vector=amite_basin res=10
v.to.rast in=amite_basin out=amite_region_10m use=val value=1
r.mapcalc expression="amite_elevation_10m = if(amite_region_10m, amite_ned_10m)"
g.remove -f type=raster name=amite_ned_10m
```

Shaded relief
```
g.region vector=amite_basin res=10
r.mask raster=amite_region_10m
r.relief input=amite_elevation_10m output=amite_relief_10m zscale=15
r.shade shade=amite_relief_10m color=amite_elevation_10m output=amite_shaded_relief_10m
r.mask -r
```

## Mining region
Set region
```
g.region vector=mining_region save=mining
```

Crop NED to the region
```
g.region vector=mining_region res=3
r.resamp.interp input=amite_ned_340cm output=amite_ned_3m method=bilinear
v.to.rast in=mining_region out=mining_region_3m use=val value=1
r.mapcalc expression="mining_elevation_3m = if(mining_region_3m, amite_ned_3m)"
g.remove -f type=raster name=amite_ned_3m,amite_ned_340cm

```

Crop imagery to the region
```
g.region vector=mining_region res=1
v.to.rast in=mining_region out=mining_region_1m use=val value=1
r.mask raster=mining_region_1m
r.mapcalc expression="mining_naip_1m.1 = if(mining_region_1m, amite_naip_1m.1)"
r.mapcalc expression="mining_naip_1m.2 = if(mining_region_1m, amite_naip_1m.2)"
r.mapcalc expression="mining_naip_1m.3 = if(mining_region_1m, amite_naip_1m.3)"
r.mapcalc expression="mining_naip_1m.4 = if(mining_region_1m, amite_naip_1m.4)"
r.composite red=amite_naip_1m.1 green=amite_naip_1m.2 blue=amite_naip_1m.3 output=mining_naip_1m
r.mask -r
```

Super-sample mining region landcover to 10m and crop
```
g.region vector=mining_region res=10
v.to.rast in=mining_region out=mining_region_10m use=val value=1
r.resamp.interp input=amite_landcover_30m output=amite_landcover_10m method=nearest
r.mapcalc expression="mining_landcover_10m = if(mining_region_10m, amite_landcover_10m)"
g.remove -f type=raster name=amite_landcover_10m
r.colors map=mining_landcover_10m raster=landcover_2011
```

Sub-sample Amite basin imagery to 10m and crop
```
g.region vector=amite_basin res=10
r.mask raster=amite_region_10m
r.mapcalc expression="amite_naip_10m = if(amite_region_10m, amite_naip_1m)"
r.colors map=amite_naip_10m raster=amite_naip_1m
g.remove -f type=raster name=amite_naip_1m,amite_naip_1m.1,amite_naip_1m.2,amite_naip_1m.3,amite_naip_1m.4
r.mask -r
```

Shaded relief
```
g.region vector=mining_region res=3
r.mask raster=mining_region_3m
r.relief input=mining_elevation_3m output=mining_relief_3m zscale=10
r.shade shade=mining_relief_3m color=mining_elevation_3m output=mining_shaded_relief_3m
r.shade shade=mining_relief_3m color=mining_naip_1m output=mining_shaded_imagery_3m brighten=50
r.neighbors input=mining_elevation_3m output=mining_smoothed_3m size=15 -c
r.contour input=mining_smoothed_3m output=mining_contours_3m step=3
g.remove -f type=raster name=mining_smoothed_3m
r.mask -r
```

## Mine region
Set region for mining site
```
g.region n=3408000 s=3405250 e=708150 w=705400 save=mine
```
