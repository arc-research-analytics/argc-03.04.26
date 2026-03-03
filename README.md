# Transportation Impact Layer with H3

This repository accompanies a presentation on building a **transportation impact layer** using [H3](https://h3geo.org/), Uber's open-source hexagonal hierarchical spatial index. The notebook demonstrates how to spatially join transportation project data to H3 hexagons to identify "hot spots" of transportation investment across metro Atlanta.

## What is H3?

H3 is a geospatial indexing system that partitions the world into hexagonal cells at multiple resolutions. Hexagons are preferred over squares for spatial analysis because they have uniform adjacency (every neighbor shares an edge, not just a corner), reduce sampling bias, and provide a more natural representation of spatial proximity.

Each resolution level increases the number of cells and decreases their size:

| Resolution | Avg Hex Area | Use Case / Extent                             |
| :--------: | :----------: | --------------------------------------------- |
|     5      |   ~98 mi²    | Regional or metro                             |
|     6      |   ~14 mi²    | County or town                                |
|   **7**    | **~1.9 mi²** | **Neighborhood-level (used in this project)** |
|     8      |   ~0.3 mi²   | Block level                                   |
|     9      |  ~0.04 mi²   | Parcel level                                  |

## Getting Started

### Prerequisites

- Python 3.9+
- A basic understanding of GIS concepts (coordinate reference systems, shapefiles/GeoJSON)

### Installation

1. **Clone the repository:**

```bash
git clone https://github.com/arc-research-analytics/argc-03.04.26.git
cd argc-03.04.26
```

2. **Create a virtual environment (recommended):**

```bash
python -m venv venv
source venv/bin/activate        # Mac/Linux
venv\Scripts\activate           # Windows
```

3. **Install dependencies:**

```bash
pip install -r requirements.txt
```

> **Note on the `h3` library:** This project uses **h3 v4.x**, which introduced significant API changes from v3. If you've used h3 before, be aware that many function names changed (e.g., `polyfill` → `geo_to_cells`, `h3_to_geo_boundary` → `cell_to_boundary`). Make sure you're on v4+ by running `pip show h3`.

4. **Launch Jupyter:**

```bash
jupyter notebook
```

Then open `notebooks/h3_sandbox.ipynb` from the Jupyter interface.

---

## Part 1: Convert Any Geography to H3 Hexagons

The first building block is a function that takes any GeoDataFrame (counties, cities, census tracts, etc.) and returns a GeoDataFrame of H3 hexagons that tile the input area. The function handles CRS reprojection to WGS84, dissolving multi-feature inputs, and converting each H3 cell ID into a Shapely polygon.

Swap in your own geography to experiment — a single county shapefile, census tracts from [TIGER/Line](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html), a city boundary, or any polygon layer you have on hand. Try different resolution levels to see how the hexagon density changes.

---

## Part 2: Building the Transportation Impact Layer

Once you have a hexagonal grid, the real analytical power comes from **spatially joining data to it**. This section overlays GDOT transportation project lines onto the hex grid to create an "impact layer" showing where investment is concentrated.

### The Problem

Transportation projects are lines (roads, corridors, transit routes) that often span multiple hexagons. Simply counting which hexagon a project's centroid falls in would be misleading — a 20-mile highway project would only register in one cell. Instead, we need to **split each project line by the hexagons it passes through** and allocate its cost proportionally based on how much of the project falls within each hexagon.

### The Approach

1. Filter to active projects (pre-construction and under-construction).
2. Project both layers to a planar CRS (`EPSG:3857`) so that length calculations are in meters.
3. Use `gpd.overlay()` to intersect project lines with hexagons — this clips each line to the boundary of each hexagon it crosses.
4. Calculate each clipped segment's proportion of the total project length.
5. Allocate each project's cost estimate proportionally across the hexagons it intersects.
6. Aggregate by hexagon to get total project counts and allocated investment per cell.

### Key Concepts

**Why proportional allocation matters:** A $50M highway project that passes through 10 hexagons shouldn't count as $50M in each one. By calculating the proportion of the project's total length that falls within each hexagon, we allocate costs fairly. A hexagon containing 15% of the project's length receives 15% of its cost estimate.

**Why we project to EPSG:3857:** Length calculations on geographic coordinates (latitude/longitude) are unreliable because degrees don't correspond to consistent distances. Projecting to a planar coordinate system like Web Mercator ensures that our length-based proportions are accurate.

**Why `gpd.overlay` instead of `gpd.sjoin`:** A spatial join (`sjoin`) assigns entire features to hexagons based on a spatial relationship (intersects, contains, etc.), but it doesn't clip geometries. The `overlay` operation actually splits the project lines at hexagon boundaries, giving us the clipped segments we need for proportional allocation.

### Output Schema

The resulting GeoDataFrame adds these columns to the original hexagon grid:

| Column               | Description                                                               |
| -------------------- | ------------------------------------------------------------------------- |
| `project_count_pre`  | Number of pre-construction project segments intersecting this hexagon     |
| `allocated_cost_pre` | Proportionally allocated cost of pre-construction projects (in dollars)   |
| `project_count_uc`   | Number of under-construction project segments intersecting this hexagon   |
| `allocated_cost_uc`  | Proportionally allocated cost of under-construction projects (in dollars) |

---

## Useful Resources

- [H3 Documentation](https://h3geo.org/)
- [H3 Python API Reference](https://uber.github.io/h3-py/api_reference)
- [H3 Resolution Table](https://h3geo.org/docs/core-library/restable/) — detailed specs for each resolution level
- [GeoPandas `overlay` Documentation](https://geopandas.org/en/stable/docs/reference/api/geopandas.overlay.html)
- [GeoPandas Documentation](https://geopandas.org/)
- [Shapely Documentation](https://shapely.readthedocs.io/)

## Questions?

If you have questions about the notebook or want to learn more about how the Atlanta Regional Commission uses H3 and geospatial analysis, feel free to open an issue in this repo.
