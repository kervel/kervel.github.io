---
layout: post
title:  "Publishing a Mapbox Vector Tile (MVT) tileset to the Esri Javascript API"
date:   2024-01-12 11:45:21 +0200
categories: gis
---

I needed a way to show lots of vector data on an esri javascript map. The data came from an application that has custom logic, so publishing the data is a bit more complex than just pushing the contents of a table. I had several options to publish the data:

* Rasterizing the vector data and serving it as a raster tileset. However, this would mean that i would lose the ability to interact with the data on the client side.

* Creating a web service that serves GeoJSON, and loading the GeoJSON in the map. This works, but can get slow when the data gets large: it will always load (and process) the full database.

* Using WFS. I tried creating a WFS layer because i tought it was going to solve my problems as WFS allows to query for a bounding box. However, when i tried integrating a WFS service in esri javascript, ESRI just queried for all the features in the layer, so it was effectively as fast as loading the GeoJSON.

* Using mapbox vector tiles. They are made for this purpose: the data is not rasterized so it remains vector, possible to interact with. The data is also split in tiles, so the client only needs to load the tiles that are visible on the screen. Optionally, the tiles can be cached on the client side so that they don't need to be reloaded when the user pans the map.

I decided to go with the mapbox vector tiles. Esri documentation only supports the cases where the vector tiles are served by an ESRI web service, so i had some trouble figuring out how to get ESRI to load with the mapbox vector tiles.

I used [https://github.com/submarcos/django-vectortiles](Django-vectortiles) to generate the vectortiles, which was relatively straightforward. It creates an endpoint for tiles that is of syntax `tiles/{z}/{x}/{y}`. Using django was easy as the rest of my application is also written in django, but the django part is actually rather trivial. MVT tiles can be generated for a certain zoom/x/y directly from postgis (no need for django), for instance as documented [https://gis.stackexchange.com/questions/360197/using-st-asmvt-with-st-tileenvelope-clipping-in-mapbox](here).

While this is already enough to load the vector tiles in an mapbox, or openlayers, or ..., it ain't for esri. Esri needs a metadata json that describes the zoom levels.

I found a bit of documentation:

* Esri has a web page on the vector tile services which lists an example of the json metadata [https://developers.arcgis.com/rest/services-reference/enterprise/vector-tile-service.htm](here)
* [https://gis.stackexchange.com/questions/323178/how-to-add-geoserver-vector-tiles-in-arcgis-using-arcgis-js-api](This) stack overflow post shows how to add geoserver vector tiles in arcgis, it uses a tile schema (tms), but i wanted to restrict zoom levels, so i had to generate my own metadata.

# The code

The tile endpoint that serves a tile given x/y/z is quite straightforward and just copied from the docs of django-vectortiles (i had to use 1.0.0beta3 and not the stable version as of jan 2024):

```python
from vectortiles.backends.postgis import *
from vectortiles.rest_framework.renderers import MVTRenderer

class MyViewSet(VectorLayer, viewsets.ReadOnlyModelViewSet):
    queryset = MyObject.objects.all()
    model = MyObject
    filter_backends = [filters.OrderingFilter, DjangoFilterBackend]
    filterset_class = MyObjectFilter

    id = "my_layer_id_in_mvt"
    tile_fields = ('id', 'name', 'first_name')
    queryset_limit = 1000

    ### one might be tempted to change this number to increase detail, but don't! esri only supports
    ### 512x512 vector tiles
    tile_extent = 512 


    @action(detail=False, methods=['get'], renderer_classes=(MVTRenderer, ),
            url_path='mvt/tiles/(?P<z>\d+)/(?P<x>\d+)/(?P<y>\d+)', url_name='tile')
    def tile(self, request, *args, **kwargs):
        return Response(self.get_tile(x=int(kwargs.get('x')), y=int(kwargs.get('y')), z=int(kwargs.get('z'))))
```

Now i have to create the metadata endpoint. I'm going to put it under the `mvt/` url. Some things to know:

Esri wants a list of "level of detail" (which are zoom levels it can fetch the tiles at).
Each level of detail has:

* A resolution which is the resolution in map units (meter) of a pixel in a tile. So if one pixel on the map is actually 40m in reality, the resolution would be 40.

* A scale which is how distances on the computer screen relate to distances in reality. So if one inch on the screen is 40 inch in reality, the scale would be 40. Note that normally one would write the scale as `1/40` but esri requires 40. Obviously, this depends on the DPI of the screen.

* In the standard web mercator tile scheme, at zoom level 0 the whole earth fits in one tile, and then every next zoom level halves the length and the width of a tile. The coordinate system starts at the international date line.

As you can see in the snippet below, the rest of the metadata is rather straightforward. The metadata suggests that esri supports all kind of tile styles (the tileInfo block has parameters like tile size), but this tileInfo structure is shared with other types of layers in esri that do support different tile sizes. I could not get this to work with any other size than 512.


```python
    @action(detail=False, methods=['get'], 
            url_path='mvt', url_name='mvt'
            )
    def mvt(self, request, *args, **kwargs):
        earth_circumference = 40075016.686  # in meters
        tile_size = self.tile_extent  # in pixels, only 512 supported.
        dpi = (96  / 256) * tile_size # i got this constant from the fact that 256x256 raster tiles usually correspodn with 96DPI. So i just scaled it to the tile size.
        meters_per_inch = 0.0254
        # Calculate Resolution and Scale for each zoom level / Level of detail
        lod_data = []
        for zoom_level in range(23):
            # resolution for zoom level 0 is so that the whole earth fits into one tile.
            resolution = earth_circumference / (tile_size * (2 ** zoom_level))
            # the scale is now easily computed: just take the resolution, convert it to inch and then multiply by the DPI.
            resolution_inch = resolution / meters_per_inch
            scale = resolution_inch * dpi
        
            lod_data.append({
                "level": zoom_level,
                "resolution": resolution,
                "scale": scale
            })

        # now for the rest of the json.
        json = {
            "currentVersion":11.2,
            "name":"mvt",
            "capabilities":"TilesOnly",
            "type":"indexedVector",
            "defaultStyles":"resources/styles",
            "tiles":[
                "tiles/{z}/{x}/{y}/"
                ],
            "exportTilesAllowed":False,
            "maxExportTilesCount":100000,
            "isEnabled":True,
            "exportTilesAllowed":False,
            "minScale":295828763.795777,
            "maxScale":0, #https://developers.arcgis.com/javascript/latest/visualization/high-density-data/scale-range/
            "maxZoom":19,
            "tileInfo":{
                "rows":tile_size,
                "cols":tile_size,
                "dpi":int(dpi),
                "preciseDpi":dpi,
                "format":"indexedVector",
                "origin":{"x":-20037508.342787,"y":20037508.342787}, # date line
                "spatialReference":{"wkid":102100,"latestWkid":3857}, # 102100 is just old name of 3857
                "lods":lod_data},
                "resourceInfo":{"styleVersion":8,
                                "cacheInfo":{"storageInfo":{"packetSize":128,"storageFormat":"compactV2"}}}}
        return Response(json)
```

Some of the keys in the json i just copied from the example (for instance resourceInfo). I didn't fully investigate what they mean.

Also note that the metadata url gives the url of the real tile endpoint in a relative way. If you're going to change url paths, be sure to change that too. 

I also still need a default style for my map (it was refered to in the above json). Note that this style can be overridden from the javascript API. The style below is valid for polygons. If you want to publish points, you will need something else (but this part is documented so i'm not going to go in detail here)

```python
    @action(detail=False, methods=['get'], 
            url_path='mvt/resources/styles/root.json', url_name='mvt_style'
            )
    def mvt_style(self, request, *args, **kwargs):
        return Response({
            "version" : 8,
            "sources" : {
                "esri" : {
                    "type" : "vector",
                    "url" : "../../"
                }
            },
            "layers" : [{
                    "id" : "Persons",
                    "type" : "fill",
                    "source" : "esri",
                    "source-layer" : "my_layer_id_in_mvt",
                    "minzoom" : 0,
                    "layout" : {},
                    "paint" : {
                        "fill-color" : "#f7f6d5"
                    }
                }
                ]
            })
```

Now the only thing that's left to do is to add the mvt tileset to the map. This can be done by pointing either to the metadata url (which then needs to refer to a default style) or to the style url (which then needs to refer to a source). I use the former: 

```typescript
  async initMVT() {
    let [VectorTileLayer] = await loadModules(["esri/layers/VectorTileLayer"]);
    let layer  = new VectorTileLayer({
      url: "http://127.0.0.1:8000/api/person/mvt/", // obviously change this
      title: "Person MVT",
      visible: false
    });
    this._map.add(layer);
  }
```