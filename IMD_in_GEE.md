### In Google Earth Engine (GEE), exporting an entire `ImageCollection` directly to an asset in a single operation `isn't supported`. However, you can export each image within the collection individually by iterating over the collection and using `Export.image.toAsset()` for each image.
---

## 1. Selecting the Region of Interest

```javascript
var geometry = ee.Geometry.Point([85.8949998359892, 20.462094851812726]); // Can use Polygon or Shapefile 
Map.centerObject(geometry, 16);
```

- **Geometry Point**: Defines a geographic location using longitude (85.89499) and latitude (20.46209)
- **Map.centerObject**: Pans and zooms the map to focus on the given geometry. The zoom level is set to 16 (1 is global, higher values zoom in)

## 2. Defining Years of Interest

```javascript
var years = [];
for (var year = 2020; year <= 2023; year++) {
  years.push(year);
}
```

- **Dynamic List Creation**: Uses a for loop to create a list of years from 2020 to 2023, ensuring script scalability for different year ranges
---
## 3. Handling Rainfall Data

### a. Accessing Yearly Data

```javascript
var y = ee.Image("projects/pulakeshpradhan/assets/IMD/RAIN/Rain_" + year);
```

- **ee.Image**: References a single raster dataset for rainfall data
- **String Concatenation**: Dynamically builds the asset path (Rain_2020, Rain_2021, etc.)

### b. Extracting Daily Bands

```javascript
var bandNames = y.bandNames();
```

- **Band Names**: Each band corresponds to a specific day (B1, B2, ..., B365)
- The `.bandNames()` function retrieves these names for processing

### c. Calculating Dates from Bands

```javascript
var startDate = ee.Date(year + '-01-01');
var index = ee.Number.parse(ee.String(name).slice(1)).subtract(1);
var date = startDate.advance(index, 'day');
```

- **Start Date**: Uses year beginning (YYYY-01-01) as reference
- **Band-to-Date Mapping**:
  - Extracts numeric part of band name
  - Adjusts for zero-based indexing
  - Advances start date by calculated days

### d. Adding Time Properties

```javascript
bandImage = bandImage.set('system:time_start', date.millis());
```

- **Timestamping**: Adds UNIX timestamp metadata for time-based operations

### e. Creating a Yearly Image Collection

```javascript
var images = bandNames.map(function(name) { ... });
var collection = ee.ImageCollection.fromImages(images);
```

- Combines daily images into a yearly collection for efficient processing

### f. Renaming Bands

```javascript
var renamedCollection = collection.map(function(image) {
  return image.rename('rain');
});
```

- Standardizes band names to 'rain' for uniformity

### g. Merging Yearly Collections

```javascript
var mergedRainCollection = ee.ImageCollection(rainCollections[0]);
for (var i = 1; i < rainCollections.length; i++) {
  mergedRainCollection = mergedRainCollection.merge(rainCollections[i]);
}
```

- Combines all yearly collections into a single multi-year dataset
---
## 4. Temperature Data Processing (Tmax and Tmin)

Process follows rainfall pattern:
- Fetch yearly datasets (Tmax_<year> and Tmin_<year>)
- Timestamp daily bands
- Rename to 'tmax' and 'tmin'
- Merge into respective collections

## 5. Monthly Aggregation

### a. Filtering by Year and Month

```javascript
var monthlyRain = mergedRainCollection
  .filter(ee.Filter.calendarRange(year, year, 'year'))
  .filter(ee.Filter.calendarRange(month, month, 'month'));
```

- Uses calendarRange filter for temporal data selection

### b. Aggregating Rainfall

```javascript
var totalRain = monthlyRain.sum().rename('Rain');
```

- Calculates total monthly rainfall

### c. Aggregating Temperature

```javascript
var averageTmax = monthlyTmax.mean().rename('Tmax');
var averageTmin = monthlyTmin.mean().rename('Tmin');
```

- Calculates monthly temperature averages

### d. Combining Metrics

```javascript
var image = totalRain.addBands(averageTmax).addBands(averageTmin).set({
  'system:time_start': ee.Date.fromYMD(year, month, 1).millis(),
  'year': year,
  'month': month
});
```

- Creates composite image with all metrics
- Adds temporal metadata

## 6. Visualization

### Time-Series Chart Creation

```javascript
var chart = ui.Chart.image.series({
  imageCollection: monthlyCol,
  region: cuttack,
  reducer: ee.Reducer.mean(),
  scale: 1000,
  xProperty: 'system:time_start'
})
.setChartType('ComboChart')
.setOptions({
  interpolateNulls: true,
  title: 'Total Monthly Rainfall vs. Mean Monthly Tmax and Tmin: ' + year,
});
```

- Creates combo chart combining:
  - Rainfall (bars)
  - Temperature (lines)
- Customizes axes and styling for clarity

## 7. Technical Summary

### Key Features
- Dynamic programming with year/month parameters
- Efficient temporal data handling
- GEE-optimized operations
- Integrated visualization tools






# Daily Data Processing and Zonal Statistics Guide

## 1. Handling Daily Data Aggregation

### a. Defining the Time Range

```javascript
var startDate = ee.Date.fromYMD(2021, 1, 1);
var endDate = ee.Date.fromYMD(2023, 12, 31);
var days = ee.List.sequence(0, endDate.difference(startDate, 'day').subtract(1));
```

- **Time Range**: Start date is 2021-01-01, and end date is 2023-12-31
- **Day Sequence**:
  - `ee.List.sequence` generates a list of numbers representing each day
  - `endDate.difference(startDate, 'day')` calculates total days
  - Subtracting 1 aligns indices correctly for mapping

### b. Aggregating Daily Data

```javascript
var byDay = days.map(function(day) {
  var date = startDate.advance(day, 'day');
  var dayStart = date.millis();
```

- **Mapping Over Days**:
  - Calculates specific date using `startDate.advance(day, 'day')`
  - `dayStart` stores timestamp in milliseconds for temporal alignment

### c. Processing Rainfall, Tmax, and Tmin

```javascript
var dailyRain = mergedRainCollection
  .filter(ee.Filter.date(date, date.advance(1, 'day')));
var totalRain = dailyRain.sum().rename('Rain');
```

- **Filtering by Date**: Filters merged dataset for current day's images
- **Summing Rainfall**: `sum()` aggregates raster values for daily total

```javascript
var dailyTmax = mergedTmaxCollection
  .filter(ee.Filter.date(date, date.advance(1, 'day')));
var averageTmax = dailyTmax.mean().rename('Tmax');
```

- **Mean Tmax**: Computes average daily maximum temperature

```javascript
var dailyTmin = mergedTminCollection
  .filter(ee.Filter.date(date, date.advance(1, 'day')));
var averageTmin = dailyTmin.mean().rename('Tmin');
```

- **Mean Tmin**: Computes average daily minimum temperature

### d. Creating a Combined Daily Image

```javascript
var image = totalRain.addBands(averageTmax).addBands(averageTmin).set({
  'system:time_start': dayStart,
  'date': date.format('yyyy-MM-dd')
});
```

- **Combining Metrics**:
  - Merges daily rainfall, Tmax, and Tmin into single multi-band image
  - Adds metadata (timestamp and formatted date)

## 2. Creating the Daily Image Collection

```javascript
var dailyCol = ee.ImageCollection.fromImages(byDay);
```

- **Daily Image Collection**:
  - Converts daily images list to ImageCollection
  - Enables spatial/temporal operations on entire dataset

## 3. Zonal Statistics for Districts

### a. Defining the Region of Interest

```javascript
var region = ee.FeatureCollection('projects/pulakeshpradhan/assets/shp/Wb_District_new');
```

- **Region Input**: Loads district boundaries shapefile as FeatureCollection

### b. Reducing Data by Region

```javascript
var zonalStats = dailyCol.map(function(image) {
  var stats = image.reduceRegions({
    collection: region,
    reducer: ee.Reducer.mean(),
    scale: 1000,
  });
```

- **reduceRegions**:
  - Computes mean of each band for every feature
  - Uses 1 km resolution for aggregation

### c. Adding Metadata

```javascript
return stats.map(function(feature) {
  return feature.set({
    'date': image.get('date'),
    'Rain': feature.get('Rain'),
    'Tmax': feature.get('Tmax'),
    'Tmin': feature.get('Tmin'),
    'DISTRICT': feature.get('DISTRICT')
  });
});
```

- Adds daily metrics to each feature
- Preserves district identification

### d. Flattening Results

```javascript
}).flatten();
```

- Converts multiple collections into single feature collection
- Creates one row per district-day combination

## 4. Preparing Data for Export

```javascript
var selectedStats = zonalStats.select(['DISTRICT', 'date', 'Rain', 'Tmax', 'Tmin']);
```

- Retains only relevant fields for output

## 5. Exporting to CSV

```javascript
Export.table.toDrive({
  collection: selectedStats,
  description: 'ZonalTimeSeries',
  fileFormat: 'CSV',
  selectors: ['DISTRICT', 'date', 'Rain', 'Tmax', 'Tmin']
});
```

- **Export Options**:
  - Specifies collection to export
  - Names task
  - Sets CSV format
  - Defines output fields

## 6. Output and Use Cases

### Output Format
- CSV rows contain daily metrics per district
- Covers specified time period

### Applications
- Weather pattern monitoring
- Climate studies
- Disaster risk assessment
- Predictive analysis input
- Report generation

The workflow supports scaling for:
- Larger regions
- Extended time periods
- Efficient spatial-temporal integration
### Processing Flow
1. Data access and preparation
2. Temporal aggregation
3. Multi-metric combination
4. Visual analysis and interpretation
