Map.addLayer(mun_basin)
var stations = ee.FeatureCollection(table);

// Display on map
Map.addLayer(stations, {color: 'red'}, 'Stations');
Map.centerObject(stations, 8);

//loading the datasets
var chirps = ee.ImageCollection("UCSB-CHG/CHIRPS/DAILY")
                .filterDate('1995-01-01', '2024-12-31')
                .map(function(img){
                  return img.clip(mun_basin).rename('precipitation');
                });
              

// 2. Sample all stations
var ts_all = chirps.map(function(img){
  return img.sampleRegions({
    collection: stations,
    scale: 5000,
    geometries: false
  }).map(function(f){
    return f.set('date', img.date().format('YYYY-MM-dd'));
  });
}).flatten();

// 3. Convert to dictionary for wide format
var dates = ts_all.aggregate_array('date').distinct().sort();
var stationIds = stations.aggregate_array('StationID');

var wide = ee.FeatureCollection(dates.map(function(d){
  var date = d;
  var row = stationIds.map(function(id){
    var val = ts_all.filter(ee.Filter.and(
      ee.Filter.eq('StationID', id),
      ee.Filter.eq('date', date)
    )).first().get('precipitation');
    return ee.String(id).cat(':').cat(ee.String(val));
  });
  return ee.Feature(null, ee.Dictionary.fromLists(stationIds, row));
}));

// 4. Export to CSV
Export.table.toDrive({
  collection: wide,
  description: 'CHIRPS_WideFormat',
  fileFormat: 'CSV'
});
