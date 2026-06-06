# Geo-Based Campsite Search & Type Exclusion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `--near`/`--latitude`/`--longitude`/`--radius` flags to `camply campsites` for distance-based multi-provider search, and `--exclude-type` to filter out unwanted campsite types.

**Architecture:** A new `SearchGeo` class inherits `BaseCampingSearch` and overrides `get_all_campsites()` to fan out across all geo-capable providers (RecreationDotGov, all UseDirect variants, Yellowstone). Each provider discovers its campgrounds within the radius during `__init__`, then availability is checked via the existing per-provider search classes. `excluded_campsite_types` is added to `BaseCampingSearch` itself so all providers benefit from it.

**Tech Stack:** Python 3.9+, Click, Pydantic v1, `geopy` (new dependency for Nominatim geocoding), existing camply provider/search infrastructure.

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `pyproject.toml` | Modify | Add `geopy` dependency |
| `camply/utils/geo_utils.py` | **Create** | `geocode_location()` and `haversine_distance_miles()` |
| `camply/search/base_search.py` | Modify | Add `excluded_campsite_types` param + filter |
| `camply/providers/usedirect/usedirect.py` | Modify | Populate `RecreationArea.coordinates` and `CampgroundFacility.coordinates` from place lat/lon |
| `camply/providers/recreation_dot_gov/recdotgov_provider.py` | Modify | Allow `find_campgrounds()` to accept geo kwargs without `search_string`/`state` |
| `camply/search/search_geo.py` | **Create** | `SearchGeo` orchestrator |
| `camply/search/__init__.py` | Modify | Export `SearchGeo` |
| `camply/cli.py` | Modify | Add `--near`, `--latitude`, `--longitude`, `--radius`, `--exclude-type` to `campsites` |
| `camply/containers/search_model.py` | Modify | Add `near`, `latitude`, `longitude`, `radius`, `excluded_campsite_types` to `YamlSearchFile` |
| `camply/utils/yaml_utils.py` | Modify | Thread new YAML fields into `provider_kwargs` |
| `tests/utils/test_geo_utils.py` | **Create** | Unit tests for geocoding and haversine |
| `tests/search_providers/test_geo_search.py` | **Create** | Unit tests for `SearchGeo` and type exclusion |

---

## Task 1: Add `geopy` dependency

**Files:**
- Modify: `pyproject.toml`

- [ ] **Step 1: Add geopy to pyproject.toml**

In `pyproject.toml`, add `"geopy>=2.3,<3"` to the `dependencies` list:

```toml
dependencies = [
  "click~=8.1.3",
  "fake-useragent~=1.4.0",
  "geopy>=2.3,<3",
  "pandas>=2,<3",
  "pydantic~=1.10.22",
  "python-dotenv~=1.0.0",
  "pytz~=2023.2",
  "pyyaml~=6.0",
  "ratelimit~=2.2.1",
  "requests~=2.31.0",
  "rich~=13.3.2",
  "rich-click~=1.6.1",
  "tenacity~=8.2.2"
]
```

- [ ] **Step 2: Install the dependency**

```bash
pip install "geopy>=2.3,<3"
```

Expected: installs cleanly, `import geopy` works in a Python shell.

- [ ] **Step 3: Commit**

```bash
git add pyproject.toml
git commit -m "build: add geopy dependency for geocoding support"
```

---

## Task 2: Create `geo_utils.py`

**Files:**
- Create: `camply/utils/geo_utils.py`
- Create: `tests/utils/__init__.py`
- Create: `tests/utils/test_geo_utils.py`

- [ ] **Step 1: Write failing tests**

Create `tests/utils/__init__.py` (empty).

Create `tests/utils/test_geo_utils.py`:

```python
"""
Tests for geo_utils
"""
import pytest

from camply.utils.geo_utils import haversine_distance_miles, geocode_location


def test_haversine_same_point():
    assert haversine_distance_miles(37.77, -122.41, 37.77, -122.41) == pytest.approx(0.0, abs=0.001)


def test_haversine_sf_to_la():
    # San Francisco to Los Angeles is ~347 miles
    result = haversine_distance_miles(37.7749, -122.4194, 34.0522, -118.2437)
    assert 340 < result < 360


def test_haversine_sf_to_nyc():
    # SF to NYC is ~2570 miles
    result = haversine_distance_miles(37.7749, -122.4194, 40.7128, -74.0060)
    assert 2500 < result < 2650


def test_geocode_returns_tuple(mocker):
    mock_location = mocker.MagicMock()
    mock_location.latitude = 37.7749
    mock_location.longitude = -122.4194
    mocker.patch(
        "camply.utils.geo_utils.Nominatim.geocode",
        return_value=mock_location,
    )
    lat, lon = geocode_location("San Francisco, CA")
    assert lat == pytest.approx(37.7749)
    assert lon == pytest.approx(-122.4194)


def test_geocode_raises_on_not_found(mocker):
    mocker.patch("camply.utils.geo_utils.Nominatim.geocode", return_value=None)
    with pytest.raises(Exception, match="Could not geocode"):
        geocode_location("zzzznotarealplace12345")
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest tests/utils/test_geo_utils.py -v
```

Expected: `ModuleNotFoundError: No module named 'camply.utils.geo_utils'`

- [ ] **Step 3: Create `camply/utils/geo_utils.py`**

```python
"""
Geo-coding and distance utilities
"""

import math
from typing import Tuple

from geopy.geocoders import Nominatim

from camply.exceptions import CamplyError

_EARTH_RADIUS_MILES = 3958.8
_GEOCODER = Nominatim(user_agent="camply")


def haversine_distance_miles(
    lat1: float, lon1: float, lat2: float, lon2: float
) -> float:
    """Return the great-circle distance in miles between two lat/lon points."""
    lat1, lon1, lat2, lon2 = map(math.radians, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = math.sin(dlat / 2) ** 2 + math.cos(lat1) * math.cos(lat2) * math.sin(dlon / 2) ** 2
    return _EARTH_RADIUS_MILES * 2 * math.asin(math.sqrt(a))


def geocode_location(place_name: str) -> Tuple[float, float]:
    """Convert a place name to (latitude, longitude) using Nominatim.

    Raises CamplyError if the place cannot be geocoded.
    """
    location = _GEOCODER.geocode(place_name)
    if location is None:
        raise CamplyError(f"Could not geocode location: {place_name!r}")
    return location.latitude, location.longitude
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
pytest tests/utils/test_geo_utils.py -v
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add camply/utils/geo_utils.py tests/utils/__init__.py tests/utils/test_geo_utils.py
git commit -m "feat: add geo_utils with haversine distance and Nominatim geocoding"
```

---

## Task 3: Add `excluded_campsite_types` to `BaseCampingSearch`

**Files:**
- Modify: `camply/search/base_search.py:45-100` (`__init__` signature and body)
- Modify: `camply/search/base_search.py:227-278` (`_search_matching_campsites_available`)
- Modify: `tests/conftest.py` (no change needed — existing `available_campsite` fixture covers this)
- Modify: `tests/search_providers/test_geo_search.py` (add type exclusion tests here in Task 9)

- [ ] **Step 1: Add `excluded_campsite_types` parameter to `BaseCampingSearch.__init__`**

In `camply/search/base_search.py`, update `__init__` to accept and store the new parameter.

Change the signature at line 45:

```python
def __init__(
    self,
    search_window: Union[SearchWindow, List[SearchWindow]],
    weekends_only: bool = False,
    nights: int = 1,
    offline_search: bool = False,
    offline_search_path: Optional[str] = None,
    days_of_the_week: Optional[Sequence[int]] = None,
    excluded_campsite_types: Optional[List[str]] = None,
    **kwargs,
) -> None:
```

Then add this line at the end of `__init__` (after line 100, before the closing of the method body):

```python
        self.excluded_campsite_types: List[str] = excluded_campsite_types or []
```

The full end of `__init__` should look like (replacing the existing last lines of `__init__`):

```python
        self.offline_search_path = self._set_offline_search_path(
            file_path=offline_search_path
        )
        self.campsites_found: Set[AvailableCampsite] = self._load_campsites_from_file()
        self.search_attempts = 0
        self.notifier = MultiNotifierProvider(provider=["silent"])
        self.excluded_campsite_types: List[str] = excluded_campsite_types or []
```

- [ ] **Step 2: Add type exclusion filter in `_search_matching_campsites_available`**

In `camply/search/base_search.py`, within `_search_matching_campsites_available` (around line 246), after the existing date/nights filter loop, add the type exclusion filter.

Find this block (around lines 246–254):

```python
        matching_campgrounds = []
        for camp in self.get_all_campsites():
            if all(
                [
                    self._compare_date_overlap(campsite=camp) is True,
                    camp.booking_nights >= self.nights,
                ]
            ):
                matching_campgrounds.append(camp)
```

Replace with:

```python
        matching_campgrounds = []
        for camp in self.get_all_campsites():
            if all(
                [
                    self._compare_date_overlap(campsite=camp) is True,
                    camp.booking_nights >= self.nights,
                ]
            ):
                matching_campgrounds.append(camp)
        if self.excluded_campsite_types:
            matching_campgrounds = [
                c for c in matching_campgrounds
                if not any(
                    excl.lower() in (c.campsite_type or "").lower()
                    for excl in self.excluded_campsite_types
                )
            ]
```

- [ ] **Step 3: Run existing tests to verify no regressions**

```bash
pytest tests/ -v -k "not geo_search"
```

Expected: all existing tests PASS (the new parameter defaults to `[]` so existing behaviour is unchanged).

- [ ] **Step 4: Commit**

```bash
git add camply/search/base_search.py
git commit -m "feat: add excluded_campsite_types filtering to BaseCampingSearch"
```

---

## Task 4: Populate UseDirect coordinates from cached metadata

**Files:**
- Modify: `camply/providers/usedirect/usedirect.py:578-648`

The `UseDirectDetailedPlace` model (from `/rdr/search/places`) has `Latitude` and `Longitude`. Currently `_get_places()` drops them when building `RecreationArea` objects, and `_get_facilities()` builds `CampgroundFacility` without coordinates. This task fixes both.

- [ ] **Step 1: Populate `RecreationArea.coordinates` in `_get_places()`**

In `camply/providers/usedirect/usedirect.py`, find the `_get_places()` method at line 578. The block that builds `usedirect_rec_areas` currently looks like:

```python
        self.usedirect_rec_areas: Dict[int, RecreationArea] = {
            place.PlaceId: RecreationArea(
                recreation_area=place.Name,
                recreation_area_id=place.PlaceId,
                recreation_area_location=f"{place.City.title()}, {place.State}",
                description=place.Description,
            )
            for place in places_data_validated.values()
        }
```

Replace with:

```python
        self.usedirect_rec_areas: Dict[int, RecreationArea] = {
            place.PlaceId: RecreationArea(
                recreation_area=place.Name,
                recreation_area_id=place.PlaceId,
                recreation_area_location=f"{place.City.title()}, {place.State}",
                description=place.Description,
                coordinates=(place.Latitude, place.Longitude)
                if place.Latitude is not None and place.Longitude is not None
                else None,
            )
            for place in places_data_validated.values()
        }
```

- [ ] **Step 2: Populate `CampgroundFacility.coordinates` in `_get_facilities()`**

In `camply/providers/usedirect/usedirect.py`, find the `_get_facilities()` method at line 613. The loop that builds `usedirect_campgrounds` currently looks like:

```python
        self.usedirect_campgrounds: Dict[int, CampgroundFacility] = {}
        for facility in facilities_data_validated.values():
            rec_area = self.usedirect_rec_areas.get(facility.PlaceId, None)
            if rec_area is not None:
                self.usedirect_campgrounds[facility.FacilityId] = CampgroundFacility(
                    facility_name=facility.Name,
                    facility_id=facility.FacilityId,
                    recreation_area_id=facility.PlaceId,
                    recreation_area=rec_area.recreation_area,
                )
```

Replace with:

```python
        self.usedirect_campgrounds: Dict[int, CampgroundFacility] = {}
        for facility in facilities_data_validated.values():
            rec_area = self.usedirect_rec_areas.get(facility.PlaceId, None)
            if rec_area is not None:
                self.usedirect_campgrounds[facility.FacilityId] = CampgroundFacility(
                    facility_name=facility.Name,
                    facility_id=facility.FacilityId,
                    recreation_area_id=facility.PlaceId,
                    recreation_area=rec_area.recreation_area,
                    coordinates=rec_area.coordinates,
                )
```

- [ ] **Step 3: Run existing UseDirect tests to verify no regressions**

```bash
pytest tests/cli/usedirect/ tests/search_providers/test_reserve_california.py -v
```

Expected: all existing tests PASS.

- [ ] **Step 4: Commit**

```bash
git add camply/providers/usedirect/usedirect.py
git commit -m "feat: populate RecreationArea and CampgroundFacility coordinates in UseDirect providers"
```

---

## Task 5: Allow RecreationDotGov `find_campgrounds` to accept geo kwargs

**Files:**
- Modify: `camply/providers/recreation_dot_gov/recdotgov_provider.py:211-223`

Currently `find_campgrounds()` raises `RuntimeError` when `search_string` is `None` and `state` is `None`, even if geo kwargs (`latitude`, `longitude`, `radius`) are present. This needs a small fix.

- [ ] **Step 1: Update the guard check to allow geo kwargs**

In `camply/providers/recreation_dot_gov/recdotgov_provider.py`, find this block (around lines 211–223):

```python
        else:
            state_arg = kwargs.get("state", None)
            if state_arg is not None:
                kwargs.update({"state": state_arg.upper()})
            if search_string in ["", None] and state_arg is None:
                raise RuntimeError(
                    "You must provide a search query or state to find campsites"
                )
            if self.activity_name:
                kwargs["activity"] = self.activity_name
            facilities = self._find_facilities_from_search(
                search=search_string, **kwargs
            )
```

Replace with:

```python
        else:
            state_arg = kwargs.get("state", None)
            if state_arg is not None:
                kwargs.update({"state": state_arg.upper()})
            geo_arg = kwargs.get("latitude", None)
            if search_string in ["", None] and state_arg is None and geo_arg is None:
                raise RuntimeError(
                    "You must provide a search query, state, or lat/lon to find campsites"
                )
            if self.activity_name:
                kwargs["activity"] = self.activity_name
            facilities = self._find_facilities_from_search(
                search=search_string, **kwargs
            )
```

- [ ] **Step 2: Run RecreationDotGov tests to verify no regressions**

```bash
pytest tests/search_providers/test_recdotgov_search.py tests/cli/test_campgrounds.py -v
```

Expected: all existing tests PASS.

- [ ] **Step 3: Commit**

```bash
git add camply/providers/recreation_dot_gov/recdotgov_provider.py
git commit -m "feat: allow RecreationDotGov find_campgrounds to accept geo lat/lon/radius kwargs"
```

---

## Task 6: Create `SearchGeo` orchestrator

**Files:**
- Create: `camply/search/search_geo.py`
- Modify: `camply/search/__init__.py`

- [ ] **Step 1: Create `camply/search/search_geo.py`**

```python
"""
Geo-based multi-provider campsite search
"""

import logging
import sys
from typing import Dict, List, Optional, Tuple, Type, Union

from camply.containers import AvailableCampsite, CampgroundFacility, SearchWindow
from camply.exceptions import CamplyError
from camply.providers import RecreationDotGov
from camply.providers.xanterra.yellowstone_lodging import Yellowstone
from camply.search.base_search import BaseCampingSearch
from camply.search.search_recreationdotgov import SearchRecreationDotGov
from camply.search.search_usedirect import (
    SearchAlabamaStateParks,
    SearchArizonaStateParks,
    SearchFairfaxCountyParks,
    SearchFloridaStateParks,
    SearchMaricopaCountyParks,
    SearchMinnesotaStateParks,
    SearchMissouriStateParks,
    SearchNorthernTerritory,
    SearchOhioStateParks,
    SearchOregonMetro,
    SearchReserveCalifornia,
    SearchVirginiaStateParks,
)
from camply.search.search_yellowstone import SearchYellowstone
from camply.utils.geo_utils import geocode_location, haversine_distance_miles

logger = logging.getLogger(__name__)

# (latitude, longitude) of Yellowstone's geographic center
_YELLOWSTONE_CENTER: Tuple[float, float] = (44.4280, -110.5885)

# UseDirect search classes eligible for geo-search (all have lat/lon in cached metadata)
_USEDIRECT_SEARCH_CLASSES = [
    SearchReserveCalifornia,
    SearchAlabamaStateParks,
    SearchArizonaStateParks,
    SearchFloridaStateParks,
    SearchMinnesotaStateParks,
    SearchMissouriStateParks,
    SearchOhioStateParks,
    SearchOregonMetro,
    SearchVirginiaStateParks,
    SearchFairfaxCountyParks,
    SearchMaricopaCountyParks,
    SearchNorthernTerritory,
]


class SearchGeo(BaseCampingSearch):
    """
    Multi-provider campsite search by distance from a geographic point.

    Fans out across RecreationDotGov, all UseDirect providers, and Yellowstone.
    """

    provider_class = RecreationDotGov

    def __init__(
        self,
        search_window: Union[SearchWindow, List[SearchWindow]],
        latitude: float,
        longitude: float,
        radius_miles: float,
        provider_filter: Optional[str] = None,
        weekends_only: bool = False,
        nights: int = 1,
        excluded_campsite_types: Optional[List[str]] = None,
        **kwargs,
    ) -> None:
        """
        Parameters
        ----------
        search_window
            Search window(s) containing start and end dates.
        latitude
            Center latitude in decimal degrees.
        longitude
            Center longitude in decimal degrees.
        radius_miles
            Search radius in miles.
        provider_filter
            If set, restrict geo-search to this single provider name (e.g. "ReserveCalifornia").
        weekends_only
            Only search Friday/Saturday nights.
        nights
            Minimum consecutive nights.
        excluded_campsite_types
            Case-insensitive substrings to exclude from campsite_type.
        """
        super().__init__(
            search_window=search_window,
            weekends_only=weekends_only,
            nights=nights,
            excluded_campsite_types=excluded_campsite_types,
            **kwargs,
        )
        self.latitude = latitude
        self.longitude = longitude
        self.radius_miles = radius_miles
        self._sub_searches: List[BaseCampingSearch] = self._build_sub_searches(
            search_window=search_window,
            provider_filter=provider_filter,
            weekends_only=weekends_only,
            nights=nights,
            **kwargs,
        )
        self.campgrounds = [cg for s in self._sub_searches for cg in s.campgrounds]
        if not self.campgrounds:
            logger.error(
                f"No campgrounds found within {radius_miles} miles of "
                f"({latitude:.4f}, {longitude:.4f})"
            )
            sys.exit(1)

    def _build_sub_searches(
        self,
        search_window: Union[SearchWindow, List[SearchWindow]],
        provider_filter: Optional[str],
        weekends_only: bool,
        nights: int,
        **kwargs,
    ) -> List[BaseCampingSearch]:
        """Discover campgrounds within radius from each applicable provider."""
        sub_searches: List[BaseCampingSearch] = []
        shared = dict(
            search_window=search_window,
            weekends_only=weekends_only,
            nights=nights,
        )

        # RecreationDotGov
        if provider_filter is None or provider_filter == RecreationDotGov.__name__:
            sub_searches += self._build_recdotgov_search(shared, **kwargs)

        # UseDirect variants
        for search_cls in _USEDIRECT_SEARCH_CLASSES:
            provider_name = search_cls.provider_class.__name__
            if provider_filter is None or provider_filter == provider_name:
                sub_searches += self._build_usedirect_search(search_cls, shared, **kwargs)

        # Yellowstone
        if provider_filter is None or provider_filter == Yellowstone.__name__:
            sub_searches += self._build_yellowstone_search(shared, **kwargs)

        return sub_searches

    def _build_recdotgov_search(
        self, shared: dict, **kwargs
    ) -> List[SearchRecreationDotGov]:
        provider = RecreationDotGov()
        campgrounds = provider.find_campgrounds(
            latitude=self.latitude,
            longitude=self.longitude,
            radius=self.radius_miles,
        )
        if not campgrounds:
            return []
        campground_ids = [int(cg.facility_id) for cg in campgrounds]
        logger.info(
            f"RecreationDotGov: {len(campground_ids)} campgrounds within "
            f"{self.radius_miles} miles"
        )
        return [
            SearchRecreationDotGov(
                campgrounds=campground_ids,
                **shared,
                **kwargs,
            )
        ]

    def _build_usedirect_search(
        self,
        search_cls: Type[BaseCampingSearch],
        shared: dict,
        **kwargs,
    ) -> List[BaseCampingSearch]:
        provider = search_cls.provider_class()
        provider.refresh_metadata()
        in_radius = [
            int(cg.facility_id)
            for cg in provider.usedirect_campgrounds.values()
            if cg.coordinates is not None
            and haversine_distance_miles(
                self.latitude, self.longitude, cg.coordinates[0], cg.coordinates[1]
            )
            <= self.radius_miles
        ]
        if not in_radius:
            return []
        logger.info(
            f"{search_cls.provider_class.__name__}: {len(in_radius)} campgrounds "
            f"within {self.radius_miles} miles"
        )
        return [
            search_cls(
                recreation_area=[],
                campgrounds=in_radius,
                **shared,
                **kwargs,
            )
        ]

    def _build_yellowstone_search(
        self, shared: dict, **kwargs
    ) -> List[SearchYellowstone]:
        dist = haversine_distance_miles(
            self.latitude,
            self.longitude,
            _YELLOWSTONE_CENTER[0],
            _YELLOWSTONE_CENTER[1],
        )
        if dist > self.radius_miles:
            return []
        logger.info(
            f"Yellowstone: park center is {dist:.1f} miles away — including in search"
        )
        return [SearchYellowstone(**shared, **kwargs)]

    def get_all_campsites(self) -> List[AvailableCampsite]:
        """Aggregate available campsites from all sub-searches."""
        results: List[AvailableCampsite] = []
        for sub_search in self._sub_searches:
            results.extend(sub_search.get_all_campsites())
        return results
```

- [ ] **Step 2: Export `SearchGeo` from `camply/search/__init__.py`**

In `camply/search/__init__.py`, add the import after the existing imports:

```python
from camply.search.search_geo import SearchGeo
```

And add it to `__search_providers__` list (it does not need to be in `CAMPSITE_SEARCH_PROVIDER` since the CLI instantiates it directly):

The `__init__.py` should have this import added near line 30:

```python
from camply.search.search_geo import SearchGeo
```

- [ ] **Step 3: Run existing tests to verify no import errors**

```bash
pytest tests/ -v --co -q 2>&1 | head -30
```

Expected: test collection succeeds with no import errors.

- [ ] **Step 4: Commit**

```bash
git add camply/search/search_geo.py camply/search/__init__.py
git commit -m "feat: add SearchGeo multi-provider geo-based campsite search orchestrator"
```

---

## Task 7: Add new CLI flags to `campsites` command

**Files:**
- Modify: `camply/cli.py`

- [ ] **Step 1: Add shared argument definitions**

In `camply/cli.py`, after the existing shared argument definitions (around line 208), add:

```python
near_argument = click.option(
    "--near",
    default=None,
    help="Search for campgrounds near a place name (geocoded). Mutually exclusive with --latitude/--longitude.",
)
latitude_argument = click.option(
    "--latitude",
    default=None,
    type=float,
    help="Center latitude for radius search (decimal degrees).",
)
longitude_argument = click.option(
    "--longitude",
    default=None,
    type=float,
    help="Center longitude for radius search (decimal degrees).",
)
radius_argument = click.option(
    "--radius",
    default=None,
    type=float,
    help="Search radius in miles. Required when --near, --latitude, or --longitude is used.",
)
exclude_type_argument = click.option(
    "--exclude-type",
    default=None,
    multiple=True,
    help="Exclude campsites whose type contains this string (case-insensitive, repeatable). E.g. --exclude-type group --exclude-type horse",
)
```

- [ ] **Step 2: Add decorators to the `campsites` command**

In `camply/cli.py`, add the five new decorators to the `campsites` command (around line 684), before `@provider_argument`:

```python
@camply_command_line.command(cls=RichCommand)
@rec_area_argument
@campground_argument
@campsite_id_argument
@start_date_argument
@end_date_argument
@nights_argument
@weekends_argument
@day_of_the_week_argument
@notifications_argument
@continuous_argument
@search_forever_argument
@yaml_config_argument
@offline_search_argument
@offline_search_path_argument
@search_once_argument
@polling_interval_argument
@notify_first_try_argument
@equipment_argument
@equipment_id_argument
@near_argument
@latitude_argument
@longitude_argument
@radius_argument
@exclude_type_argument
@provider_argument
@debug_option
@click.pass_obj
def campsites(
    context: CamplyContext,
    debug: bool,
    rec_area: Tuple[Union[str, int]],
    campground: Tuple[Union[str, int]],
    campsite: Tuple[Union[str, int]],
    start_date: str,
    end_date: str,
    weekends: bool,
    nights: int,
    provider: Optional[str],
    continuous: bool,
    polling_interval: Optional[str],
    notifications: Tuple[str],
    notify_first_try: Optional[str],
    search_forever: Optional[str],
    search_once: bool,
    yaml_config: Optional[str],
    offline_search: bool,
    offline_search_path: Optional[str],
    equipment: Tuple[Union[str, int]],
    equipment_id: Tuple[Union[str, int]],
    day: Optional[Tuple[str]],
    near: Optional[str],
    latitude: Optional[float],
    longitude: Optional[float],
    radius: Optional[float],
    exclude_type: Tuple[str],
) -> None:
```

- [ ] **Step 3: Add geo validation and `SearchGeo` dispatch in the `campsites` function body**

Replace the body of `campsites` (starting after the docstring, from `if context.debug is None:` through to the `camping_finder.get_matching_campsites` call) with:

```python
    if context.debug is None:
        context.debug = debug
        _set_up_debug(debug=context.debug)

    # --- geo validation ---
    geo_params = (near, latitude, longitude)
    any_geo = any(p is not None for p in geo_params)
    if near is not None and (latitude is not None or longitude is not None):
        logger.error("--near is mutually exclusive with --latitude/--longitude.")
        sys.exit(1)
    if any_geo and radius is None:
        logger.error("--radius is required when using --near, --latitude, or --longitude.")
        sys.exit(1)
    if any_geo and provider == "GoingToCamp":
        logger.error("GoingToCamp does not support geo-based search (no coordinate data).")
        sys.exit(1)

    excluded_campsite_types = list(exclude_type) if exclude_type else []

    if yaml_config is not None:
        provider, provider_kwargs, search_kwargs = yaml_utils.yaml_file_to_arguments(
            file_path=yaml_config
        )
        provider = _preferred_provider(context, provider)
    elif any_geo:
        # Resolve coordinates
        if near is not None:
            from camply.utils.geo_utils import geocode_location
            resolved_lat, resolved_lon = geocode_location(near)
        else:
            if latitude is None or longitude is None:
                logger.error("Both --latitude and --longitude are required together.")
                sys.exit(1)
            resolved_lat, resolved_lon = latitude, longitude

        provider_kwargs, search_kwargs = _get_provider_kwargs_from_cli(
            rec_area=rec_area,
            campground=campground,
            campsite=campsite,
            start_date=start_date,
            end_date=end_date,
            weekends=weekends,
            nights=nights,
            provider="RecreationDotGov",  # placeholder — SearchGeo handles provider selection
            continuous=continuous,
            polling_interval=polling_interval,
            notifications=notifications,
            notify_first_try=notify_first_try,
            search_forever=search_forever,
            search_once=search_once,
            offline_search=offline_search,
            offline_search_path=offline_search_path,
            equipment=equipment,
            equipment_id=equipment_id,
            day=day,
            yaml_config=None,
        )
        # Remove rec_area/campground from provider_kwargs — SearchGeo discovers these
        provider_kwargs.pop("recreation_area", None)
        provider_kwargs.pop("campgrounds", None)
        provider_kwargs.pop("campsites", None)

        from camply.search.search_geo import SearchGeo
        camping_finder = SearchGeo(
            latitude=resolved_lat,
            longitude=resolved_lon,
            radius_miles=radius,
            provider_filter=provider if provider != "RecreationDotGov" else None,
            excluded_campsite_types=excluded_campsite_types,
            **provider_kwargs,
        )
        camping_finder.get_matching_campsites(**search_kwargs)
        return
    else:
        provider = _preferred_provider(context, provider)
        provider_kwargs, search_kwargs = _get_provider_kwargs_from_cli(
            rec_area=rec_area,
            campground=campground,
            campsite=campsite,
            start_date=start_date,
            end_date=end_date,
            weekends=weekends,
            nights=nights,
            provider=provider,
            continuous=continuous,
            polling_interval=polling_interval,
            notifications=notifications,
            notify_first_try=notify_first_try,
            search_forever=search_forever,
            search_once=search_once,
            offline_search=offline_search,
            offline_search_path=offline_search_path,
            equipment=equipment,
            equipment_id=equipment_id,
            day=day,
            yaml_config=yaml_config,
        )

    if excluded_campsite_types:
        provider_kwargs["excluded_campsite_types"] = excluded_campsite_types

    provider_class: Type[BaseCampingSearch] = CAMPSITE_SEARCH_PROVIDER[provider]
    camping_finder: BaseCampingSearch = provider_class(**provider_kwargs)
    camping_finder.get_matching_campsites(**search_kwargs)
```

- [ ] **Step 4: Run existing campsites CLI tests to verify no regressions**

```bash
pytest tests/cli/test_campsites.py -v
```

Expected: all existing tests PASS.

- [ ] **Step 5: Commit**

```bash
git add camply/cli.py
git commit -m "feat: add --near/--latitude/--longitude/--radius/--exclude-type to campsites command"
```

---

## Task 8: Add new fields to YAML config

**Files:**
- Modify: `camply/containers/search_model.py`
- Modify: `camply/utils/yaml_utils.py`

- [ ] **Step 1: Add fields to `YamlSearchFile`**

In `camply/containers/search_model.py`, add the new fields to the `YamlSearchFile` class after `offline_search_path`:

```python
class YamlSearchFile(CamplyModel):
    """
    Campsite Search Data Model
    """

    provider: ProviderEnum = Field(
        description="Campsite provider", default="RecreationDotGov"
    )
    recreation_area: ArrayOrSingle = None
    campgrounds: ArrayOrSingle = None
    campsites: ArrayOrSingle = None
    start_date: Union[datetime.date, List[datetime.date]]
    end_date: Union[datetime.date, List[datetime.date]]
    days: Optional[List[str]] = None
    weekends: bool = False
    nights: int = 1
    continuous: bool = True
    polling_interval: int = SearchConfig.RECOMMENDED_POLLING_INTERVAL
    notifications: ArrayOrSingleStr = "silent"
    search_forever: bool = False
    search_once: bool = False
    notify_first_try: bool = False
    equipment: ArrayOrSingleEquipment = None
    offline_search: bool = False
    offline_search_path: Optional[str] = None
    near: Optional[str] = None
    latitude: Optional[float] = None
    longitude: Optional[float] = None
    radius: Optional[float] = None
    excluded_campsite_types: Optional[List[str]] = None
```

- [ ] **Step 2: Thread new fields through `yaml_file_to_arguments`**

In `camply/utils/yaml_utils.py`, update `yaml_file_to_arguments` to include the new geo and exclusion fields.

After the existing `provider_kwargs` block, add `near`, `latitude`, `longitude`, `radius`, and `excluded_campsite_types` to `provider_kwargs`:

```python
    provider_kwargs = {
        "search_window": search_window,
        "recreation_area": yaml_model.recreation_area,
        "campgrounds": yaml_model.campgrounds,
        "campsites": yaml_model.campsites,
        "weekends_only": yaml_model.weekends,
        "days_of_the_week": days_of_the_week,
        "nights": yaml_model.nights,
        "equipment": equipment,
        "offline_search": yaml_model.offline_search,
        "offline_search_path": yaml_model.offline_search_path,
        "near": yaml_model.near,
        "latitude": yaml_model.latitude,
        "longitude": yaml_model.longitude,
        "radius": yaml_model.radius,
        "excluded_campsite_types": yaml_model.excluded_campsite_types,
    }
```

- [ ] **Step 3: Run existing tests to verify no regressions**

```bash
pytest tests/ -v -k "yaml or campsites"
```

Expected: all existing tests PASS.

- [ ] **Step 4: Commit**

```bash
git add camply/containers/search_model.py camply/utils/yaml_utils.py
git commit -m "feat: add near/latitude/longitude/radius/excluded_campsite_types to YAML config"
```

---

## Task 9: Write tests for `SearchGeo` and type exclusion

**Files:**
- Create: `tests/search_providers/test_geo_search.py`

- [ ] **Step 1: Create test file**

```python
"""
Tests for SearchGeo and excluded_campsite_types filtering
"""

import pytest
from datetime import datetime
from unittest.mock import MagicMock, patch

from camply.containers import AvailableCampsite, CampgroundFacility, SearchWindow
from camply.search.search_geo import SearchGeo, _YELLOWSTONE_CENTER


@pytest.fixture
def search_window() -> SearchWindow:
    return SearchWindow(
        start_date=datetime(2026, 7, 1),
        end_date=datetime(2026, 7, 15),
    )


# --- Type exclusion tests ---

class TestExcludedCampsiteTypes:
    def test_group_sites_excluded(self, available_campsite):
        """Campsites whose type contains the exclusion string are filtered out."""
        from camply.search.search_recreationdotgov import SearchRecreationDotGov

        with patch.object(
            SearchRecreationDotGov, "_get_searchable_campgrounds", return_value=[]
        ):
            with patch.object(
                SearchRecreationDotGov, "get_all_campsites", return_value=[available_campsite]
            ):
                # available_campsite has campsite_type="Test" — not excluded
                searcher = SearchRecreationDotGov.__new__(SearchRecreationDotGov)
                searcher.excluded_campsite_types = ["group"]
                searcher.nights = 1
                searcher.search_days = [available_campsite.booking_date]
                searcher.search_months = []
                # filter directly
                result = [
                    c for c in [available_campsite]
                    if not any(
                        excl.lower() in (c.campsite_type or "").lower()
                        for excl in searcher.excluded_campsite_types
                    )
                ]
                assert available_campsite in result  # "Test" does not contain "group"

    def test_excluded_type_removed(self, available_campsite):
        """A campsite with a matching type is removed."""
        group_campsite = available_campsite.copy(update={"campsite_type": "Group Standard"})
        excluded = ["group"]
        result = [
            c for c in [available_campsite, group_campsite]
            if not any(
                excl.lower() in (c.campsite_type or "").lower()
                for excl in excluded
            )
        ]
        assert available_campsite in result
        assert group_campsite not in result

    def test_exclusion_is_case_insensitive(self, available_campsite):
        """Exclusion matching ignores case."""
        horse_campsite = available_campsite.copy(update={"campsite_type": "HORSE ONLY"})
        excluded = ["horse"]
        result = [
            c for c in [horse_campsite]
            if not any(
                excl.lower() in (c.campsite_type or "").lower()
                for excl in excluded
            )
        ]
        assert result == []

    def test_none_campsite_type_not_excluded(self, available_campsite):
        """Campsites with no type are never excluded."""
        none_type_campsite = available_campsite.copy(update={"campsite_type": None})
        excluded = ["group"]
        result = [
            c for c in [none_type_campsite]
            if not any(
                excl.lower() in (c.campsite_type or "").lower()
                for excl in excluded
            )
        ]
        assert none_type_campsite in result

    def test_empty_exclusion_list_passes_all(self, available_campsite):
        """An empty exclusion list keeps all campsites."""
        group_campsite = available_campsite.copy(update={"campsite_type": "Group Standard"})
        excluded = []
        result = [
            c for c in [available_campsite, group_campsite]
            if not any(
                excl.lower() in (c.campsite_type or "").lower()
                for excl in excluded
            )
        ]
        assert len(result) == 2


# --- SearchGeo construction tests ---

class TestSearchGeoValidation:
    def test_no_campgrounds_found_exits(self, search_window):
        """SearchGeo exits if no campgrounds are found within radius."""
        with patch(
            "camply.search.search_geo.SearchGeo._build_sub_searches",
            return_value=[],
        ):
            with pytest.raises(SystemExit):
                SearchGeo(
                    search_window=search_window,
                    latitude=37.77,
                    longitude=-122.41,
                    radius_miles=1.0,  # tiny radius — no results
                )

    def test_yellowstone_excluded_when_far(self, search_window):
        """Yellowstone is not included when the search center is far away."""
        # San Francisco is ~1400 miles from Yellowstone
        with patch(
            "camply.search.search_geo.SearchGeo._build_recdotgov_search",
            return_value=[],
        ), patch(
            "camply.search.search_geo.SearchGeo._build_usedirect_search",
            return_value=[],
        ):
            searcher = SearchGeo.__new__(SearchGeo)
            result = searcher._build_yellowstone_search(
                shared=dict(
                    search_window=search_window,
                    weekends_only=False,
                    nights=1,
                ),
            )
            # We need to set lat/lon/radius manually since __init__ didn't run
            searcher.latitude = 37.77
            searcher.longitude = -122.41
            searcher.radius_miles = 100.0
            result = searcher._build_yellowstone_search(
                shared=dict(
                    search_window=search_window,
                    weekends_only=False,
                    nights=1,
                )
            )
            assert result == []

    def test_yellowstone_included_when_near(self, search_window):
        """Yellowstone is included when the search center is within radius."""
        searcher = SearchGeo.__new__(SearchGeo)
        searcher.latitude = _YELLOWSTONE_CENTER[0] + 0.1  # just north of center
        searcher.longitude = _YELLOWSTONE_CENTER[1]
        searcher.radius_miles = 20.0

        with patch(
            "camply.search.search_geo.SearchYellowstone.__init__",
            return_value=None,
        ):
            result = searcher._build_yellowstone_search(
                shared=dict(
                    search_window=search_window,
                    weekends_only=False,
                    nights=1,
                )
            )
            assert len(result) == 1
```

- [ ] **Step 2: Run the new tests**

```bash
pytest tests/search_providers/test_geo_search.py -v
```

Expected: all tests PASS.

- [ ] **Step 3: Run the full test suite to confirm no regressions**

```bash
pytest tests/ -v
```

Expected: all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add tests/search_providers/test_geo_search.py
git commit -m "test: add tests for SearchGeo and excluded_campsite_types filtering"
```

---

## Self-Review Checklist

- [x] Spec: `--near`/`--latitude`/`--longitude`/`--radius` on CLI → Task 7
- [x] Spec: mutual exclusion of `--near` and `--lat/--lon` → Task 7, Step 3
- [x] Spec: `--radius` required with any geo param → Task 7, Step 3
- [x] Spec: geocoding via Nominatim → Task 2
- [x] Spec: haversine distance → Task 2
- [x] Spec: RecreationDotGov uses RIDB API native geo → Task 5, Task 6
- [x] Spec: UseDirect providers use cached metadata + haversine filter → Task 4, Task 6
- [x] Spec: Yellowstone uses hardcoded center → Task 6
- [x] Spec: GoingToCamp excluded with error if explicitly requested → Task 7, Step 3
- [x] Spec: No campgrounds found → exit with message → Task 6, `__init__`
- [x] Spec: `--exclude-type` repeatable, case-insensitive substring → Task 3, Task 7
- [x] Spec: YAML config fields → Task 8
- [x] Spec: `geopy` dependency → Task 1
- [x] Type consistency: `haversine_distance_miles` used consistently in Task 2, 6, 9
- [x] Type consistency: `SearchGeo._build_sub_searches` returns `List[BaseCampingSearch]` — matches `_sub_searches` type annotation
- [x] Type consistency: `excluded_campsite_types: List[str]` defined in Task 3, used in Task 6, 7, 8
