# Geo-Based Campsite Search & Type Exclusion

**Date:** 2026-06-06
**Status:** Approved

## Overview

Two related features added to the `camply campsites` command:

1. **Geo-based search** — find campsites within a radius (miles) of a location, fanning out across all providers that have coordinate data, without the user needing to specify rec-area or campground IDs manually.
2. **Campsite type exclusion** — filter out unwanted campsite types (e.g. group sites, horse-only, boat-access-only) from results.

---

## Feature 1: Geo-Based Search

### New CLI flags

```
--near TEXT         Place name, geocoded to lat/lon via Nominatim (mutually exclusive with --latitude/--longitude)
--latitude FLOAT    Center latitude in decimal degrees
--longitude FLOAT   Center longitude in decimal degrees
--radius FLOAT      Search radius in miles (required when any geo param is present)
```

**Examples:**
```bash
camply campsites --near "San Francisco, CA" --radius 100 --start-date 2026-07-10 --end-date 2026-07-14
camply campsites --latitude 37.77 --longitude -122.41 --radius 100 --start-date 2026-07-10 --end-date 2026-07-14
```

When geo params are present and `--provider` is not specified, the search fans out across all providers that have coordinate data (see Provider Support below). Users may pass `--provider` to restrict geo search to a single provider (e.g. `--provider ReserveCalifornia --near "Los Angeles" --radius 50`). Passing `--provider GoingToCamp` with geo params is an error.

### New YAML config fields

```yaml
near: "San Francisco, CA"     # mutually exclusive with latitude/longitude
latitude: 37.77
longitude: -122.41
radius: 100                   # miles
```

### Provider Support

| Provider | Coordinate Source | Discovery Method |
|---|---|---|
| RecreationDotGov | RIDB API native | Pass `latitude`, `longitude`, `radius` to `/facilities` endpoint |
| ReserveCalifornia | UseDirect cached metadata | Haversine filter on `UseDirectFacility.Latitude/Longitude` |
| AlabamaStateParks | UseDirect cached metadata | Haversine filter |
| ArizonaStateParks | UseDirect cached metadata | Haversine filter |
| FloridaStateParks | UseDirect cached metadata | Haversine filter |
| MinnesotaStateParks | UseDirect cached metadata | Haversine filter |
| MissouriStateParks | UseDirect cached metadata | Haversine filter |
| OhioStateParks | UseDirect cached metadata | Haversine filter |
| OregonMetro | UseDirect cached metadata | Haversine filter |
| VirginiaStateParks | UseDirect cached metadata | Haversine filter |
| FairfaxCountyParks | UseDirect cached metadata | Haversine filter |
| MaricopaCountyParks | UseDirect cached metadata | Haversine filter |
| NorthernTerritory | UseDirect cached metadata | Haversine filter |
| Yellowstone | Hardcoded center (~44.5°N, 110.5°W) | Include if park center is within radius |
| **GoingToCamp** | **None** | **Not supported — error if explicitly combined with geo params** |

### New Components

#### `camply/utils/geo_utils.py`

Two functions:

```python
def geocode_location(place_name: str) -> Tuple[float, float]:
    """Convert a place name to (latitude, longitude) using Nominatim (OpenStreetMap).
    Raises CamplyError if the place cannot be geocoded."""

def haversine_distance_miles(lat1: float, lon1: float, lat2: float, lon2: float) -> float:
    """Return the great-circle distance in miles between two coordinates."""
```

New dependency: `geopy` (for Nominatim geocoder).

#### `camply/search/search_geo.py` — `SearchGeo`

New class inheriting `BaseCampingSearch`. Its `get_all_campsites()` method:

1. Resolves center coordinates (geocode `--near` if needed)
2. For each applicable provider, runs geo-discovery to get campground IDs within radius:
   - RecreationDotGov: calls `_find_facilities_from_search(search=None, latitude=..., longitude=..., radius=...)` — these kwargs pass through to the RIDB `/facilities` API which accepts them natively
   - UseDirect variants: populates `CampgroundFacility.coordinates` from cached metadata and haversine-filters
   - Yellowstone: checks hardcoded park center against radius
3. Instantiates each provider's search class with the discovered campground IDs
4. Calls each provider's `get_all_campsites()` and returns the aggregated list

This design preserves the existing `BaseCampingSearch.get_matching_campsites()` polling/notification infrastructure unchanged.

#### Changes to `SearchUseDirect` / UseDirect provider

- Populate `CampgroundFacility.coordinates` from `UseDirectFacility.Latitude` / `UseDirectFacility.Longitude` when building the campground list. This field exists on the model but is currently never set.

### Error Handling

| Condition | Behaviour |
|---|---|
| `--near` and `--latitude`/`--longitude` both provided | Exit with error: mutually exclusive |
| Any geo param provided without `--radius` | Exit with error: `--radius` is required |
| `--near` geocoding fails (place not found) | Exit with error: "Could not geocode location: \<place\>" |
| `--provider GoingToCamp` with geo params | Exit with error: "GoingToCamp does not support geo-based search" |
| No campgrounds found within radius | Exit with message: "No campgrounds found within \<N\> miles of \<location\>" rather than polling forever |
| Provider returns 0 campgrounds (too far away) | Silently skip that provider — no error, no polling |

---

## Feature 2: Campsite Type Exclusion

### New CLI flag

```
--exclude-type TEXT    Exclude campsite types matching this string (repeatable)
```

**Examples:**
```bash
camply campsites --rec-area 2991 --start-date 2026-07-10 --end-date 2026-07-14 \
  --exclude-type group --exclude-type horse --exclude-type "boat access"
```

Can be combined with geo-based search:
```bash
camply campsites --near "San Francisco, CA" --radius 100 \
  --start-date 2026-07-10 --end-date 2026-07-14 \
  --exclude-type group --exclude-type horse
```

### New YAML config field

```yaml
excluded_campsite_types:
  - group
  - horse
  - boat access
```

### Matching behaviour

Case-insensitive **substring** match against `AvailableCampsite.campsite_type`. A value of `"group"` excludes `"Group Standard"`, `"Group Walk-To"`, `"GROUP SITE"`, etc. Users can be more specific (`"group standard"`) to narrow the exclusion. Empty list (default) excludes nothing.

### Implementation

Add `excluded_campsite_types: List[str]` parameter to `BaseCampingSearch.__init__()`. In `get_matching_campsites()`, after fetching available campsites and before sending notifications, apply:

```python
if self.excluded_campsite_types:
    campsites = [
        c for c in campsites
        if not any(
            excl.lower() in (c.campsite_type or "").lower()
            for excl in self.excluded_campsite_types
        )
    ]
```

Because this lives in `BaseCampingSearch`, it applies automatically to all providers including `SearchGeo`.

---

## Dependencies

| Package | Purpose | Notes |
|---|---|---|
| `geopy` | Nominatim geocoding | Free, no API key required |

---

## Testing

- Unit tests for `haversine_distance_miles` with known coordinate pairs
- Unit tests for type exclusion filter with mixed-case type strings
- Integration test: geo-discovery against RecreationDotGov returns campgrounds for a known lat/lon/radius
- Unit test: `SearchGeo` skips providers that return 0 campgrounds without error
- Unit test: `--near` + `--latitude` mutual exclusion raises error
- Unit test: missing `--radius` raises error
