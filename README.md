# GTFS-Based Transit Itinerary Reconstruction

This repository contains the implementation of the **GTFS-based route matching algorithm** described in:

> **"Beyond Mode Detection: Reconstructing Detailed Transit Itineraries from Crowdsourced GPS Trajectories"**
> Ojas Jagtap, Naman Awasthi, Mohit Jain, Saad Mohammad Abrar, Vanessa Frías-Martínez
> *SpatialConnect '25, November 3-6, 2025, Minneapolis, MN, USA*
> DOI: [10.1145/3764924.3770890](https://doi.org/10.1145/3764924.3770890)

## Overview

Traditional mobility analysis focuses on broad mode detection (walk/bus/rail), but fails to identify **specific transit lines** and **stop-level behaviors** critical for transit planning and equity analysis. This algorithm reconstructs detailed transit itineraries from raw GPS trajectories, identifying:

- **Specific transit routes** taken (e.g., "Bus Route 10" not just "bus")
- **Boarding and alighting stops** for each transit leg
- **Walking segments** connecting transit rides
- **Transfer patterns** and wait times

### Key Features

✅ **R-tree spatial indexing** for efficient route matching
✅ **LCSS-based trajectory similarity** robust to GPS noise
✅ **~80% accuracy** on real-world crowdsourced GPS data
✅ **Stop-level resolution** for boarding/alighting identification
✅ **Automatic segment merging** to reconstruct continuous rides
✅ **Multi-modal support** (bus, rail, metro, walking)

## Algorithm Pipeline

```
Raw GPS Trajectory
       ↓
[1] Preprocessing & Cleaning
   - Remove GPS drift/outliers
   - Densify to uniform intervals
   - Kalman filter smoothing
       ↓
[2] Staypoint Detection
   - Sliding window algorithm
   - Identify stops/transfers
       ↓
[3] Trip Segmentation
   - Split at staypoints
   - Create trip legs
       ↓
[4] R-tree Spatial Query
   - Retrieve candidate routes
   - Filter by bounding box
       ↓
[5] LCSS Route Matching
   - Calculate trajectory similarity
   - Select best-fit route
       ↓
[6] Stop Identification
   - Match segment endpoints to GTFS stops
   - Identify boarding/alighting locations
       ↓
[7] Segment Merging
   - Merge consecutive legs on same route
   - Reconstruct continuous rides
       ↓
Detailed Transit Itinerary
(routes + stops + walking segments)
```

## Installation

### Prerequisites

- Python 3.8+
- Jupyter Notebook
- GTFS data files for your transit system

### Dependencies

```bash
pip install numpy pandas geopandas shapely folium matplotlib rtree tqdm trackintel transbigdata
```

You'll also need the `gtfs_functions` library for GTFS data handling. Install from:
```bash
pip install gtfs-functions
```

## Usage

### Quick Start

```python
# 1. Load the notebook
jupyter notebook rtree.ipynb

# 2. Build R-tree index from GTFS data
route_index, gtfs_routes = gen_rtree()

# 3. Load your GPS trajectory data
trip = pd.read_csv("trajectory.csv")  # columns: timestamp, tripIdPk, lat, lon

# 4. Process trajectory through the pipeline
pfs = generate_pfs(trip)                              # Preprocess & segment
matched_routes = match_all_legs(pfs, gtfs_routes, route_index)  # Match routes
seg_data = gen_seg_data(matched_routes)               # Identify stops
merged = merge_segments(seg_data)                     # Merge segments

# 5. View results
itinerary_df = gen_seg_df(merged)                     # Summary table
map_viz = plot(merged)                                # Interactive map
```

### Input Data Format

Your GPS trajectory CSV should have these columns:

| Column | Type | Description |
|--------|------|-------------|
| `tripIdPk` | string | Unique trip identifier |
| `timestamp` | datetime | GPS point timestamp |
| `lat` | float | Latitude (WGS84) |
| `lon` | float | Longitude (WGS84) |

Example:
```csv
tripIdPk,timestamp,lat,lon
trip_001,2024-01-15 08:30:00,39.2904,-76.6122
trip_001,2024-01-15 08:30:30,39.2908,-76.6118
...
```

### GTFS Data Setup

1. Download GTFS feeds for your transit system (ZIP format)
2. Place them in a `GTFS/` directory
3. Update paths in `gen_rtree()` function:

```python
def gen_rtree():
    # Update these paths to match your GTFS files
    localbus_path = 'GTFS/your_localbus.zip'
    metro_path = 'GTFS/your_metro.zip'
    # ... add more feeds as needed
```

### Output Format

The algorithm produces a structured itinerary with this information:

```python
{
    'route_name': 'MONDAWMIN - BAYVIEW',        # Transit line name (None for walking)
    'route_id': 'route_10',                     # GTFS route identifier
    'start_stop_name': 'Main St & 1st Ave',     # Boarding stop
    'end_stop_name': 'City Hall Station',       # Alighting stop
    'seg_geom': LineString([...]),              # GPS trajectory geometry
    'route_geom': LineString([...])             # Matched GTFS route geometry
}
```

### Batch Processing

To process multiple trips:

```python
from tqdm import tqdm

results = []
for trip_id, trip_data in tqdm(trips.groupby('tripIdPk')):
    pfs = generate_pfs(trip_data)
    matched = match_all_legs(pfs, gtfs_routes, route_index)
    seg_data = gen_seg_data(matched)
    merged = merge_segments(seg_data)

    results.append({
        'trip_id': trip_id,
        'itinerary': gen_seg_df(merged)
    })
```

## Algorithm Parameters

Key parameters can be tuned in the code:

### Preprocessing (`generate_pfs`)
```python
speedlimit=120      # Max speed (km/h) for outlier detection
dislimit=1000       # Max jump distance (m) for outlier detection
timegap=10          # Densification interval (seconds)
dist_threshold=100  # Staypoint radius (m)
time_threshold=3    # Staypoint duration (minutes)
```

### Route Matching (`match_leg_to_route_lcss`)
```python
epsilon=0.0005      # LCSS distance threshold (~50m)
```

### Stop Identification (`gen_seg_data`)
```python
buffer=0.0005       # Spatial buffer for stop matching (~50m)
```

These defaults work well for urban transit with ~2 points/minute GPS sampling.

## Performance

**Evaluated on Baltimore City transit dataset (100 journeys, 232 trip legs):**

|  Metric  | Value |
|----------|-------|
| Accuracy | 80.2% |

**Comparison with state-of-the-art mode detection models:**
- Traditional ML (Stenneth et al.): Identifies "bus" but not which bus route
- Deep learning (Pei et al.): 83% mode accuracy on GeoLife dataset, but 53% on Baltimore data
- **Our approach**: 80% accuracy **with specific route identification**

## Algorithm Details

### R-tree Spatial Indexing

The R-tree data structure enables efficient spatial queries:

- **Without R-tree**: O(n) comparison against all ~1000+ routes
- **With R-tree**: O(log n) query returns only 10-50 candidate routes
- Typical speedup: **50-100x faster**

### LCSS Trajectory Matching

Longest Common Subsequence (LCSS) finds the longest matching subsequence of GPS points between trajectory and route:

**Why LCSS over DTW or Fréchet?**
- ✅ **Robust to outliers** (gaps in matching allowed)
- ✅ **Handles GPS noise** (small deviations don't break match)
- ✅ **Temporal flexibility** (no strict time alignment)
- ✅ **Better for parallel routes** (doesn't overfit to nearby shapes)

**Empirical comparison on our dataset:**
- LCSS: **80% accuracy**
- DTW: 72% (overfit to parallel routes)
- Fréchet: 68% (sensitive to local noise)

### Segment Merging

Merges consecutive segments on the same route that connect at shared stops. Handles:
- GPS signal loss creating artificial breaks
- Staypoint detection errors
- Brief stops (traffic lights, passenger loading)

## Use Cases

This algorithm enables:

### 🚌 **Transit Planning**
- Identify underserved routes and neighborhoods
- Optimize schedules based on actual usage patterns
- Detect unreliable connections requiring service improvements

### ⚖️ **Equity Analysis**
- Compare wait times across different neighborhoods
- Measure transit accessibility for vulnerable populations
- Evaluate service quality disparities

### 📊 **Performance Monitoring**
- Track actual vs. scheduled service
- Measure transfer efficiency
- Identify routes with low ridership

### 🔬 **Research Applications**
- Urban mobility pattern analysis
- Multi-modal trip chain reconstruction
- Travel behavior studies

## Limitations

- **Requires GTFS data**: Algorithm needs complete GTFS feeds with route shapes
- **GPS quality dependent**: Very poor GPS data (<1 point/minute) may fail to match
- **Static schedules**: Uses static GTFS, doesn't account for real-time detours
- **Urban focus**: Tuned for dense urban transit networks

## Citation

If you use this code in your research, please cite:

```bibtex
@inproceedings{jagtap2025beyond,
  author = {Jagtap, Ojas and Awasthi, Naman and Jain, Mohit and Abrar, Saad Mohammad and Fr\'ias-Mart\'inez, Vanessa},
  title = {Beyond Mode Detection: Reconstructing Detailed Transit Itineraries from Crowdsourced GPS Trajectories},
  booktitle = {Proceedings of the 1st ACM SIGSPATIAL International Workshop on Spatial Intelligence for Smart and Connected Communities},
  series = {SpatialConnect '25},
  year = {2025},
  location = {Minneapolis, MN, USA},
  doi = {10.1145/3764924.3770890},
  publisher = {ACM}
}
```

## Related Work

This is the **GTFS-based algorithm** from our paper. We also developed a **Google Maps API-based approach** that achieves 85% accuracy. See the paper for details.

## Contact

**Authors:**
- Ojas Jagtap - [ojagtap@umd.edu](mailto:ojagtap@umd.edu)
- Naman Awasthi - [nawasthi@umd.edu](mailto:nawasthi@umd.edu)
- Mohit Jain - [jainm27@umd.edu](mailto:jainm27@umd.edu)
- Saad Mohammad Abrar - [sabrar@umd.edu](mailto:sabrar@umd.edu)
- Vanessa Frías-Martínez - [vfrias@umd.edu](mailto:vfrias@umd.edu)

**Affiliation:** University of Maryland

## Acknowledgments

This research was conducted as part of the [BALTO project](https://balto.umd.edu/), a community-centered toolkit for equitable public transit advocacy.

---

**Keywords:** Transit Itinerary Reconstruction, GTFS Spatial Indexing, Multi-Modal Trajectory Analysis, Urban Mobility Equity
