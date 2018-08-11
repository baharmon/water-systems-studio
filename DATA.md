# GRASS GIS data for Louisiana
NAD 1983 UTM 15N

## Statewide data

Import USGS data at 1/3 arc second / 30m resolution
```
g.extension extension=r.in.usgs operation=add
g.region vector=la_counties
r.in.usgs product=nlcd output_name=nlcd_2011_landcover output_directory=/usgs resampling_method=nearest
r.in.usgs product=ned output_name=ned_30m output_directory=usgs resampling_method=bilinear
```

Crop rasters to region
```
g.region vector=la_boundary res=30
v.to.rast in=la_boundary out=la_region use=val value=1
r.mapcalc expression="elevation_30m = if(la_region, ned_30m)"
r.mapcalc expression="landcover_2011 = if(la_region, nlcd_2011_landcover)"
```

## Baton Rouge data
Set mapset
```
g.mapset mapset=baton_rouge
```

Import vector data from city
```
v.import input=Parish_Boundary.shp layer=Parish_Boundary output=parish
v.import input=Building_Footprint.shp layer=Building_Footprint output=buildings
v.import input=Downtown_Development_District.shp layer=Downtown_Development_District output=downtown
```


Crop rasters to region at 30m resolution
```
g.region vector=parish res=30
v.to.rast in=parish out=parish_30m use=val value=1
r.mapcalc expression="landcover_30m = if(parish_30m, landcover_2011)"
r.colors map=landcover_30m raster=landcover_2011
```

Import USGS NED data
```
r.in.usgs product=ned output_name=ned_10m output_directory=/usgs ned_dataset=ned13sec resampling_method=bilinear
```

Crop rasters to region at 10m resolution
```
g.region vector=parish res=10
v.to.rast in=parish out=parish_10m use=val value=1
r.mapcalc expression="ned_elevation_10m = if(parish_10m, ned_10m)"
r.colors -e map=ned_elevation_10m color=viridis
g.remove -f type=raster name=ned_10m
```

Import ATlAS lidar to a new location

```
v.proj location=louisiana mapset=baton_rouge input=parish
g.region vector=parish res=10
r.in.xyz --overwrite input=raw_3009139ne.csv output=raw_3009139ne separator=comma vscale=0.3048
```
To automate run `import_atlas.py`

Reproject ATLAS lidar
```
g.mapset mapset=baton_rouge
g.region vector=parish res=10
r.proj location=atlas mapset=PERMANENT input=atlas_surface_10m method=bilinear memory=3000
r.mapcalc expression="surface_10m = if(parish_10m, atlas_surface_10m)"
r.colors -e map=surface_10m color=viridis
g.rename raster=surface_10,atlas_surface_10m --overwrite
```

Crop downtown maps
```
g.region vector=downtown res=10
v.to.rast in=downtown out=downtown_10m use=val value=1
r.mapcalc expression="downtown_surface_10m = if(downtown_10m, atlas_surface_10m)"
r.colors -e map=downtown_surface_10m color=viridis
```

## Amite watershed data
Set region
```
g.region vector=amite_basin save=amite
```

Import USGS NED data
```
g.region vector=amite_basin
r.in.usgs product=ned output_name=usgs resampling_method=bilinear
r.in.usgs product=ned output_name=amite_ned_10m output_directory=/usgs ned_dataset=ned13sec resampling_method=bilinear
r.in.usgs product=ned output_name=amite_ned_340cm output_directory=/usgs ned_dataset=ned19sec resampling_method=bilinear
```

Import NAIP Imagery
```
g.region vector=amite_basin
r.in.usgs product=naip output_name=amite_naip_1m output_directory=/usgs
```

Import NLCD
```
r.in.usgs product=nlcd output_name=nlcd_landcover_2011 output_directory=/usgs resampling_method=nearest
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
r.shade shade=amite_relief_10m color=amite_landcover_30m output=amite_shaded_landcover brighten=36
r.mask -r
```

## Mining region
Set region
```
g.region vector=mining_region save=mining
```

Create vector area from region
```
v.in.region output=mining_region
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
g.region n=3408000 s=3405250 e=708150 w=705400 save=mine res=1
```

Create vector area from region
```
v.in.region output=mine_region
```

Supersample and crop elevation to the region
```
v.to.rast in=mine_region out=mine_region_1m use=val value=1
r.mask raster=mining_region_1m
r.resamp.interp input=mining_elevation_3me output=mine_elevation_1m method=bicubic
r.colors -e map=mine_elevation_1m color=elevation
r.mask -r
```

Compute topographic parameters and shaded relief
```
r.mask raster=mining_region_1m
r.contour input=mine_elevation_1m output=mine_contours_1m step=1
r.neighbors -c input=mine_elevation_1 output=mine_smoothed_1m size=11
r.contour input=mine_smoothed_1m output=mine_contours_1m step=1 --overwrite
g.remove -f type=raster name=mine_smoothed_1m
r.geomorphon elevation=mine_elevation_1m forms=mine_landforms_1m search=36 skip=0 flat=1 dist=0
r.relief input=mine_elevation_1m output=mine_relief_1m zscale=3
r.skyview input=mine_elevation_1m output=mine_skyview_1m ndir=8 colorized_output=mine_colorized_skyview_1m
r.shade shade=mine_relief_1m color=mine_colorized_skyview_1moutput=mine_shaded_relief brighten=40
r.shade shade=mine_skyview_1m color=mining_naip_1m output=mine_shaded_imagery brighten=20
r.shade shade=mine_relief_1m color=mine_landforms_1m output=mine_shaded_landforms brighten=36
r.mask -r
```
