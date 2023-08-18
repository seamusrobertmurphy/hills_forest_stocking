---
title: "Hills Forest ReMapping"
author: "Summit-GIS"
date: "17/08/2023"
output: 
  pdf_document:
    toc: TRUE
    toc_depth: 5
    number_sections: FALSE
    df_print: tibble
    latex_engine: xelatex
  zotero: TRUE
---

```{r setup, echo=FALSE, message=FALSE,warning=FALSE, error=FALSE}
library(sf)
library(sp)
library(terra)
library(raster)
library(dplyr)
library(caret)
library(caretEnsemble)
library(ForestTools)
library(lidR)
library(randomForest)
library(e1071)
library(rgdal)
library(rgeos)
library(Rcpp)
library(rmarkdown)
library(knitr)
library(MASS)
library(car)
#devtools::install_github(("gearslaboratory/gdalUtils"))
library(gdalUtils)
#library(gdalUtilities)
#webshot::install_phantomjs(force = TRUE)
#knit_hooks$set(webgl = hook_webgl)
#knit_hooks$set(rgl.static = hook_rgl)
knitr::opts_chunk$set(echo = TRUE, warning=FALSE, error=FALSE, message = FALSE)
set.seed(123)
```

## Action:

The following documents tasks undertaken to ReMap Hills Forest Area and identify changes from baseline datasets. Using LiDAR DEM and DSM, a canopy height model was derived from which a stem count was calculated according to Qi's algorithm (2020).

## 1. Input: Merge & Reproject to LiDAR crs (or WKT)

```{r, fig.show='hold', out.width="50%", eval=FALSE, echo=TRUE}
# Merge chunks
filez_dem = list.files("~/Desktop/Summit_Forestry/dem", full.names = T, all.files = FALSE, pattern = '.tif$') 
filez_dsm = list.files("~/Desktop/Summit_Forestry/dsm", full.names = T, all.files = FALSE, pattern = '.tif$') 
dem_raster_list <- lapply(filez_dem, raster)
dsm_raster_list <- lapply(filez_dsm, raster)
dem_raster = do.call(merge, c(dem_raster_list, tolerance = 1))
dsm_raster = do.call(merge, c(dsm_raster_list, tolerance = 1))
writeRaster(dem_raster, filename = "~/Desktop/Summit_Forestry/dem/dem_raster.tif", overwrite=TRUE)
writeRaster(dsm_raster, filename = "~/Desktop/Summit_Forestry/dsm/dsm_raster.tif", overwrite=TRUE)
dsm_rast = terra::rast(dsm_raster)
dem_rast = terra::rast(dem_raster)
elev_rast = terra::rast(dem_raster)
chm_rast = (dsm_rast - dem_rast)

# Sometimes possible to reproject to DEM/DSM custom projection
## terra::crs(elev_rast) =  crs(chm_rast)

# Cant handle it, reprojecting everything to WKT EPSG2193 
terra::crs(elev_rast) =  "epsg:2193"
terra::crs(chm_rast) =  "epsg:2193"
terra::plot(elev_rast, main='DEM(m) - LINZ') 
terra::plot(chm_rast, main='CHM(m)') 
```

## Input: Derive AOi & Mask

```{r, fig.show='hold', out.width="50%", eval=FALSE, echo=TRUE}
mask_sf = sf::read_sf("~/Desktop/Summit_Forestry/stands/HILL-0341-009.shp")
mask_rast = rasterize(vect(mask_sf), chm_rast, touches = TRUE)
mask_rast = terra::resample(mask_rast, chm_rast, method="near") # choose from one of these two
mask_raster = raster::raster(mask_rast)
writeRaster(mask_raster, filename = "~/Desktop/Summit_Forestry/stands/mask_raster_009.tif", overwrite=TRUE)
terra::crs(mask_rast) =  "epsg:2193"
# elev_rast = terra::resample(elev_rast, chm_rast, method="near")
# chm_rast = terra::resample(chm_rast, mask_rast, method="near")
ggplot(mask_sf) + geom_sf(aes(fill = 'red'), show.legend = FALSE)

# Apply Mask
elev = mask(elev_rast, mask_rast, inverse=TRUE)
chm = mask(chm_rast, mask_rast, inverse=TRUE)
elev_raster = raster::raster(elev)
chm_raster = raster::raster(chm)
writeRaster(elev_raster, filename = "~/Desktop/Summit_Forestry/dem/elev_raster.tif", overwrite=TRUE)
writeRaster(chm_raster, filename = "~/Desktop/Summit_Forestry/dsm/chm_raster.tif", overwrite=TRUE)

# Stack rasters & visualize
## covs_m1 = raster::stack(
##  chm_raster,
##  elev_raster, 
##  mask_raster)
## rasterVis::levelplot(covs_m1)
```

## 2. Input: Derive Stem Map using ITD algorithm and 95% Canopy Height

```{r, fig.show='hold', out.width="50%", eval=FALSE}
# jerry-rig with Plowrigths variable window function (North America Conifers)
wf_plowright<-function(x){ 
  a=0.05
  b=0.6 
  y<-a*x+b 
  return(y)}
heights <- seq(0,40,0.5)
window_plowright <- wf_plowright(heights)
plot(heights, window_plowright, type = "l", ylim = c(0,12), xlab="point elevation (m)", ylab="window diameter (m)", main='Plowright, 2018; y=0.05*x+0.6')
```

## Input: Derive Height & Stems from TreeTops using Plowright taper & 95% function

```{r, eval=TRUE, fig.show='hold', out.width="33%", echo=FALSE}
kernel <- matrix(1,3,3)
chm_raster = focal(chm_rast, w = kernel, fun = median, na.rm = TRUE) %>% raster()
ttops_1.5mfloor_plowright = ForestTools::vwf(chm_raster, wf_plowright, 1.5)
writeOGR(ttops_1.5floor_plowright, "~/Desktop/Summit_Forestry/stands", "treetops_hills_009", driver = "ESRI Shapefile") # memory dump here

quant95 <- function(x, ...) 
  quantile(x, c(0.95), na.rm = TRUE)
custFuns <- list(quant95, max)
names(custFuns) <- c("95thQuantile", "Max")

ttops_1.5floor_raster <- ForestTools::sp_summarise(ttops_1.5floor_plowright, grid = 1, variables = "height", statFuns = custFuns)
chm_95height_raster = ttops_1.5floor_raster[["height95thQuantile"]]
chm_95height_rast = terra::rast(chm_95height_raster)
stem_count_raster = ttops_1.5floor_raster[["TreeCount"]]
stem_count_sf = st_as_sf(stem_count_raster)
stem_count_rast = rasterize(vect(stem_count_sf), raster_template, fun = sum, touches = TRUE)
stem_count_ha_rast = stem_count_rast*100 #aggregate to 100m resolution for stems per hectare
mypalette<-brewer.pal(8,"Greens")
plot(chm_95height_raster, col = mypalette, alpha=0.6)
plot(st_geometry(stem_count_sf["treeID"]), add=TRUE, cex = 0.001, pch=19, col = 'red', alpha=0.8) 

raster::writeRaster(chm_95height_raster, filename = "~/Desktop/Summit_Forestry/stands/chm_95height.tif", overwrite=TRUE)
raster::writeRaster(stem_count_raster, filename = "~/Desktop/Summit_Forestry/stands/stem_count.tif", overwrite=TRUE)
chm_95height_rast = terra::rast(chm_95height_raster)
stem_count_rast = terra::rast(stem_count_raster)
terra::plot(chm_95height_rast)
terra::plot(stem_count_rast)


# Point Cloud Pipeline (or smaller areas with lighter memory tasks...)
## stem_count_sp = find_trees(chm_95height, lmf(wf_plowright), uniqueness = "bitmerge")
## stem_count_sf = st_as_sf(stem_count_sp)
## mypalette<-brewer.pal(8,"Greens")
## plot(chm_95height_raster, col = mypalette, alpha=0.6)
## plot(st_geometry(stem_count_sf["treeID"]), add=TRUE, cex = 0.001, pch=19, col = 'red', alpha=0.8)

# Point Cloud Pipeline: Check Tree-ID as proof of stem count...
## algo = dalponte2016(chm_95height_raster, stem_count_sp)
## chm_95h_segmented = segment_trees(chm_95height_raster, algo)
## chm_95h_segmented_crowns = delineate_crowns(chm_95h_segmented, type = 'convex')
## chm_95h_segmented_crowns_spplot = sp::plot(chm_95h_segmented_crowns, cex=0.000001, axes = TRUE, alpha=0.1)

# Point Cloud Pipeline: Derive raster
## chm_95h_segmented_crowns_sf = st_as_sf(chm_95h_segmented_crowns, coords = c("XTOP", "YTOP"), st_crs=2193)
## psych::describe(chm_95h_segmented_crowns_sf$treeID)
## raster_template = rast(ext(chm_95height_raster), resolution = 1, crs = st_crs(chm_95height_raster)$wkt)
## segmented_crowns_rast = rasterize(vect(chm_95h_segmented_crowns_sf), raster_template, fun = sum, touches = TRUE)
## stem_count_ha_rast = segmented_crowns_rast*100 #convert from 1m resolution 100m-sq fpr stems per hectare
## stem_count_ha_raster = raster::raster(stem_count_ha_rast, filename = "~/Desktop/Summit_Forestry/stands/stem_count_ha_raster.tif")
## stem_count_ha_raster = writeRaster(stem_count_ha_raster, filename = "~/Desktop/Summit_Forestry/stands/stem_count_ha_raster.tif", overwrite=TRUE)
## stem_count_ha_raster = raster::raster(stem_count_ha_rast, filename = "~/Desktop/Summit_Forestry/stands/stem_count_ha_raster.tif")
## plot(segmented_crowns_rast)
## plot(stem_count_ha_raster)


```

# Outputs: Compare Stem Maps in Polygon, Heatmap and Histograms

```{r, fig.show='hold', out.width="50%", echo=FALSE}
stem_count_sf = st_as_sf(stem_count)
plot(st_geometry(las_tile_ahbau_ttops_sf["treeID"]), add=TRUE, cex = 0.001, pch=19, col = 'red', alpha=0.8)
plot(stem_count_raster, main="Quesnel Developed Blocks WSVHA", cex.main=0.8, maxpixels=22000000)
hist(stem_count_raster, main="Quesnel Developed Blocks WSVHA", cex.main=0.8, maxpixels=22000000) 
```

