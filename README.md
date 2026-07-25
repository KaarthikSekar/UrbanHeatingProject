# UrbanCool Chennai 🌡️
 
Mapping urban heat islands (UHI) in Chennai using satellite data and machine learning, to identify neighborhoods that most need trees and cooling infrastructure.
 
<img width="1497" height="946" alt="Screenshot 2026-07-25 at 6 10 21 PM" src="https://github.com/user-attachments/assets/e6ca0eda-b498-4bfa-b855-cf692b520aca" />

 
## What this project does
 
Chennai, like most cities, doesn't heat up evenly — some neighborhoods run several degrees hotter than others due to less vegetation, more concrete, and dense construction. This project builds an end-to-end pipeline that:
 
1. Pulls satellite data for the city (temperature, vegetation, land cover, elevation, weather)
2. Grids the city into 250m cells and computes average values per cell
3. Trains a model to predict surface temperature from land features
4. Maps predicted heat intensity across the city
5. Flags the hottest, least-green neighborhoods as priority zones for intervention
## Pipeline
 
| Step | Script | What it does |
|------|--------|---------------|
| 0 | [`01_download_gee_data.py`](01_download_gee_data.py) | Downloads 5 satellite layers from Google Earth Engine (LST, NDVI, land cover, elevation, ERA5 weather) as GeoTIFFs |
| 1 | [`02_build_grid_features.py`](02_build_grid_features.py) | Overlays a 250m grid on the city and averages each raster layer per cell |
| 2 | [`03_train_model.py`](03_train_model.py) | Trains a Random Forest and a neural network to predict temperature from land features; evaluates with a spatial train/test split |
| 3 | [`04_map_predictions.py`](04_map_predictions.py) | Predicts temperature for every cell, draws the heat map, and ranks cells by a "hot + low vegetation" priority score |
| 4 | [`05_identify_neighborhoods.py`](05_identify_neighborhoods.py) | Reverse-geocodes the top priority cells to real neighborhood names via OpenStreetMap |
 
Run them in order (`01` → `05`).
 
## Data sources
 
| Layer | Source | Resolution |
|-------|--------|------------|
| Land Surface Temperature | MODIS (`MODIS/061/MOD11A2`) | 1 km |
| NDVI (vegetation) | Sentinel-2 SR | 10 m |
| Land cover | ESA WorldCover v200 | 10 m |
| Elevation | SRTM | 30 m |
| Weather (air temp, precipitation) | ERA5-Land Monthly Aggregated | ~9 km |
 
Time period: June–August 2024 (summer). Study area: Chennai city boundary ([`city_boundary.geojson`](city_boundary.geojson)).
 
## Setup
 
```bash
pip install earthengine-api geemap geopandas rasterio rasterstats shapely pandas scikit-learn tensorflow matplotlib geopy
```
 
You'll need a free [Google Earth Engine](https://earthengine.google.com/) account and project to run `01_download_gee_data.py`. Replace the project ID on the `ee.Initialize(...)` line with your own.
 
## Repo structure
 
```
├── 01_download_gee_data.py
├── 02_build_grid_features.py
├── 03_train_model.py
├── 04_map_predictions.py
├── 05_identify_neighborhoods.py
├── city_boundary.geojson          # study area boundary
├── rasters/                       # output of step 0 (lst.tif, ndvi.tif, landcover.tif, elevation.tif, era5.tif)
├── grid_features.csv              # output of step 1: one row per grid cell
├── grid_features_geometry.gpkg    # cell shapes, for mapping
├── priority_cells.csv             # output of step 3: top priority cells
├── priority_cells_named.csv       # output of step 4: priority cells with neighborhood names
└── uhi_priority_map.png           # final heat map
```
 
## Results
 
The model was trained on ~7,700 grid cells across Chennai. Predicted heat intensity ranges from well below to ~2.5°C above the city's land average, concentrated in the western/central urban core and dropping off toward the coast.
 
Top priority neighborhoods identified (hottest + least vegetated):
 
| Neighborhood | Heat above average (°C) | NDVI |
|---|---|---|
| Zone 7, Ambattur | +2.5 | 0.12 |
| Alapakkam | +2.3 | 0.10 |
| Zone 11, Valasaravakkam | +2.5 | 0.12 |
 
See [`priority_cells_named.csv`](priority_cells_named.csv) for the full ranked list
## Notes & limitations
 
- NDVI comes from median Sentinel-2 imagery filtered to <60% cloud cover — monsoon season coverage is imperfect, so some cells may be noisier than others.
- The spatial checkerboard train/test split avoids nearby-cell leakage but the model is still only as good as the correlation between land features and temperature — it doesn't capture things like building height, traffic, or waste heat.
- Neighborhood names come from OpenStreetMap's free Nominatim geocoder and may not always match official ward boundaries.
