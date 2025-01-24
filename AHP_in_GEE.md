## AHP in GEE
***

### **Step 1: Setting Up the Study Area**

The study focuses on flood susceptibility mapping in Odisha, India, using datasets from Google Earth Engine (GEE).

1.  **Define the time period of the Study**:
    
    ```javascript
    Map.setOptions("SATELLITE")

    // set start and end year
    var startyear = 2021; 
    var endyear = 2022
    // make a date object
    var startDate = ee.Date.fromYMD(startyear,1, 1);
    var endDate = ee.Date.fromYMD(endyear + 1, 1, 1);
    
    // make a list with years
    var years = ee.List.sequence(startyear, endyear);


    //Function to normalize
    function normalize(image, minValue, maxValue) {
      return image.subtract(minValue).divide(maxValue - minValue);
    }

    ```
    
    *   This specifies the boundary of Odisha, likely using a shapefile uploaded as an asset.
2.  **Set Default Viewpoint**:
    
    ```javascript
    var defaultCenterPoint = [83.95282597902448, 20.79701546664889];
    var defaultZoom = 8;
    ```
    
    *   Defines the default center point and zoom level for the map
---
Made by [Pulakesh Pradhan](https://www.linkedin.com/in/pulakeshpradhan/)

* * *

### Step 2: Importing Datasets with Min/Max Calculation

1.   **Defining the Region of Interest (Geometry)**:

*   You define your region of interest as the state of Odisha using the FAO/GAUL/2015/level1 dataset. This dataset contains the boundaries of administrative regions at the level of states and provinces. By filtering the dataset for 'Orissa' (the official name of Odisha), you specify the area to perform all further analysis.

```javascript
var geometry = ee.FeatureCollection('FAO/GAUL/2015/level1').filter(ee.Filter.eq('ADM1_NAME', 'Orissa'));
```

2.  **Elevation Data (DEM)**:

*   You load the Shuttle Radar Topography Mission (SRTM) dataset (30-meter resolution), which provides elevation data. You clip it to the defined region of Odisha and then calculate the minimum and maximum elevation values for that region.

```javascript
var dem = ee.Image('USGS/SRTMGL1_003').clip(geometry);
var demMinMax = dem.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 30, bestEffort: true});
print('DEM Min and Max:', demMinMax);
```

This helps understand the elevation range within Odisha, which could be used for further analysis of terrain or natural disaster risk assessments.

3.  **Slope Calculation**:

*   Using the DEM, you calculate the slope (steepness) of the terrain using `ee.Terrain.slope()`. This gives insight into how steep or flat the terrain is across the region.

```javascript
var slope = ee.Terrain.slope(dem);
var slopeMinMax = slope.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 30, bestEffort: true});
print('Slope Min and Max:', slopeMinMax);
```

Slope is an important factor in land use, water runoff, and agriculture planning.

4.  **Soil Properties**:

*   You use the ISRIC SoilGrids dataset to extract information on the soil content (clay, sand, and silt) at a depth of 0-5 cm. These properties influence agricultural productivity, water retention, and erosion risk.
    
*   **Clay Content**:
    

```javascript
var clayISRIC = ee.Image("projects/soilgrids-isric/clay_mean").select('clay_0-5cm_mean').clip(geometry);
var clayMinMax = clayISRIC.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 250, bestEffort: true});
print('Clay Min and Max:', clayMinMax);
```

*   **Sand Content**:

```javascript
var sandISRIC = ee.Image("projects/soilgrids-isric/sand_mean").select('sand_0-5cm_mean').clip(geometry);
var sandMinMax = sandISRIC.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 250, bestEffort: true});
print('Sand Min and Max:', sandMinMax);
```

*   **Silt Content**:

```javascript
var siltISRIC = ee.Image("projects/soilgrids-isric/silt_mean").select('silt_0-5cm_mean').clip(geometry);
var siltMinMax = siltISRIC.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 250, bestEffort: true});
print('Silt Min and Max:', siltMinMax);
```

5.  **Water Body Data**:

*   The GLCF/GLS\_WATER dataset provides information on water bodies (lakes, rivers, etc.). You clip the dataset to Odisha and calculate the minimum and maximum water presence in the region.

```javascript
var dataset = ee.ImageCollection('GLCF/GLS_WATER').filterBounds(geometry).max();
var water = dataset.clip(geometry);
var waterMinMax = water.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 30, bestEffort: true});
print('Water Min and Max:', waterMinMax);
```

6.  **Distance from Water**:

*   Using the water body data, you calculate the Euclidean distance from each pixel to the nearest water body. This can be useful for flood risk analysis or to evaluate areas near water sources.

```javascript
var distWater = water.select('water').eq(2)
  .distance({kernel: ee.Kernel.euclidean(100), skipMasked: false})
  .clip(geometry)
  .rename('distance');
var distWaterMinMax = distWater.reduceRegion({reducer: ee.Reducer.minMax(), geometry: geometry, scale: 30, bestEffort: true});
print('Distance from Water Min and Max:', distWaterMinMax);
```

7.  **Flow Accumulation**:

*   Flow accumulation, from the `WWF/HydroSHEDS/15ACC` dataset, helps assess how water collects and flows across the region. This data is used in hydrological modeling, especially to identify flood-prone areas.

```javascript
var flowaccumImage = ee.Image('WWF/HydroSHEDS/15ACC');
```

8.  **Topographic Wetness Index (TWI)**:

*   TWI is calculated using flow accumulation and slope data to identify areas with a tendency to accumulate water. TWI is important for understanding water retention and flood risks.

```javascript
var twiImage = ((flowaccumImage.add(1)).multiply(ee.Image.pixelArea())
             .divide((slope.multiply(0.0174528)).tan()).add(0.001).log()).rename('TWI');
```

9.  **Normalized Difference Vegetation Index (NDVI)**:

*   NDVI is calculated from MODIS satellite data. It gives a measure of vegetation health, which is useful for monitoring land use, agriculture, and the overall health of ecosystems.

```javascript
var ndviCollection = ee.ImageCollection('MODIS/006/MOD13A1')
  .filter(ee.Filter.date(startDate, endDate))
  .select('NDVI').mean().clip(geometry);
```

10.  **Rainfall Data**:

*   The CHIRPS dataset is used to calculate the mean annual precipitation. This is crucial for understanding the water availability, agricultural planning, and flood risk in the region.

```javascript
var chirps = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");
var annualPrecip = ee.ImageCollection.fromImages(
  years.map(function (year) {
    var annual = chirps.filter(ee.Filter.calendarRange(year, year, 'year')).sum();
    return annual.set('year', year).set('system:time_start', ee.Date.fromYMD(year, 1, 1));
  })
);
var annualMean = annualPrecip.mean().clip(geometry);
```

**Recommendation:**

It's advisable to use `bestEffort` only when necessary, such as when working with exceptionally large datasets that cannot be processed at the desired resolution. For most applications, setting `bestEffort` to `false` ensures that the analysis is performed at the intended resolution, maintaining accuracy. If you encounter memory or processing issues, consider optimizing your data or analysis approach before resorting to `bestEffort`.

For more detailed information, refer to the GEE documentation on `reduceRegion()`.
* * *
Made by [Pulakesh Pradhan](https://www.linkedin.com/in/pulakeshpradhan/)
* * *


### **Step 3: Normalizing the Data**

Once the datasets are imported, they need to be normalized to prepare them for the flood susceptibility calculation. Here's how it is done:

Normalization is performed to scale all datasets to a range of 0 to 1, making them comparable. The formula used is:  $\text{Normalized Value} = \frac{\text{Value} - \text{Min}}{\text{Max} - \text{Min}}$ 

1.  **DEM**:
    
    ```javascript
    var normDEM = normalize(dem, (-68), 1664);
    ```
    
2.  **Slope**:
    
    ```javascript
    var normSlope = normalize(slope, (0), 77.99649810791016);
    ```
    
3.  **Soil Properties**:
    
    *   **Clay**:
        
        ```javascript
        var normClay = normalize(clayISRIC, (189), 468);
        ```
        
    *   **Sand**:
        
        ```javascript
        var normSand = normalize(sandISRIC, (99), 684);
        ```
        
    *   **Silt**:
        
        ```javascript
        var normSilt = normalize(siltISRIC, (107), 578);
        ```
        
4.  **Distance from Water**:
    
    ```javascript
    var normWaterDis = normalize(distWater, (0), 141.4213562373095);
    ```
    
5.  **TWI**:
    
    ```javascript
    var normTWI = normalize(twiImage, (-6.907755278982137), 24.282822036966103);
    ```
    
6.  **NDVI**:
    
    ```javascript
    var normNDVI = normalize(ndviCollection, (-1978), 9282);
    ```
    
7.  **Rainfall**:
    
    ```javascript
    var normRain = normalize(annualMean, (1102.677119962871), 2178.0353754013777);
    ```
    
---
Made by [Pulakesh Pradhan](https://www.linkedin.com/in/pulakeshpradhan/)


* * *
### **Step 4: Assigning Weightages and Classifying Layers**

After normalizing the data, weightages are assigned to each layer based on its contribution to flood susceptibility. This step applies the **Analytic Hierarchy Process (AHP)** to combine multiple criteria into a final susceptibility score.

* * *

#### **4.1. Assign Weightages to Each Layer**

Each normalized layer is classified and multiplied by a specific weightage value. These weights represent the importance of each criterion in contributing to flood susceptibility.

* * *

1.  **DEM (Digital Elevation Model)**:  
    Higher elevation generally reduces flood risk, so lower values are given more weight.
    
    ```javascript
    var demClassified = normDEM.where(normDEM.lt(0.2), normDEM.multiply(3.50))
        .where(normDEM.gte(0.2).and(normDEM.lt(0.4)), normDEM.multiply(2.10))
        .where(normDEM.gte(0.4).and(normDEM.lt(0.6)), normDEM.multiply(0.70))
        .where(normDEM.gte(0.6), normDEM.multiply(0.70));
    ```
    
2.  **Slope**:  
    Steeper slopes generally have faster runoff, reducing flood risk. Gentle slopes are more susceptible.
    
    ```javascript
    var slopeClassified = normSlope.where(normSlope.lt(0.2), normSlope.multiply(11.50))
        .where(normSlope.gte(0.2).and(normSlope.lt(0.4)), normSlope.multiply(6.90))
        .where(normSlope.gte(0.4).and(normSlope.lt(0.6)), normSlope.multiply(3.45))
        .where(normSlope.gte(0.6), normSlope.multiply(1.15));
    ```
    
3.  **Clay Content**:  
    High clay content slows water infiltration, increasing flood susceptibility.
    
    ```javascript
    var clayClassified = normClay.where(normClay.lt(0.2), normClay.multiply(1))
        .where(normClay.gte(0.2).and(normClay.lt(0.4)), normClay.multiply(0.60))
        .where(normClay.gte(0.4).and(normClay.lt(0.6)), normClay.multiply(0.40))
        .where(normClay.gte(0.6), normClay.multiply(0.20));
    ```
    
4.  **Sand and Silt**:  
    Sand and silt content are similarly classified based on their contribution to infiltration or runoff.
    
    *   **Sand**:
        
        ```javascript
        var sandClassified = normSand.where(normSand.lt(0.2), normSand.multiply(5))
            .where(normSand.gte(0.2).and(normSand.lt(0.4)), normSand.multiply(4))
            .where(normSand.gte(0.4).and(normSand.lt(0.6)), normSand.multiply(3))
            .where(normSand.gte(0.6), normSand.multiply(2));
        ```
        
    *   **Silt**:
        
        ```javascript
        var siltClassified = normSilt.where(normSilt.lt(0.2), normSilt.multiply(5))
            .where(normSilt.gte(0.2).and(normSilt.lt(0.4)), normSilt.multiply(4))
            .where(normSilt.gte(0.4).and(normSilt.lt(0.6)), normSilt.multiply(3))
            .where(normSilt.gte(0.6), normSilt.multiply(2));
        ```
        
5.  **Distance from Water**:  
    Areas closer to water bodies are more susceptible to flooding.
    
    ```javascript
    var waterDisClassified = normWaterDis.where(normWaterDis.lt(0.2), normWaterDis.multiply(1.80))
        .where(normWaterDis.gte(0.2).and(normWaterDis.lt(0.4)), normWaterDis.multiply(0.6))
        .where(normWaterDis.gte(0.4).and(normWaterDis.lt(0.6)), normWaterDis.multiply(0.45))
        .where(normWaterDis.gte(0.6), normWaterDis.multiply(0.15));
    ```
    
6.  **Topographic Wetness Index (TWI)**:  
    Areas with high TWI values (indicative of wet conditions) are more flood-prone.
    
    ```javascript
    var twiClassified = normTWI.where(normTWI.lt(0.2), normTWI.multiply(6.40))
        .where(normTWI.gte(0.2).and(normTWI.lt(0.4)), normTWI.multiply(4.80))
        .where(normTWI.gte(0.4).and(normTWI.lt(0.6)), normTWI.multiply(3.20))
        .where(normTWI.gte(0.6), normTWI.multiply(1.60));
    ```
    
7.  **NDVI (Vegetation)**:  
    Vegetated areas are less prone to flooding.
    
    ```javascript
    var ndviClassified = normNDVI.where(normNDVI.lt(0.2), normNDVI.multiply(2.50))
        .where(normNDVI.gte(0.2).and(normNDVI.lt(0.4)), normNDVI.multiply(1.50))
        .where(normNDVI.gte(0.4).and(normNDVI.lt(0.6)), normNDVI.multiply(0.75))
        .where(normNDVI.gte(0.6), normNDVI.multiply(0.25));
    ```
    
8.  **Rainfall**:  
    Higher rainfall increases flood risk significantly.
    
    ```javascript
    var rainClassified = normRain.where(normRain.lt(0.2), normRain.multiply(3.30))
        .where(normRain.gte(0.2).and(normRain.lt(0.4)), normRain.multiply(6.60))
        .where(normRain.gte(0.4).and(normRain.lt(0.6)), normRain.multiply(9.90))
        .where(normRain.gte(0.6), normRain.multiply(13.20));
    ```
    
9.  **Land Use Land Cover (LULC)**:  
    LULC classes are weighted based on their flood susceptibility.
    
    ```javascript
        
    // Land Use Land Cover (LULC) classification
    var lulcClasses = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1').filter(ee.Filter.date(startDate, endDate)).select('label').mosaic().clip(geometry);
    
    //LULC total waight = 7.88 
    var lulcClassesWeightage = lulcClasses
      .where(lulcClasses.eq(0), lulcClasses.multiply(1.1))
      .where(lulcClasses.eq(1), lulcClasses.multiply(1.65))
      .where(lulcClasses.eq(2), lulcClasses.multiply(1.65))
      .where(lulcClasses.eq(3), lulcClasses.multiply(0.55))
      .where(lulcClasses.eq(4), lulcClasses.multiply(1.65))
      .where(lulcClasses.eq(5), lulcClasses.multiply(2.2))
      .where(lulcClasses.eq(6), lulcClasses.multiply(1.1))
      .where(lulcClasses.eq(7), lulcClasses.multiply(0.55))
      .where(lulcClasses.eq(8), lulcClasses.multiply(0.55));
    

---
Made by [Pulakesh Pradhan](https://www.linkedin.com/in/pulakeshpradhan/)



* * *

#### **4.2. Visualizing Weights**

Weights are visualized using palettes for better interpretation:

```javascript
var visSlopeWeightage = { min: 0, max: 1, palette: ["0b1eff", "4be450", "fffca4", "ffa011", "ff0000"] };
// Map.addLayer(demClassified.clip(geometry), visSlopeWeightage, 'DEM Weightage');
```

* * *
### **Step 5: Combining Layers into a Flood Susceptibility Map**

This is the critical step where all the weighted and classified layers are combined to calculate a final flood susceptibility score for the region.

* * *

#### **5.1. Combine All Weighted Layers**

The layers are added together to compute the overall flood susceptibility score. Each layer's contribution is based on its assigned weightages:

```javascript
var allLayers = demClassified
    .add(slopeClassified)
    .add(clayClassified)
    .add(waterDisClassified)
    .add(twiClassified)
    .add(ndviClassified)
    .add(rainClassified)
    .add(lulcClassesWeightage)
    .rename('ahp');
```

*   The variable `allLayers` represents the cumulative flood susceptibility score based on the **Analytic Hierarchy Process (AHP)**.

* * *

#### **5.2. Normalize the Final Susceptibility Map**

To ensure consistency and interpretability, the combined map is normalized again to a range of 0 to 1:

```javascript
var normalizedAHPCalculation = normalize(allLayers, (0.4629975633999295), 27.592958559295713);  // Min and max values
```

*   The normalized flood susceptibility map makes it easier to classify areas as low, medium, or high risk.

* * *

#### **5.3. Visualize the Final Map**

The final flood susceptibility map is visualized using a gradient palette, where:

*   **Blue** represents low susceptibility.
*   **Red** represents high susceptibility.

```javascript
Map.addLayer(
    normalizedAHPCalculation,
    {
        min: 0,
        max: 1,
        palette: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', 
                  '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
    },
    'Flood Susceptibility'
);
```

* * *

#### **5.4. Export the Results**

The final flood susceptibility map can be exported for further use:

```javascript
Export.image.toDrive({
    image: normalizedAHPCalculation,
    description: 'FloodSusceptibilityAHP',
    folder: 'GEE',
    fileNamePrefix: 'FloodSusceptibilityAHP',
    region: geometry,
    scale: 30,
    crs: 'EPSG:32644',
    maxPixels: 1e13
});
```

This will save the map as a GeoTIFF file in the specified Google Drive folder for detailed analysis or sharing.
---
Made by [Pulakesh Pradhan](https://www.linkedin.com/in/pulakeshpradhan/)


* * *
### **Step 6: Creating the User Interface (UI)**

This step involves designing an interactive user interface (UI) for visualizing and interacting with the flood susceptibility map. The UI is built using Google Earth Engine's **UI API** and includes features like layer controls, opacity sliders, and legends.

* * *

#### **6.1. Create the Main Panel**

The main panel is the primary container for the UI components, such as the title, description, and controls.

```javascript
var mainPanel = ui.Panel({
  style: {
    width: '350px',
    padding: '8px',
    border: '1px solid black',
    backgroundColor: '#f9f9f9'
  }
});
```

* * *

#### **6.2. Add Title and Metadata**

The title and metadata provide context about the project.

```javascript
var titlePanel = ui.Panel([
  ui.Label({
    value: 'Flood Susceptibility Mapping in Odisha',
    style: {
      fontWeight: 'bold',
      fontSize: '20px',
      textAlign: 'center',
      color: '#333'
    }
  }),
  ui.Label({
    value: 'By Pulakesh Pradhan*, Dr. Ranjana Bajpai**, & Sribas Patra*',
    style: {
      fontSize: '14px',
      fontWeight: 'bold',
      textAlign: 'center'
    }
  }),
  ui.Label({
    value: '*PhD Scholar | **Professor',
    style: {
      fontSize: '12px',
      fontWeight: 'bold',
      textAlign: 'center'
    }
  }),
  ui.Label({
    value: 'Dept. of Geography, Ravenshaw University',
    style: {
      fontSize: '12px',
      textAlign: 'center'
    }
  })
]);
mainPanel.add(titlePanel);
```

* * *

#### **6.3. Add Description and Instructions**

This section provides an overview of the web app and usage instructions.

```javascript
var descriptionPanel = ui.Panel([
  ui.Label({
    value: 'This web app uses multi-criteria analysis (Analytic Hierarchy Process - AHP) to visualize flood susceptibility across Odisha.',
    style: { fontSize: '13px' }
  }),
  ui.Label({
    value: 'Instructions:',
    style: { fontWeight: 'bold', fontSize: '13px' }
  }),
  ui.Label({
    value: '1. Use the layer visibility checkbox to show/hide layers.\n' +
           '2. Adjust layer opacity using the slider.',
    style: { fontSize: '12px' }
  })
]);
mainPanel.add(descriptionPanel);
```

* * *

#### **6.4. Layer Controls**

Layer controls allow users to toggle visibility and adjust opacity for different map layers.

```javascript
var layerPanel = ui.Panel({
  style: { padding: '8px' }
});

layerPanel.add(ui.Label({
  value: 'Layer Controls',
  style: { fontWeight: 'bold', fontSize: '15px' }
}));

var visibilityCheckbox = ui.Checkbox({
  label: 'Show Flood Susceptibility Layer',
  value: true,
  onChange: function(checked) {
    Map.layers().get(0).setShown(checked);
  }
});

var opacitySlider = ui.Slider({
  min: 0,
  max: 1,
  value: 1,
  step: 0.1,
  onChange: function(value) {
    Map.layers().get(0).setOpacity(value);
  },
  style: { margin: '8px 0px' }
});

layerPanel.add(visibilityCheckbox);
layerPanel.add(ui.Label({
  value: 'Layer Opacity:',
  style: { fontSize: '13px' }
}));
layerPanel.add(opacitySlider);
mainPanel.add(layerPanel);
```

* * *

#### **6.5. Add a Legend**

The legend provides visual cues to interpret the flood susceptibility map.

```javascript
var legend = ui.Panel({
  style: { padding: '8px', position: 'bottom-right' }
});

legend.add(ui.Label({
  value: 'Flood Susceptibility',
  style: { fontWeight: 'bold', fontSize: '14px' }
}));

legend.add(ui.Thumbnail({
  image: ee.Image.pixelLonLat().select(0),
  params: {
    bbox: [0, 0, 1, 0.1],
    dimensions: '200x10',
    format: 'png',
    min: 0,
    max: 1,
    palette: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', 
              '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
  },
  style: { stretch: 'horizontal', margin: '0px 8px' }
}));

legend.add(ui.Panel({
  widgets: [
    ui.Label('Low', { margin: '4px 8px' }),
    ui.Label('High', { margin: '4px 8px', textAlign: 'right' })
  ],
  layout: ui.Panel.Layout.flow('horizontal')
}));
mainPanel.add(legend);
```

* * *

#### **6.6. Set Up the Map Panel**

The map panel displays the flood susceptibility map.

```javascript
var mapPanel = ui.Map();
mapPanel.setOptions('SATELLITE');
mapPanel.setCenter(85.8959378, 20.4646314, 8);

mapPanel.addLayer(
  normalizedAHPCalculation,
  {
    min: 0,
    max: 1,
    palette: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', 
              '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
  },
  'Flood Susceptibility'
);
```

* * *

#### **6.7. Assemble the UI**

Finally, the main panel and map panel are added to the app interface.

```javascript
ui.root.clear();
ui.root.add(mainPanel);
ui.root.add(mapPanel);
```

* * *

### Final Step: Summary of the Entire Flood Susceptibility Mapping Process

Now that we’ve gone through each part in detail, let’s summarize the entire workflow of this flood susceptibility mapping project:

* * *

### **1\. Setting Up the Study Area and Datasets**

*   **Study Area**: Odisha, India, defined using a shapefile.
*   **Datasets**:
    *   DEM for elevation and slope.
    *   Soil properties (clay, sand, silt).
    *   Water bodies and proximity.
    *   Topographic Wetness Index (TWI).
    *   NDVI (vegetation health).
    *   Rainfall data (CHIRPS).
    *   Land Use Land Cover (LULC).

* * *

### **2\. Data Preprocessing**

*   Each dataset is clipped to the Odisha boundary.
*   Additional metrics are calculated:
    *   **Slope** from DEM.
    *   **TWI** using flow accumulation and slope.
    *   Annual rainfall averages from CHIRPS.

* * *

### **3\. Normalization**

*   All datasets are normalized to a scale of 0 to 1 to ensure comparability.

* * *

### **4\. Weight Assignment**

*   Weightages are assigned based on the relative contribution of each factor to flood susceptibility.
*   Classification is applied to each normalized dataset using thresholds and weight values.

* * *

### **5\. Combining Layers**

*   Weighted layers are summed using the **Analytic Hierarchy Process (AHP)** to compute the overall flood susceptibility score.
*   The combined susceptibility map is normalized for easier interpretation.

* * *

### **6\. Visualization and Export**

*   The flood susceptibility map is visualized using a color gradient (blue to red).
*   The map is exported as a GeoTIFF for external use.

* * *

### **7\. Creating an Interactive User Interface (UI)**

*   A web-based UI is built in Google Earth Engine with the following features:
    *   Title, description, and metadata display.
    *   Layer controls for toggling visibility and adjusting opacity.
    *   A legend to interpret susceptibility scores.
    *   Map display with the flood susceptibility layer.
*   The interface is assembled and made interactive.

* * *

### **Key Outputs**

1.  **Flood Susceptibility Map**:
    *   Highlights areas in Odisha at varying risk levels (low to high).
    *   Can be used for disaster preparedness and planning.
2.  **Interactive UI**:
    *   Allows users to explore data layers and customize visualizations.

* * *
# Pulakesh Pradhan


[![ResearchGate](https://img.shields.io/badge/ResearchGate-Profile-green?logo=researchgate&style=flat-square)](https://www.researchgate.net/profile/Pulakesh-Pradhan)
[![ORCID](https://img.shields.io/badge/ORCID-Profile-brightgreen?logo=orcid&style=flat-square)](https://orcid.org/0000-0003-3103-3617)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Profile-blue?logo=linkedin&style=flat-square)](https://www.linkedin.com/in/pulakeshpradhan/)
[![Twitter](https://img.shields.io/badge/Twitter-Profile-blue?logo=twitter&style=flat-square)](https://x.com/codeforgeo)
[![Medium](https://img.shields.io/badge/Medium-Profile-black?logo=medium&style=flat-square)](https://medium.com/@pulakeshpradhan)

## Contact

- **Phone**: ☏ +91-8617812861  
- **Email**: 📧 pulakesh.mid@gmail.com  
- **Address**:  
  📕 Department of Geography  
  🏫 Ravenshw University  
