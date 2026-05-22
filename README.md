# GIS Dataset Dealing Note #1: CO₂ Raster Downscaling

> **Case #1:** Downscaling coarse-resolution road transport carbon emission raster data into high-resolution grids using road network density information.

---

##  1. Project Introduction

In urban and transportation research, **ArcPy** and *GeoPandas* are among the most commonly used GIS data processing libraries in Python.
However, many tutorials online **only explain syntax superficially**, while few demonstrate how these libraries are applied in *real-world urban and transportation research workflows*. 

This repository is part of my **master's research notes**. I will share several **GIS data processing cases** developed during my graduate research, explaining how these libraries can be utilized in practical spatial analysis tasks.

### 🤝 Acknowledgments & Discussion
The code in this repository was completed through:
* **Self-modification and debugging** 
* **Deep discussions with ChatGPT** 
* **Assistance from Gemini** 

Suggestions, feedback, and academic discussions are always welcome! ❤

---

## 2. What Does This Code Do?

In this case, I attempted to downscale coarse-resolution road transport carbon emission raster data (*Coarse-resolution Raster*, approximately **11.1 km** grid size) into fine-resolution grids (*Fine-resolution Fishnet*, **1 km** fishnet grids) using **weighted road network density information** as spatial weights.

---

### Step 0 — Parameter Settings
In this step, global parameters are defined, including the **city name**, **raster path**, **fishnet boundary**, **road network layers**, and **road weights**.

Different road types are assigned specific weights based on their estimated contribution to traffic emissions:
| Road Type | Weight |
| :--- | :--- |
| **Highway** | `0.1933` |
| **National Road** | `0.2353` |
| **Provincial Road** | `0.1193` |
| **County Road** | `0.4520` |

---

###  Step 1 — Load Fishnet and Convert Geometry to 2D
The fine-resolution fishnet shapefile is loaded via *GeoPandas*:
```python
fine_gdf = gpd.read_file(boundary_shp)
```

###  Step 2 — Raster Processing
The raster dataset is processed using Rasterio. Main operations include:
clipping raster by study area,
extracting raster cells,
converting raster cells into polygons.
out_image, out_transform = mask(...)

Raster polygons are then converted into a GeoDataFrame:
```python
coarse_gdf = gpd.GeoDataFrame(...)
```
This step transforms raster-based carbon emission data into vector polygons for subsequent spatial operations.

###  Step 3 — Spatial Mapping Between Fine and Coarse Grids
To determine which fine grids belong to which coarse raster polygons:
```python
mapping = gpd.sjoin(...)
```
Instead of directly intersecting polygons, centroid-based matching is used:
```python
fine_centroids.geometry = fine_centroids.geometry.centroid
```
This approach avoids many topology and fragmentation errors.

###  Step 4 — Road Density Calculation
Road shapefiles are loaded one by one:
```python
road = gpd.read_file(...)
```
Spatial joins are then performed:
```python
road_match = gpd.sjoin(...)
```
For each fine grid:
intersecting road segments are identified,
road lengths are summed,
weighted road density is calculated.

Road length is computed using:
```python
road_match.geometry.length
```

### Step 5 — Downscaling Calculation

The structural weight $W_i$ for each fine grid unit is calculated based on the localized road infrastructure attributes:
$$W_i = \text{road\_weight} \times \text{road\_length} \times \text{grid\_area}$$
Then, group-by normalization is applied to aggregate the weights within each corresponding coarse raster cell boundary:
```python
# Sum up total weights within each coarse grid mapping group
mapping["W_sum"] = mapping.groupby("coarse_id")["W_i"].transform("sum")
```

### Step 6 — Export Results
The final high-resolution downscaled transport emission *GeoDataFrame* is serialized and exported locally using the standard GeoPandas output API:
```python
# Export the result to a specified spatial data format
final_gdf.to_file("downscaled_transport_emissions.shp")
```

## 4. GeoPandas Syntax Used in This Case

This project leverages several core functionalities of the **GeoPandas** library to handle vector data operations. Below is a quick reference guide to the syntax used in this workflow:

|  Function / Property |  Operational Usage |
| :--- | :--- |
| `gpd.read_file()` | Reads vector datasets (e.g., ESRI Shapefiles, GeoPackages) |
| `gpd.GeoDataFrame()` | Instantiates a spatially enabled structural DataFrame |
| `.to_crs()` | Transforms coordinate tracking into alternative Coordinate Reference Systems |
| `.geometry` | Accesses the active geometric tracking column within the dataset |
| `.centroid` | Extracts representative geometric point centroids from polygons |
| `gpd.sjoin()` | Executes spatial overlay joins based on topological predicates (e.g., `within`, `intersects`) |
| `.area` | Computes exact planar area metrics of individual polygon elements |
| `.length` | Computes linear map distances of line string geometries |
| `.to_file()` | Serializes and exports spatial data frames back into local storage files |

## 5. This workflow combines multiple Python GIS libraries:
Library	Purpose
GeoPandas	Vector spatial analysis
Rasterio	Raster processing
Shapely	Geometry operations
Pandas	Data manipulation
NumPy	Numerical operations
Notes

## 6. This repository focuses on:
practical GIS workflows,
urban spatial analysis,
transportation-related data processing,
and research-oriented Python GIS usage.

The code is still under continuous refinement and optimization.
