# mbtiles

Node.js utilities and [tilelive](https://github.com/mapbox/tilelive.js) integration for the [MBTiles](http://mbtiles.org) format.

[![Build Status](https://travis-ci.org/mapbox/node-mbtiles.svg?branch=master)](https://travis-ci.org/mapbox/node-mbtiles)
[![Build status](https://ci.appveyor.com/api/projects/status/04wbok5rs3eroffe)](https://ci.appveyor.com/project/Mapbox/node-mbtiles)

# Installation

```
npm install node-mbtiles
```

```javascript
var MBTiles = require('node-mbtiles');
```

# API

### Constructor

All MBTiles instances need to be constructed before any of the methods become available. *NOTE: All methods described below assume you've taken this step.*

```javascript
let mbtiles=await new MBTiles('./path/to/file.mbtiles?mode={ro, rw, rwc}');
```

The `mode` query parameter is a opening flag of mbtiles. It is optional, default as `rwc`. Available flags are:

- `ro`: readonly mode, will throw error if the mbtiles is not existed.
- `rw`: read and write mode, will throw error if the mbtiles is not existed.
- `rwc`: read, write and create mode, will create a new mbtiles if the mbtiles is not existed.

### Reading

**`async getTile(z, x, y)`**

Get an individual tile from the MBTiles table. This can be a raster or gzipped vector tile. Also returns headers that are important for serving over HTTP.
原来@mapbox的callback(null,tile,headers)
更改为 callback(null,{
  tile:tile,
  headers:headers
});

```javascript
let tileInfo=await mbtiles.getTile(z, x, y);
```

**`async getInfo()`**

Get info of an MBTiles file, which is stored in the `metadata` table. Includes information like zoom levels, bounds, vector_layers, that were created during generation. This performs fallback queries if certain keys like `bounds`, `minzoom`, or `maxzoom` have not been provided.

```javascript
let info=await mbtiles.getInfo();
```

**`async getGrid(z, x, y)`**

Get a [UTFGrid](https://github.com/mapbox/utfgrid-spec) tile from the MBTiles table.
原来@mapbox的callback(null,grid,headers)
更改为 callback(null,{
  grid:grid,
  headers:headers
});
```javascript
let gridInfo=await mbtiles.getGrid(z, x, y);
```

### Writing

**`async startWriting`** AND **`async stopWriting`**

In order to write a new (or currently existing) MBTiles file you need to "start" and "stop" writing. First, [construct](#constructor) the MBTiles object.

```javascript
await mbtiles.startWriting();
await mbtiles.stopWriting();
```

**`async putTile(z, x, y, buffer)`**

Add a new tile buffer to a specific ZXY. This can be a raster tile or a _gzipped_ vector tile (we suggest using `require('zlib')` to gzip your tiles).

```javascript
var zlib = require('zlib');

zlib.gzip(fs.readFileSync('./path/to/file.mvt'), async function(err, buffer) {
  await mbtiles.putTile(0, 0, 0, buffer);
});
```

**`async putInfo(data)`**

Put an information object into the metadata table. Any nested JSON will be stringified and stored in the "json" row of the metadata table. This will replace any matching key/value fields in the table.

```javascript
var exampleInfo = {
  "name": "hello-world",
  "description":"the world in vector tiles",
  "format":"pbf",
  "version": 2,
  "minzoom": 0,
  "maxzoom": 4,
  "center": "0,0,1",
  "bounds": "-180.000000,-85.051129,180.000000,85.051129",
  "type": "overlay",
  "json": `{"vector_layers": [ { "id": "${layername}", "description": "", "minzoom": 0, "maxzoom": 4, "fields": {} } ] }`
};

await mbtiles.putInfo(exampleInfo);
```

**`async putGrid(z, x, y, grid)`**

Inserts a [UTFGrid](https://github.com/mapbox/utfgrid-spec) tile into the MBTiles store. Grids are in JSON format.

```javascript
var fs = require('fs');
var grid = JSON.parse(fs.readFileSync('./path/to/grid.json', 'utf8'));
await mbtiles.putGrid(0, 0, 0, grid);
```

## Hook up to tilelive

When working at scale, node-mbtiles is meant to be used within a [Tilelive](https://github.com/mapbox/tilelive) ecosystem. For example, you could set up an MBTiles file as a "source" and an S3 destination as a "sink" (using tilelive-s3). Assuming you have a system set up with an `mbtiles://` protocol that points to a specific file and authorized to write to the s3 bucket:

```javascript
var tilelive = require('@mapbox/tilelive');
var MBTiles = require('@mapbox/mbtiles');
var s3 = require('@mapbox/tilelive-s3');

s3.registerProtocols(tilelive);
MBTiles.registerProtocols(tilelive);

var sourceUri = 'mbtiles:///User/hello/path/to/file.mbtiles';
var sinkUri = 's3://my-bucket/tiles/{z}/{x}/{y}';

// load the mbtiles source
tilelive.load(sourceUri, function(err, src) {
  // load the s3 sink
  tilelive.load(sinkUri, function(err, dest) {

    var options = {}; // prepare options for tilelive copy
    options.listScheme = src.createZXYStream(); // create ZXY stream from mbtiles

    // now copy all tiles to the destination
    tilelive.copy(src, dst, options, function(err) {
      console.log('tiles are now on s3!');
    });
  });
});
```

# Test

```
npm test
```
