# GIS Dataset Dealing Note #1: CO₂ Raster Downscaling

> **Case #1:** Downscaling coarse road-transport CO₂ emission data to 1 km grids using road-network information.

---

## 1. Why I Did This

This is one of the GIS data-processing problems I encountered during my Master's research.

The original road-transport CO₂ dataset I had was a raster with a spatial resolution of roughly **11.1 km**. My other urban variables, however, were organized using **1 km × 1 km grids**.

So I had a fairly practical problem:

> **How can I redistribute the CO₂ value of a large raster cell into smaller grids without simply assuming that emissions are evenly distributed everywhere?**

My first thought was to divide the value equally among all 1 km grids inside each raster cell. That is easy, but it does not make much sense for road-transport emissions. A grid containing several major roads should probably receive a larger share than a grid containing very little road infrastructure.

I therefore used **road-network information as a spatial proxy** for redistributing the original emissions.

The basic idea is:

**Coarse CO₂ raster → identify 1 km grids → calculate road-based weights → redistribute CO₂**

This is not meant to reconstruct the "true" 1 km emission inventory. It is a **spatial downscaling assumption** based on the idea that road-transport emissions are related to the amount and type of road infrastructure within each grid.

---

## 2. Workflow

The workflow mainly uses **GeoPandas**, **Rasterio**, and **Shapely**.

### Step 0 — Set the Inputs

I first define the study area, raster file, 1 km fishnet, road-network layers, and the weights assigned to different road classes.

For the road weights, I referred to the vehicle-emission information reported by **Zheng et al. (2014)** and **calculated average representative weights** for the road categories used in my dataset.

| Road Type | Weight |
| :--- | ---: |
| Highway | `0.1933` |
| National Road | `0.2353` |
| Provincial Road | `0.1193` |
| County Road | `0.4520` |

One thing worth mentioning here is that these weights are **assumptions used for spatial allocation**, rather than directly observed traffic volumes for each road segment.

If detailed traffic-count or vehicle-flow data were available, I would prefer to use those instead.

---

### Step 1 — Prepare the 1 km Fishnet

The fine-resolution grid is loaded with GeoPandas:

```python
fine_gdf = gpd.read_file(boundary_shp)
```

I also clean the geometries and make sure that all layers use compatible coordinate reference systems.

This sounds trivial, but CRS consistency became quite important later because I use both **road length** and **grid area** in the calculation. Calculating those directly in a geographic CRS would give meaningless distance and area values.

---

### Step 2 — Convert the Coarse Raster to Spatial Units

The original CO₂ raster is first clipped to the study area using Rasterio:

```python
out_image, out_transform = mask(...)
```

I then convert the valid raster cells into polygons and store them as a GeoDataFrame:

```python
coarse_gdf = gpd.GeoDataFrame(...)
```

I did this because most of the later operations — matching grids, roads, and emission units — were easier for me to manage in one vector-based GeoPandas workflow.

So at this point I have two spatial layers:

- **11.1 km CO₂ raster cells**
- **1 km analysis grids**

The next question is simply which small grid belongs to which coarse CO₂ cell.

---

### Step 3 — Match Fine Grids to Coarse Raster Cells

For this mapping I use the **centroid of each 1 km grid**:

```python
fine_centroids.geometry = fine_centroids.geometry.centroid

mapping = gpd.sjoin(...)
```

I initially considered using polygon intersections. In practice, that creates another question: a 1 km grid located along a raster boundary can overlap two coarse cells, which means I would then need another rule for splitting that grid.

Using the centroid gives each fine grid **one parent raster cell**.

It is a simplification, but for this workflow it keeps the spatial correspondence much cleaner.

---

### Step 4 — Measure the Road Network Inside Each Grid

Next, I load the different road layers:

```python
road = gpd.read_file(...)
```

and match road segments to the 1 km grids using spatial joins:

```python
road_match = gpd.sjoin(...)
```

Road length is then calculated from the geometry:

```python
road_match.geometry.length
```

The important part here is that I do not treat every road equally.

A kilometer of one road class does not necessarily represent the same transport activity as a kilometer of another road class, so the road lengths are combined with the road-class weights defined earlier.

This gives each 1 km grid a simple measure of its relative road infrastructure intensity.

---

### Step 5 — Build the Downscaling Weights

For each fine grid `i`, I calculate a structural weight:

```text
W_i = road_weight × road_length × grid_area
```

where:

- `road_weight` is the road-class weight;
- `road_length` is the total road length within the fine grid;
- `grid_area` is the area of the fine grid.

The weights are then normalized **within each original coarse raster cell**:

```python
mapping["W_sum"] = (
    mapping.groupby("coarse_id")["W_i"]
    .transform("sum")
)
```

Conceptually, this is the part that actually performs the downscaling.

If one coarse raster cell contains several 1 km grids, its original CO₂ value is distributed among those grids according to their relative road-based weights.

In simplified form:

```text
CO2_i = CO2_coarse × (W_i / sum(W_j))
```

where `W_j` represents the weights of all fine grids belonging to the same parent coarse raster cell.

An important constraint I wanted to preserve is:

```text
sum(CO2_i) = CO2_coarse
```

within each parent coarse raster cell.

In other words, **the workflow redistributes the original emission value spatially; it should not create or remove emissions within the parent raster cell.**

---

### Step 6 — Export the 1 km Result

Finally, I attach the downscaled CO₂ estimates back to the 1 km spatial units and export the result:

```python
final_gdf.to_file("downscaled_transport_emissions.shp")
```

The resulting layer can then be joined with the other 1 km urban variables used in my later spatial modeling.
---

## 3. A Few Things I Learned From This Case

Looking back, the Python syntax was actually the easier part of this task.

The more difficult part was deciding **what spatial assumptions I was willing to make**.

### Why Road Density?

Road density is useful because it provides spatial information at a much finer resolution than the original emission raster.

But it is still only a proxy.

Two grids with similar road lengths may have completely different traffic volumes, congestion levels, vehicle compositions, and therefore emissions.

If traffic-count or vehicle-flow data were available, the downscaling weights could be improved considerably.

So I would not interpret the resulting 1 km values as independently observed emission estimates.

They are better understood as:

> **A road-informed spatial redistribution of the original coarse emission inventory.**

### Why Centroid Matching?

Centroid matching was partly a practical decision.

A full polygon intersection is geometrically more precise, but it introduces fractional overlaps between the 1 km grids and coarse raster cells.

For my analysis, I preferred having each 1 km unit associated with one parent emission cell.

For another application — especially where the two spatial resolutions are closer — I might use area-weighted intersection instead.

### Spatial Resolution Does Not Magically Increase Information

This was probably the most important lesson from this exercise.

Turning an 11.1 km raster into a 1 km dataset does **not** mean that I suddenly have true 1 km observations.

The additional spatial detail comes from the **road-network assumptions introduced during downscaling**, not from the original CO₂ dataset itself.

That distinction becomes especially important when the downscaled variable is later used in statistical or machine-learning models.

---

## 4. GIS Operations Used

I am keeping this section mainly as a quick reference for myself.

| Function / Property | Where I Used It |
| :--- | :--- |
| `gpd.read_file()` | Load fishnet and road layers |
| `gpd.GeoDataFrame()` | Store raster cells as vector features |
| `.to_crs()` | Keep distance and area calculations consistent |
| `.centroid` | Assign each fine grid to a coarse cell |
| `gpd.sjoin()` | Match grids, raster polygons, and roads |
| `.area` | Calculate grid area |
| `.length` | Calculate road length |
| `.to_file()` | Export the downscaled dataset |

### Main Libraries

- **GeoPandas** — vector spatial operations
- **Rasterio** — raster clipping and extraction
- **Shapely** — geometry handling
- **Pandas / NumPy** — tabular calculations

---

## 5. About These Notes

I started keeping these notes because I found that GIS tutorials often explain **how a function works**, but not necessarily **why you would choose one spatial operation over another in an actual research workflow**.

So this repository is less of a formal tutorial and more of a record of problems I encountered during my Master's research: what I tried, what assumptions I made, and what I would probably do differently with better data.

Some of the code was developed through my own debugging and experimentation, together with discussions with **ChatGPT** and **Gemini**. I have tried to document the methodological decisions separately from the code itself, because those decisions are usually the part I want to remember later.

The workflow is still being refined, so suggestions and alternative approaches are welcome.

---

## Reference

[1] Zheng, B., Huo, H., Zhang, Q., Yao, Z. L., Wang, X. T., Yang, X. F., Liu, H., & He, K. B. (2014). High-resolution mapping of vehicle emissions in China in 2008. *Atmospheric Chemistry and Physics, 14*, 9787–9805. https://doi.org/10.5194/acp-14-9787-2014
