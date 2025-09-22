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
  ee.Initialize()

  # --- CONFIG ---
  ROI  = ee.Geometry.Rectangle([-67.5, 17.8, -65.1, 18.6])
  COLL = 'NASA/VIIRS/002/VNP21A1N'
  BAND = 'LST_1KM'  # change to 'LST' if needed
  SCALE = 1000
  COVERAGE_THRESHOLD = 0.1  # maximum % of masked pixels allowed before trying older image

  # --- helper: attach coverage fraction in [0,1] as 'coverage' property ---
  def with_coverage(img):
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
      print('Coverage fraction:', cov)

  # Auto range (raw values) from ROI
  stats = img.select(BAND).reduceRegion(
      reducer=ee.Reducer.minMax(),
      geometry=ROI,
      scale=SCALE,
      maxPixels=1e9,
      tileScale=4
  ).getInfo()
  vmin = stats[f'{BAND}_min']
  vmax = stats[f'{BAND}_max']
  print('Auto range (raw):', vmin, vmax)

  # Map
  vis = dict(min=vmin, max=vmax,
            palette=['040274','2359a1','66c2a5','ffd92f','f46d43','a50026'],
            bands=[BAND])
  m = geemap.Map(center=[18.2, -66.3], zoom=8)
  m.addLayer(img.clip(ROI), vis, f'Latest MODIS Terra LST (raw, >={COVERAGE_THRESHOLD*100}% coverage)')
  m


```
