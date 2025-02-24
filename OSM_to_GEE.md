
```python
# !pip install osmnx

# Import required libraries
import os
import ee
import geemap
import osmnx as ox
import shutil # Import the shutil module

# Authenticate and initialize Earth Engine
ee.Authenticate()
ee.Initialize(project="geepulakesh")

# Authenticate and initialize Earth Engine
Map =geemap.Map()
# Map
# Define the place
place = 'Darjeeling, India'

# Define the tags to search for parks
tags = {'leisure': 'park', 'landuse':'park'}
# tags = {
#         'leisure': ['park', 'recreation_ground', 'pitch'],
#         'landuse': 'park',
#         'amenity': ['hospital', 'pharmacy', 'school', 'university', 'police', 'clinic', 'ambulance_station', 'fire_station'],
#         'highway': 'street_lamp',
#         'shop': True,
#         'office': True,
#         'industrial': True,
#         'public_transport': True
#     }

print(f"Searching for parks in {place} using tags: {tags}")


# Get city boundary and parks
boundary_gdf = ox.geocode_to_gdf(place)
parks_gdf = ox.features_from_place(place, tags=tags)

# Convert the GeoDataFrame (parks) to an Earth Engine FeatureCollection
eeParks = geemap.gdf_to_ee(parks_gdf)
eeStudyArea = geemap.gdf_to_ee(boundary_gdf)


# Add parks FeatureCollection to the map
Map.addLayer(eeStudyArea, {}, "Mumbai")
Map.addLayer(eeParks, {}, "Parks in Mumbai")
Map.centerObject(eeStudyArea, 12)

# Display the map
Map




# Ensure correct geometry type (convert to MultiPolygon if needed)
parks_gdf["geometry"] = parks_gdf["geometry"].apply(lambda geom: geom if geom.geom_type == "MultiPolygon" else geom.buffer(0))

# Create a folder to store shapefiles
shp_folder = "shape"
os.makedirs(shp_folder, exist_ok=True)

# Save city boundary as shapefile
# boundary_gdf.to_file(f"{shp_folder}/mumbai_boundary.shp", driver="ESRI Shapefile")

# Save parks data as shapefile
parks_gdf.to_file(f"{shp_folder}/mumbai_parks.shp", driver="ESRI Shapefile")

# Create a ZIP file containing all shapefiles
zip_filename = "mumbai_shapefiles.zip"
shutil.make_archive("mumbai_shapefiles", 'zip', shp_folder)

print(f"Shapefiles saved and zipped as {zip_filename}")
```
