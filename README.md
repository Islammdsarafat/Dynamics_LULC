# Dynamics_LULC
// Construct a collection of corresponding Dynamic World and Sentinel-2 for inspection
var START = ee.Date('2025-03-01');
var END = ee.Date('2025-03-25');

// Define the region of interest using a point
var centerPoint = ee.Geometry.Point(89.3849, 25.5876);

// Create the combined filter
var colFilter = ee.Filter.and(
    ee.Filter.bounds(centerPoint),
    ee.Filter.date(START, END)
);

// Create collections
var dwCol = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1').filter(colFilter);
var s2Col = ee.ImageCollection('COPERNICUS/S2_HARMONIZED')
    .filter(colFilter)
    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20));

// Function to link DW and S2 images by closest timestamp
function linkCollections(dwImage) {
  var dwTime = dwImage.date();
  var filteredS2 = s2Col.filter(ee.Filter.date(
    dwTime.advance(-12, 'hours'), 
    dwTime.advance(12, 'hours')
  ));
  
  var s2Image = ee.Algorithms.If({
    condition: filteredS2.size().gt(0),
    trueCase: filteredS2.sort('system:time_start').first(),
    falseCase: ee.Image([0, 0, 0]).selfMask().rename(['B4', 'B3', 'B2'])
  });
  
  return ee.Image.cat([
    ee.Image(s2Image).select(['B4', 'B3', 'B2']),
    dwImage
  ]).copyProperties(dwImage, ['system:time_start']);
}

// Link collections and get first image
var linkedCol = dwCol.map(linkCollections);
var linkedImg = ee.Image(linkedCol.first());

// Visualization parameters
var CLASS_NAMES = [
  'water', 'trees', 'grass', 'flooded_vegetation', 'crops',
  'shrub_and_scrub', 'built', 'bare', 'snow_and_ice'
];

var VIS_PALETTE = [
  '419bdf', '397d49', '88b053', '7a87c6', 'e49635', 'dfc35a', 'c4281b',
  'a59b8f', 'b39fe1'
];

// Create visualization
var dwRgb = linkedImg
  .select('label')
  .visualize({min: 0, max: 8, palette: VIS_PALETTE})
  .divide(255);

var top1Prob = linkedImg.select(CLASS_NAMES).reduce(ee.Reducer.max());
var top1ProbHillshade = ee.Terrain.hillshade(top1Prob.multiply(100)).divide(255);
var dwRgbHillshade = dwRgb.multiply(top1ProbHillshade);

// Display results
Map.setCenter(89.3849, 25.5876, 10);  // Centered on your ROI point
Map.addLayer(
  linkedImg.select(['B4', 'B3', 'B2']), 
  {min: 0, max: 3000, gamma: 1.4}, 
  'Sentinel-2 RGB'
);
Map.addLayer(
  dwRgbHillshade, 
  {min: 0, max: 0.65}, 
  'Dynamic World Classification'
);

// Add legend
var legend = ui.Panel({
  style: {
    position: 'bottom-right',
    padding: '8px 15px'
  }
});

var legendTitle = ui.Label({
  value: 'Dynamic World Classes',
  style: {
    fontWeight: 'bold',
    fontSize: '18px',
    margin: '0 0 4px 0',
    padding: '0'
  }
});

legend.add(legendTitle);

for (var i = 0; i < CLASS_NAMES.length; i++) {
  var colorBox = ui.Label({
    style: {
      backgroundColor: '#' + VIS_PALETTE[i],
      padding: '8px',
      margin: '0 0 4px 0'
    }
  });

  var description = ui.Label({
    value: CLASS_NAMES[i],
    style: {margin: '0 0 4px 6px'}
  });

  var legendRow = ui.Panel({
    widgets: [colorBox, description],
    layout: ui.Panel.Layout.Flow('horizontal')
  });

  legend.add(legendRow);
}

Map.add(legend);
// Function to export images as GeoTIFF
function exportImageToDrive(image, description, folder, region, scale) {
  Export.image.toDrive({
    image: image,
    description: description,
    folder: folder,
    scale: scale,
    region: region,
    maxPixels: 1e13,
    fileFormat: 'GeoTIFF',
    formatOptions: {
      cloudOptimized: true
    }
  });
}

// Define export parameters
var exportScale = 10; // 10m resolution
var exportFolder = 'GEE_Exports'; // Folder in your Google Drive
var exportRegion = centerPoint.buffer(5000).bounds(); // 5km buffer around point

// Prepare images for export
var s2Export = linkedImg.select(['B4', 'B3', 'B2'])
  .visualize({min: 0, max: 3000, gamma: 1.4});

var dwExport = linkedImg.select('label')
  .visualize({min: 0, max: 8, palette: VIS_PALETTE});

// Create export tasks
exportImageToDrive(s2Export, 'Sentinel2_RGB', exportFolder, exportRegion, exportScale);
exportImageToDrive(dwExport, 'DynamicWorld_Classification', exportFolder, exportRegion, exportScale);

// Print export message
print('Export tasks created for:');
print('1. Sentinel-2 RGB composite');
print('2. Dynamic World classification');
print('Check the Tasks tab to run the exports.');
