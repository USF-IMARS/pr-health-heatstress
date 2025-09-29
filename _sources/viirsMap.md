---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  name: python3          # or "ir" for R
  display_name: Python 3
---


# VIIRS LST from GEE Dataset 

```{code-cell} ipython3
:tags: [hide-input]   # folded by default; click to toggle
 
# pip install earthengine-api geemap
import ee, geemap

try:
    ee.Initialize()
except Exception as e:
    print("Earth Engine initialization failed. Please authenticate.", e)
    # Fallback for environments where ee.Authenticate() is needed
    ee.Authenticate()
    ee.Initialize()


# --- CONFIG ---
ROI  = ee.Geometry.Rectangle([-67.5, 17.8, -65.1, 18.6])
COLL = 'NASA/VIIRS/002/VNP21A1N'
BAND = 'LST_1KM'  # Land Surface Temperature band
SCALE = 1000
COVERAGE_THRESHOLD = 0.1  # maximum % of masked pixels allowed before trying older image

# --- helper: attach coverage fraction in [0,1] as 'coverage' property ---
def with_coverage(img):
    """Calculates the fraction of non-masked pixels for the ROI."""
    valid = img.select(BAND).mask().unmask(0)   # 1 for valid, 0 for masked
    frac = valid.reduceRegion(
        reducer=ee.Reducer.mean(),
        geometry=ROI,
        scale=SCALE,
        maxPixels=1e9,
        tileScale=4
    ).get(BAND)
    return img.set('coverage', frac)

from datetime import datetime, timedelta, timezone
NOW_STR = datetime.now(timezone.utc).strftime('%Y-%m-%d')
MIN_STR = (datetime.now(timezone.utc) - timedelta(days=61)).strftime('%Y-%m-%d')  # ~2 months

# Newest-first collection with coverage computed (last ~2 months only)
ic_all = (
    ee.ImageCollection(COLL)
      .filterBounds(ROI)
      .filterDate(MIN_STR, NOW_STR)
      .sort('system:time_start', False)
      .map(with_coverage)
)

good_ic  = ic_all.filter(ee.Filter.gte('coverage', COVERAGE_THRESHOLD))

# Pick newest good image; if none, fallback to newest overall
img = ee.Image(ee.Algorithms.If(good_ic.size().gt(0), good_ic.first(),
                                ic_all.first()))

# Info
date = ee.Date(img.get('system:time_start')).format('YYYY-MM-dd').getInfo()
cov  = ee.Number(img.get('coverage')).getInfo() if img.get('coverage') else None
print('Chosen image date:', date)
if cov is not None:
    print(f'Coverage fraction: {cov:.2%}')


# --- MODIFIED PLOTTING METHOD ---

# Create the map object first
m = geemap.Map(center=[18.2, -66.3], zoom=8)

# 1. Use percentiles for a more robust color stretch, less sensitive to outliers.
#    This is generally a better method than using absolute min/max.
stats = img.select(BAND).reduceRegion(
    reducer=ee.Reducer.percentile([2, 98]), # Calculate 2nd and 98th percentiles
    geometry=ROI,
    scale=SCALE,
    maxPixels=1e9,
    tileScale=4
).getInfo()

# 2. Check if stats were successfully calculated. If not, the image is likely empty.
#    This prevents the script from trying to add an empty layer.
if stats and stats.get(f'{BAND}_p2') is not None and stats.get(f'{BAND}_p98') is not None:
    vmin = stats[f'{BAND}_p2']
    vmax = stats[f'{BAND}_p98']
    print('Auto range (2nd to 98th percentile):', vmin, vmax)

    # Define visualization parameters with the calculated percentiles
    vis_params = {
        'min': vmin,
        'max': vmax,
        'palette': ['040274','2359a1','66c2a5','ffd92f','f46d43','a50026'],
        'bands': [BAND]
    }
    
    # Add the layer to the map
    m.addLayer(
        img.clip(ROI),
        vis_params,
        f'VIIRS LST ({date})'
    )
    print(f"\nSuccessfully added layer for date: {date}")

else:
    # This block executes if the chosen image has no valid pixels in the ROI
    print('\nWarning: The chosen image is empty for the given ROI.')
    print('No data layer will be added to the map.')

# Display the map. In a Jupyter environment, this line will render the map.
# If running as a script, you might want to save it: m.to_html('map.html')
m

```
