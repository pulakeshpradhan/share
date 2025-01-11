

```javascript
var years = ['2020', '2021', '2022'];
var allCollections = [];

years.forEach(function(year) {
  var y = ee.Image("projects/pulakeshpradhan/assets/IMD/RAIN/Rain_" + year);
  var bandNames = y.bandNames(); // Get the band names
  var startDate = ee.Date(year + '-01-01'); // Set the start date

  var images = bandNames.map(function(name){
    var bandImage = y.select([name]);
    var index = ee.Number.parse(ee.String(name).slice(1)).subtract(1); // Use ee.String.slice
    var date = startDate.advance(index, 'day');
    bandImage = bandImage.set('system:time_start', date.millis());
    return bandImage;
  });

  var collection = ee.ImageCollection.fromImages(images);

  var renamedCollection = collection.map(function(image) {
    return image.rename('b1');
  });

  allCollections.push(renamedCollection);
});

// Merge all collections into a single collection
var mergedCollection = ee.ImageCollection(allCollections[0]);
for (var i = 1; i < allCollections.length; i++) {
  mergedCollection = mergedCollection.merge(allCollections[i]);
}

print(mergedCollection);
Map.addLayer(mergedCollection);




var chart = ui.Chart.image.series({
  imageCollection: mergedCollection.select(['b1'], ['rainfall']),
  region: geometry,
  reducer: ee.Reducer.mean(),
  scale: 4638.3
}).setChartType('ColumnChart')
  .setOptions({
      title: 'IMD Daily Rainfall Time-Series',
      vAxis: {title: 'Rainfall (mm)'},
      hAxis: {title: '', format: 'YYYY-MMM', gridlines: {count: 12}},
      series: {
        0: {color: 'blue'}
      },
    })

// Print the chart
print(chart);


```
