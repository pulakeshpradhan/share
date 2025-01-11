

```javascript
// Select a location
// var geometry = ee.Geometry.Point([85.8949998359892, 20.462094851812726]);
// Map.centerObject(geometry, 16);

var years = [];
for (var year = 2020; year <= 2023; year++) {
  years.push(year);
}

// var stringYears = years.map(function(year) {
//   return '"' + year.toString() + '"';
// });

// print(stringYears);


// RAIN COLLECTION

//***********************************************
var rainCollections = [];

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
    return image.rename('rain');
  });

  rainCollections.push(renamedCollection);
});

// Merge all rain collections into a single collection 
var mergedRainCollection = ee.ImageCollection(rainCollections[0]);
for (var i = 1; i < rainCollections.length; i++) {
  mergedRainCollection = mergedRainCollection.merge(rainCollections[i]);
}

print('Merged Rain Collection', mergedRainCollection);
Map.addLayer(mergedRainCollection, {}, 'Rain Collection');


// TMAX COLLECTION
//***********************************************
var tmaxCollections = [];

years.forEach(function(year) {
  var y = ee.Image("projects/pulakeshpradhan/assets/IMD/TMAX/Tmax_" + year);
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
    return image.rename('tmax');
  });

  tmaxCollections.push(renamedCollection);
});

// Merge all Tmax collections into a single collection 
var mergedTmaxCollection = ee.ImageCollection(tmaxCollections[0]);
for (var i = 1; i < tmaxCollections.length; i++) {
  mergedTmaxCollection = mergedTmaxCollection.merge(tmaxCollections[i]);
}

print('Merged Tmax Collection', mergedTmaxCollection);
Map.addLayer(mergedTmaxCollection, {}, 'Tmax Collection');


// TMIN COLLECTION
//***********************************************
var tminCollections = [];

years.forEach(function(year) {
  var y = ee.Image("projects/pulakeshpradhan/assets/IMD/TMIN/Tmin_" + year);
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
    return image.rename('tmin');
  });

  tminCollections.push(renamedCollection);
});

// Merge all Tmin collections into a single collection 
var mergedTminCollection = ee.ImageCollection(tminCollections[0]);
for (var i = 1; i < tminCollections.length; i++) {
  mergedTminCollection = mergedTminCollection.merge(tminCollections[i]);
}

print('Merged Tmin Collection', mergedTminCollection);
Map.addLayer(mergedTminCollection, {}, 'Tmin Collection');








// //***********************************************************************MONthly******
// // Define the months and the year
// var months = ee.List.sequence(1, 12);
// var year = 2022; // Set the year you're interested in

// // Function to aggregate daily data into monthly totals or averages
// var byMonth = months.map(function(month) {
//     // Total monthly rainfall
//     var monthlyRain = mergedRainCollection
//       .filter(ee.Filter.calendarRange(year, year, 'year'))
//       .filter(ee.Filter.calendarRange(month, month, 'month'));
//     var totalRain = monthlyRain.sum().rename('Rain');

//     // Average Tmax for the month
//     var monthlyTmax = mergedTmaxCollection
//       .filter(ee.Filter.calendarRange(year, year, 'year'))
//       .filter(ee.Filter.calendarRange(month, month, 'month'));
//     var averageTmax = monthlyTmax.mean().rename('Tmax');

//     // Average Tmin for the month
//     var monthlyTmin = mergedTminCollection
//       .filter(ee.Filter.calendarRange(year, year, 'year'))
//       .filter(ee.Filter.calendarRange(month, month, 'month'));
//     var averageTmin = monthlyTmin.mean().rename('Tmin');

//     // Combine Rain, Tmax, and Tmin into one image and set the correct time property
//     var image = totalRain.addBands(averageTmax).addBands(averageTmin).set({
//       'system:time_start': ee.Date.fromYMD(year, month, 1).millis(),
//       'year': year,
//       'month': month
//     });

//     return image;
// });

// // Create the monthly image collection from the images
// var monthlyCol = ee.ImageCollection.fromImages(byMonth);

// print('Monthly Collection with Rain, Tmax, Tmin:', monthlyCol);

// // Now create a time-series chart
// // Rain as bars, Tmax and Tmin as lines
// var chart = ui.Chart.image.series({
//   imageCollection: monthlyCol,
//   region: cuttack,  // Update with your region of interest
//   reducer: ee.Reducer.mean(),
//   scale: 1000,  // Adjust scale for your region and data resolution
//   xProperty: 'system:time_start'  // Ensure the chart uses system:time_start for the X-axis
// }).setChartType('ComboChart')
//   .setOptions({
//     interpolateNulls: true,
//     title: 'Total Monthly Rainfall vs. Mean Monthly Tmax and Tmin: ' + year,
//     lineWidth: 1,
//     pointSize: 5,
//     vAxes: {
//       // Axis for Rainfall
//       0: {title: 'Precipitation (mm)', gridlines: {count: 5}, viewWindow: {min: 0},
//           titleTextStyle: { bold: true, color: '#737373' }},
//       // Axis for Tmax and Tmin
//       1: {title: 'Temperature (°C)', gridlines: {color: 'none'}, viewWindow: {min: 0},
//           titleTextStyle: { bold: true, color: '#2b8cbe' }},
//     },
//     hAxis: {
//       gridlines: {color: 'none'}
//     },
//     series: {
//       0: {  // Rainfall as bars
//         type: 'bars',
//         color: '#737373',
//         targetAxisIndex: 0
//       },
//       1: {  // Tmax as line
//         type: 'line',
//         color: 'red',
//         lineWidth: 2,
//         pointShape: 'circle',
//         pointSize: 5,
//         targetAxisIndex: 1
//       },
//       2: {  // Tmin as line
//         type: 'line',
//         color: 'blue',
//         lineWidth: 2,
//         pointShape: 'square',
//         pointSize: 5,
//         targetAxisIndex: 1
//       }
//     },
//   });

// print(chart);






//***********************************************************************DAily******
// Define the days of the year
var startDate = ee.Date.fromYMD(2022, 1, 1);
var endDate = startDate.advance(2, 'year');
var days = ee.List.sequence(0, endDate.difference(startDate, 'day').subtract(1));

// Function to aggregate daily data
var byDay = days.map(function(day) {
  var date = startDate.advance(day, 'day');
  var dayStart = date.millis();
  var dayEnd = date.advance(1, 'day').millis();

  // Total daily rainfall
  var dailyRain = mergedRainCollection
    .filter(ee.Filter.date(date, date.advance(1, 'day')));
  var totalRain = dailyRain.sum().rename('Rain');

  // Average Tmax for the day
  var dailyTmax = mergedTmaxCollection
    .filter(ee.Filter.date(date, date.advance(1, 'day')));
  var averageTmax = dailyTmax.mean().rename('Tmax');

  // Average Tmin for the day
  var dailyTmin = mergedTminCollection
    .filter(ee.Filter.date(date, date.advance(1, 'day')));
  var averageTmin = dailyTmin.mean().rename('Tmin');

  // Combine Rain, Tmax, and Tmin into one image and set the correct time property
  var image = totalRain.addBands(averageTmax).addBands(averageTmin).set({
    'system:time_start': dayStart,
    'year': date.get('year'),
    'month': date.get('month'),
    'day': date.get('day')
  });

  return image;
});

// Create the daily image collection from the images
var dailyCol = ee.ImageCollection.fromImages(byDay);

print('Daily Collection with Rain, Tmax, Tmin:', dailyCol);

// Now create a time-series chart
// Rain as bars, Tmax and Tmin as lines
var chart = ui.Chart.image.series({
  imageCollection: dailyCol,
  region: cuttack,  // Update with your region of interest
  reducer: ee.Reducer.mean(),
  scale: 1000,  // Adjust scale for your region and data resolution
  xProperty: 'system:time_start'  // Ensure the chart uses system:time_start for the X-axis
}).setChartType('ComboChart')
  .setOptions({
    interpolateNulls: true,
    title: 'Total Daily Rainfall vs. Mean Daily Tmax and Tmin: ' + year,
    lineWidth: 1,
    pointSize: 2,
    vAxes: {
      // Axis for Rainfall
      0: {title: 'Precipitation (mm)', gridlines: {count: 5}, viewWindow: {min: 0},
          titleTextStyle: { bold: true, color: '#737373' }},
      // Axis for Tmax and Tmin
      1: {title: 'Temperature (°C)', gridlines: {color: 'none'}, viewWindow: {min: 0},
          titleTextStyle: { bold: true, color: '#2b8cbe' }},
    },
    hAxis: {
      gridlines: {color: 'none'}
    },
    series: {
      0: {  // Rainfall as bars
        type: 'bars',
        color: '#737373',
        targetAxisIndex: 0
      },
      1: {  // Tmax as line
        type: 'line',
        color: 'red',
        lineWidth: 2,
        pointShape: 'circle',
        pointSize: 5,
        targetAxisIndex: 1
      },
      2: {  // Tmin as line
        type: 'line',
        color: 'blue',
        lineWidth: 2,
        pointShape: 'square',
        pointSize: 5,
        targetAxisIndex: 1
      }
    },
  });

print(chart);

/////////////////////////////=== CSV === \\\\\\\\\\\\\\\\\\\\\\\\\\\\\


//*********************************************************** Daily CSV
// Define the days of the year
var startDate = ee.Date.fromYMD(2021, 1, 1);
var endDate = ee.Date.fromYMD(2023, 12, 31);
// var endDate = startDate.advance(2, 'year');

// var endDate = startDate.advance(2, 'year');
var days = ee.List.sequence(0, endDate.difference(startDate, 'day').subtract(1));

// Function to aggregate daily data
var byDay = days.map(function(day) {
  var date = startDate.advance(day, 'day');
  var dayStart = date.millis();
  
  // Total daily rainfall
  var dailyRain = mergedRainCollection
    .filter(ee.Filter.date(date, date.advance(1, 'day')));
  var totalRain = dailyRain.sum().rename('Rain');

  // Average Tmax for the day
  var dailyTmax = mergedTmaxCollection
    .filter(ee.Filter.date(date, date.advance(1, 'day')));
  var averageTmax = dailyTmax.mean().rename('Tmax');

  // Average Tmin for the day
  var dailyTmin = mergedTminCollection
    .filter(ee.Filter.date(date, date.advance(1, 'day')));
  var averageTmin = dailyTmin.mean().rename('Tmin');

  // Combine Rain, Tmax, and Tmin into one image and set the correct time property
  var image = totalRain.addBands(averageTmax).addBands(averageTmin).set({
    'system:time_start': dayStart,
    'date': date.format('yyyy-MM-dd')
  });

  return image;
});

// Create the daily image collection from the images
var dailyCol = ee.ImageCollection.fromImages(byDay);
print('Daily Collection with Rain, Tmax, Tmin:', dailyCol);





// Define your region of interest (e.g., Cuttack)
var region = ee.FeatureCollection('projects/pulakeshpradhan/assets/shp/Wb_District_new');

// Perform zonal statistics
var zonalStats = dailyCol.map(function(image) {
  var stats = image.reduceRegions({
    collection: region,
    reducer: ee.Reducer.mean(),
    scale: 1000,
  });

  // Add a date property to each feature
  return stats.map(function(feature) {
    return feature.set({
      'date': image.get('date'),
      'Rain': feature.get('Rain'),
      'Tmax': feature.get('Tmax'),
      'Tmin': feature.get('Tmin'),
      'DISTRICT': feature.get('DISTRICT')
    });
  });
}).flatten();

// Select only required properties for export
var selectedStats = zonalStats.select(['DISTRICT', 'date', 'Rain', 'Tmax', 'Tmin']);

// Print selected zonal statistics to check the output
print('Selected Zonal Statistics:', selectedStats);

// Export the zonal time series as CSV
Export.table.toDrive({
  collection: selectedStats,
  description: 'ZonalTimeSeries',
  fileFormat: 'CSV',
  selectors: ['DISTRICT', 'date', 'Rain', 'Tmax', 'Tmin']
});




//***************************************************************** Monthly CSV
// Define the time period for which you want to aggregate monthly data
var years = ee.List.sequence(2020, 2023);
var months = ee.List.sequence(1, 12);

// Function to convert month number to month name
function getMonthName(month) {
  var monthNames = ee.List(['January', 'February', 'March', 'April', 'May', 'June', 
                            'July', 'August', 'September', 'October', 'November', 'December']);
  return monthNames.get(ee.Number(month).subtract(1));  // Subtract 1 to get correct index
}

// Function to aggregate daily data into monthly totals or averages for all years
var byYearAndMonth = years.map(function(year) {
  return months.map(function(month) {
    // Total monthly rainfall
    var monthlyRain = mergedRainCollection
      .filter(ee.Filter.calendarRange(year, year, 'year'))
      .filter(ee.Filter.calendarRange(month, month, 'month'));
    var totalRain = monthlyRain.sum().rename('Rain');

    // Average Tmax for the month
    var monthlyTmax = mergedTmaxCollection
      .filter(ee.Filter.calendarRange(year, year, 'year'))
      .filter(ee.Filter.calendarRange(month, month, 'month'));
    var averageTmax = monthlyTmax.mean().rename('Tmax');

    // Average Tmin for the month
    var monthlyTmin = mergedTminCollection
      .filter(ee.Filter.calendarRange(year, year, 'year'))
      .filter(ee.Filter.calendarRange(month, month, 'month'));
    var averageTmin = monthlyTmin.mean().rename('Tmin');

    // Combine Rain, Tmax, and Tmin into one image and set the correct time property
    var image = totalRain.addBands(averageTmax).addBands(averageTmin).set({
      'system:time_start': ee.Date.fromYMD(year, month, 1).millis(),
      'year': year,
      'month': getMonthName(month) // Convert month number to name
    });

    return image;
  });
}).flatten();

// Create the monthly image collection from the images
var monthlyCol = ee.ImageCollection.fromImages(byYearAndMonth);

print('Monthly Collection with Rain, Tmax, Tmin:', monthlyCol);

// Define your region of interest (e.g., Cuttack)
// var region = ee.FeatureCollection('projects/pulakeshpradhan/assets/shp/Wb_District_new');
var region = ee.FeatureCollection("projects/pulakeshpradhan/assets/shp/Agrometeorology_Zones_WB");
// Perform zonal statistics for each month
var zonalStats = monthlyCol.map(function(image) {
  var stats = image.reduceRegions({
    collection: region,
    reducer: ee.Reducer.mean(),
    scale: 1000,
  });

  // Add date and other properties to each feature
  return stats.map(function(feature) {
    return feature.set({
      'year': image.get('year'),
      'month': image.get('month'),
      'Rain': feature.get('Rain'),
      'Tmax': feature.get('Tmax'),
      'Tmin': feature.get('Tmin'),
      // 'DISTRICT': feature.get('DISTRICT'),
      'Zone': feature.get('Zone')
    });
  });
}).flatten();

// Select only required properties for export
var selectedStats = zonalStats.select(['Zone', 'year', 'month', 'Rain', 'Tmax', 'Tmin']); // 'DISTRICT'

// Print selected zonal statistics to check the output
print('Selected Zonal Statistics:', selectedStats);

// Export the zonal time series as CSV to Google Drive
Export.table.toDrive({
  collection: selectedStats,
  description: 'MonthlyZonalTimeSeries',
  fileFormat: 'CSV',
  selectors: ['Zone', 'year', 'month', 'Rain', 'Tmax', 'Tmin'] //'DISTRICT'
});


/////////////////////////////=== Zonal Statistics === \\\\\\\\\\\\\\\\\\\\\\\\\\\\\

//====================================================================
// Load the region feature collection
var region = ee.FeatureCollection('projects/pulakeshpradhan/assets/shp/Wb_District_new');

// Print the district names to check the list
print('Districts:', region.aggregate_array('DISTRICT'));

// Define each zone by filtering the feature collection based on district names, and add a Zone property
var Northern_Hill_Zone = region.filter(ee.Filter.inList('DISTRICT', ['Kalimpong', 'Darjeeling']))
                              .map(function(feature) { return feature.set('Zone', 'Northern Hill Zone'); });
var Terai_Teesta_Alluvial_Zone = region.filter(ee.Filter.inList('DISTRICT', ['Alipurduar', 'Jalpaiguri', 'Cooch Behar', 'Uttar Dinajpur']))
                                      .map(function(feature) { return feature.set('Zone', 'Terai Teesta Alluvial Zone'); });
var Vindhyan_Alluvial_Zone = region.filter(ee.Filter.inList('DISTRICT', ['Malda', 'Dakshin Dinajpur']))
                                  .map(function(feature) { return feature.set('Zone', 'Vindhyan Alluvial Zone'); });
var Gangetic_Alluvial_Zone = region.filter(ee.Filter.inList('DISTRICT', ['Murshidabad', 'Nadia', 'Paschim Barddhaman', 'Purba Barddhaman', 'Howrah', 'Hooghli', 'North 24-pargana']))
                                  .map(function(feature) { return feature.set('Zone', 'Gangetic Alluvial Zone'); });
var Undulating_Red_Laterite_Zone = region.filter(ee.Filter.inList('DISTRICT', ['Bankura', 'Puruliya', 'Jhargram', 'Paschim Medinipur', 'Birbhum']))
                                        .map(function(feature) { return feature.set('Zone', 'Undulating Red Laterite Zone'); });
var Coastal_Saline_Zone = region.filter(ee.Filter.inList('DISTRICT', ['Purba Medinipur', 'Kolkata', 'South 24-pargana']))
                                .map(function(feature) { return feature.set('Zone', 'Coastal Saline Zone'); });

// Combine all zones into a single feature collection with the Zone property for each feature
var allZones = Northern_Hill_Zone.merge(Terai_Teesta_Alluvial_Zone)
                .merge(Vindhyan_Alluvial_Zone)
                .merge(Gangetic_Alluvial_Zone)
                .merge(Undulating_Red_Laterite_Zone)
                .merge(Coastal_Saline_Zone);

// Print the combined feature collection to verify
print('All Zones Combined with Zone Property:', allZones);

//=====================================================
// Perform zonal statistics by aggregating per zone
var zonalStats = monthlyCol.map(function(image) {
  // Compute statistics for each district, including the assigned zone
  var stats = image.reduceRegions({
    collection: allZones,
    reducer: ee.Reducer.mean(),
    scale: 1000,
  });

  // Add date and other properties to each feature
  return stats.map(function(feature) {
    return feature.set({
      'year': image.get('year'),
      'month': image.get('month'),
      'Rain': feature.get('Rain'),
      'Tmax': feature.get('Tmax'),
      'Tmin': feature.get('Tmin'),
      'DISTRICT': feature.get('DISTRICT'),
      'Zone': feature.get('Zone')  // Add the Zone property here
    });
  });
}).flatten();

// Helper function to group and calculate mean for a specific variable
function groupAndCalculateMean(variable) {
  return zonalStats.reduceColumns({
    selectors: ['Zone', 'year', 'month', variable],
    reducer: ee.Reducer.mean().group({
      groupField: 0, // Group by 'Zone'
      groupName: 'Zone',
    }).group({
      groupField: 1, // Then group by 'year'
      groupName: 'year',
    }).group({
      groupField: 2, // Then group by 'month'
      groupName: 'month'
    })
  });
}

// Compute means separately for Rain, Tmax, and Tmin
var rainGrouped = groupAndCalculateMean('Rain');
var tmaxGrouped = groupAndCalculateMean('Tmax');
var tminGrouped = groupAndCalculateMean('Tmin');

// Print grouped statistics to verify
print('Rain Grouped Statistics:', rainGrouped);
print('Tmax Grouped Statistics:', tmaxGrouped);
print('Tmin Grouped Statistics:', tminGrouped);

// Note: Further code is needed to join these grouped results into a single collection if necessary.

// Export the grouped statistics as CSV to Google Drive
Export.table.toDrive({
  collection: ee.FeatureCollection([rainGrouped, tmaxGrouped, tminGrouped]), // Adjust as needed to merge
  description: 'Zone_MonthlyZoneTimeSeries',
  fileFormat: 'CSV',
  selectors: ['Zone', 'year', 'month', 'Rain', 'Tmax', 'Tmin']
});


```
