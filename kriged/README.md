# Kriged water-level averages by HUC-8 and HUC-10

Precomputed, so the composite hydrograph tool only downloads a few kilobytes when a watershed is selected.
Place this `kriged/` folder next to the tool's HTML page. If it is missing there, the tool reads the
GitHub Pages copy at `https://steeltapejockey.github.io/Composite-Hydrograph-Dashboard/kriged/`.

## Files
- `index.json`: every HUC-8 and HUC-10 touching New Mexico (89 + 625), with each unit's status per season (`ok`, `unstable`, or `null` = no estimate) and model diagnostics per HUC-8.
- `winter.json` and `all.json`: the results for every HUC-8 and its HUC-10s, one file per season (under `huc8` → HUC-8 code).
- `huc10.geojson`: HUC-10 boundaries for the tool's HUC-10 picker (from the USGS Watershed Boundary Dataset).

## Values (per unit, one value per year 1950-2026)
- `chg_ft`: kriged change in the areal-mean water-level elevation from `base_year` (the HUC-8's best-measured year), in feet, positive = rising.
- `se_chg_ft`: standard error of that change, calibrated (95% range = ±1.96 × se).
- `wle_ft`, `se_ft`: the areal-mean water-level elevation itself and its standard error. These ranges are much wider because they include uncertainty that is the same every year.
- `cover_km2` / `area_km2`: the averages cover only the part of the unit within 5 km of a well.
- `zigzag_factor`, `unstable`, `note`: see the quality checks below.

## Seasons
- `winter`: readings from November to February only, as in NMBGMR OFR-616, to avoid irrigation pumping. Winter Y = Nov-Dec of Y-1 + Jan-Feb of Y.
- `all`: every month, calendar year. More wells, but irrigation-season drawdown adds noise.

## Method
After Rawling, G., 2021 (rev. 2022), NMBGMR Open-File Report 616, adapted to run for every basin:
1. **Data:** the NM Unified Water Data API (all wells and depth-to-water readings, pulled 2026-11-01), plus 27,838 readings from NMBGMR open-file reports that the API lacks (OFR water levels vs API workbook). Readings with pumping or other disturbance qualifiers are dropped.
2. **Values and elevations:** one value per well per year (median). Water-level elevation = site elevation from the API minus depth to water. USGS 3DEP elevation (10 m) is used only for the 748 wells without an API elevation. Wells at the same location (to about 10 m) are merged.
3. **Model data:** one model per HUC-8 and season, using wells inside the HUC-8 or within 10 km of it.
4. **Regional trend:** a polynomial of the first two principal components of easting, northing and land elevation, constant in time. Outliers are removed by OFR-616's rule.
5. **Variogram:** a sum-metric space-time variogram fitted to lags up to 30 years. It has floors: a year-to-year nugget of at least half the variation seen in the same wells one year apart, and spatial and joint nuggets of at least 1 ft².
6. **Calibration:** leave-one-out cross-validation at up to 1,500 readings. Error ranges are widened by the square root of the mean squared normalized error (at least 1), so about 95% of left-out readings fall inside their 95% range.
7. **Change:** block contrast kriging of the change from the base year, averaged over up to 64 points covering the unit. It uses each well's readings nearest in time (within 15 years), up to 2,500 readings.

## Quality checks and caveats
- `unstable`: the kriged curve zigzags from year to year more than its stated range allows. Typical causes are wells from different aquifers measured in different years, such as the shallow and artesian aquifers of the Roswell basin. Its ranges are widened by `zigzag_factor` and the tool shows a caution.
- Years far from any measurement have wide ranges. Early years (1950s) and the 2010s, which had few surveys in many basins, deserve care.
- Units without an estimate have no area within 5 km of a well, or their HUC-8 had too few wells or readings for a model.
- Kriging smooths. One well can dominate a sparsely measured area.

## Rebuild
`scripts/kriging/`: `pull_all.py` → `dem_all.py` → `prep_wells.py` → `units.py` → `run.py winter` and `run.py all` → `build_repo.py`. Python 3 with numpy, scipy, shapely and pyproj. About 1.5 hours on 2 cores.
