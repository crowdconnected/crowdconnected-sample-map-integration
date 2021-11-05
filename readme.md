# Sample Map Integration
Overview of how to create a Leaflet map capable of displaying. The process for this could be used to create a Mapbox Map.

## Pre-requisites
- Android App with Crowd Connected SDK Implemented - [Android SDK Documentation](https://support.crowdconnected.com/hc/en-us/articles/4403605601937-Integrating-the-Android-SDK-Background-Data-Collection), [iOS SDK Documentation](https://support.crowdconnected.com/hc/en-us/articles/4403954756497-Integrating-the-iOS-SDK-Background-Data-Collection)
- Understanding of Leaflet Maps

---

## An overview of the implementation
1. In the Android App, call getSurfaces() method to load the surfaces and pass these into the webview
2. Create a map with layers for each of the surfaces
3. Register for position callbacks in the Android App, and pass these into the webview
4. Render a blue dot on the map at the users location, and if necessary switch the layer that is being displayed

---

## What is a surface?
A surface is made up of the following information that is required by Leaflet
```
{
    "appKey": string,           // Client identifier
    "surfaceID": string,        // Unique identifier for the surface
    "name": string,             // A name chosen by you to identify the surface
    "floor": number,            // Numeric floor number so floors can be correctly ordered
    "versions": number,         // A count of how many versions of the surface that have been uploaded
    "zoom": number,             // The number of zoom levels that have been generated
    "max_x": number,            // The most easterly point on the map
    "min_y": number,            // The most southerly point on the map
    "xScalingFactor": number,   // Conversion factor to go from metres into Leaflet Simple CRS
    "yScalingFactor": number,   // Conversion factor to go from metres into Leaflet Simple CRS
}
```
## Where are our tiles hosted?
Our tiles public tiles can be loaded using the following URL format:

```
https://public-map-data.crowdconnected.com/{appKey}/{surfaceID}/{v}/{z}/{x}/{y}.png
```

**Where:**
- *appKey*: The appKey belonging to the surface
- *surfaceID*: The surfaceID for the surface
- *v*: The version number of the surface to load
- *z*:  Zoom Level
- *x*:  The X Coordinate
- *y*:  The Y Coordinate

&nbsp;

# Creating the webview
## 1. Create a function to accept the surfaces
### Javascript Implementation
Below is an example function that takes the surfaces, sorts them, creates a map and calculates the bounds. It then iterates through the surfaces, creating a leaflet layer for each. These are stored in a map that is keyed by the surface name, and is then used to create the layer controls. Following that a layergroup is created to contain the marker for the bluedot.
```
initMap = function (surfacesToInit) {
    // Sort the surfaces based on their floor number
    surfacesToInit = surfacesToInit.sort((a, b) => a.floor - b.floor);

    // Create the Leaflet map, note the CRS must be Simple
    map = L.map('map', {
        crs: L.CRS.Simple
    });

    // Calculate the bounds and zoom for the map
    calculateBoundsZoom();

    // Define a url with placeholders for the various parameters
    const url = 'https://public-map-data.crowdconnected.com/{appKey}/{surfaceID}/{v}/{z}/{x}/{y}.png';

    // Iterate through the surfaces and now add them to the map, this cannot be done before map and bound creation
    for (surface of surfacesToInit) {
        const tileLayer = L.tileLayer(url, {
            appKey: surface.appKey,
            surfaceID: surface.surfaceID,
            v: surface.versions,
            tileSize: 512,                      // We generate tiles that are 512px
            minZoom: 0,
            maxZoom: surface.zoom + 4,          // This allows for zooming beyond the native tile zoom level
            maxNativeZoom: surface.zoom,        // This is how many tile zoom levels have been generated
            bounds: calculateBounds(surface.max_x, surface.min_y, surface.zoom)
        });

        mapLayers[surface.name] = tileLayer;
    }

    // Add a layer control to do simple floor selection
    L.control.layers(mapLayers).addTo(map);

    // Create a layer for holding the blue dot
    blueDotLayer = new L.LayerGroup();
    blueDotLayer.addTo(map);
```
### Android Implementation
This should be called in the Android App after the webview has initialised, below is an example:
```
webView.setWebViewClient(new WebViewClient() {
    @Override
    public void onPageFinished(WebView view, String url) {
        String script = "initMap([" +
            "{\"surfaceID\": \"d4930d9e-0451-4e8d-afd6-81bcdea1582a\",\"appKey\": \"MTzvMnNj\",\"yScalingFactor\": 0.16207842582607765,\"zoom\": 1,\"min_y\": -1000,\"name\": \"Floor 2\",\"max_x\": 894,\"versions\": 0,\"xScalingFactor\": 0.16207842582607765, \"floor\": 2}," +
            "{\"surfaceID\": \"d0b1f8b3-b76c-4aff-b628-1a54f478b918\",\"appKey\": \"MTzvMnNj\",\"yScalingFactor\": 0.16232449599785742,\"zoom\": 1,\"min_y\": -1000,\"name\": \"Floor 1\",\"max_x\": 892,\"versions\": 0,\"xScalingFactor\": 0.16232449599785742, \"floor\": 1}," +
            "{\"surfaceID\": \"fc0a8a1c-6fae-463f-8a73-76750a765da6\",\"appKey\": \"MTzvMnNj\",\"yScalingFactor\": 0.16205118442879768,\"zoom\": 1,\"min_y\": -998,\"name\": \"Ground Floor\",\"max_x\": 892,\"versions\": 0,\"xScalingFactor\": 0.16205118442879768, \"floor\": 0}," +
            "{\"surfaceID\": \"2fe03acd-c596-49f1-86c5-7740dbe1c33a\",\"appKey\": \"MTzvMnNj\",\"yScalingFactor\": 0.16224816859068747,\"zoom\": 1,\"min_y\": -999,\"name\": \"Floor 3\",\"max_x\": 894,\"versions\": 0,\"xScalingFactor\": 0.16224816859068747, \"floor\": 3}" +
            "]);";

        view.loadUrl("javascript:" + script);
        loaded[0] = true;
    }
});
```

## 2. Create the functions to calculate the bounds
The overall bounds need to be calculated for the map, this should cover the full extent of all surfaces, so the maximum of the max_x and minimum of the min_y should be found, as well as the highest zoom level.
```
calculateBoundsZoom = function () {    
    let max_x = Math.max(...surfaces.map((s) => s.max_x));
    let min_y = Math.min(...surfaces.map((s) => s.min_y));
    let maxZoom = Math.max(...surfaces.map((s) => s.zoom));

    const bounds = calculateBounds(max_x, min_y, maxZoom);

    map.fitBounds(bounds);
}
```
In leaflet the values that we have are in metres and they need to be projected onto the L.Simple.CRS coordinate system before they can be used.
```
calculateBounds = function (max_x, min_y, zoom) {
    const southWest = map.unproject([0, Math.abs(min_y)], zoom);
    const northEast = map.unproject([max_x, 0], zoom);
    return L.latLngBounds(southWest, northEast);
}
```

## 3. Create a function to accept a position and add it to the map
### Javascript Implementation
```
 addPositionWithSurface = function (surfaceID, timestamp, y, x) {
    const surface = surfaces.find((s) => s.surfaceID === surfaceID);

    if (surface && (jumpToLocation || currentSurfaceID === surfaceID)) {
        // Change the surface that is currently being displayed if it is not the existing surface
        if (currentSurfaceID !== surfaceID) {
            currentSurfaceID = surfaceID;

            Object.keys(mapLayers).forEach((key) => {
                mapLayers[key].removeFrom(map);
            });

            mapLayers[surface.name].addTo(map)
        }

        // The positions that come from the Library are in metres, we need to convert them the map scale
        y = y / (surface.yScalingFactor || 1);
        x = x / (surface.xScalingFactor || 1);
        const latLng = new L.LatLng(y, x);

        // Only create a new marker if one doesn't already exist, otherwise just update its latLng
        if (blueDotLayer && !blueDot) {
            blueDot = L.marker(latLng, {
                icon: L.divIcon({
                    className: 'cc-marker-container',
                    html: '<div class="cc-marker-circle"></div>'
                })
            }).addTo(blueDotLayer);
        } else if (blueDot) {
            blueDot.setLatLng(latLng);
        }
    }
}
```
### Android Implementation
In the Android App register for location callback events and then call the JavaScript method:
```
CrowdConnected.getInstance().registerPositionCallback(location -> webView.post(() -> {
    if (loaded[0]) {
        String javascriptMethodCall = "javascript:addPositionWithSurface('" + location.getSurfaceId() + "','"  + location.getTimestamp() + "','" + location.getY() + "','" + location.getX() + "')";
        webView.loadUrl(javascriptMethodCall);
    } 
}));
```

## Further enhancements
Currently any positions that are added to the map will cause the basemap to change, which prevents the user from being able to view a floor of their choosing.

To stop this you can add leaflet easy button and font awesome to add a button to prevent the map from changing
&nbsp;
Add the following to the header section of the HTML.
```
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/leaflet-easybutton@2/src/easy-button.css">
<script src="https://cdn.jsdelivr.net/npm/leaflet-easybutton@2/src/easy-button.js"></script>
```

Add this to the JavaScript
```
    // Listen for the baselayer changing and if there is a bluedot then remove it, otherwise the blue dot will be displayed on the wrong floors
    map.on('baselayerchange', function (change) {
        if (blueDot) {
            blueDot.removeFrom(blueDotLayer);
            blueDot = null;
        }
        currentSurfaceID = change.layer.options.surfaceID;
    });
   

    // Add a button to disable the automatic changing of the level if you wish to just navigate the map
    L.easyButton({
        states: [{
            stateName: 'location-enabled',
            icon: 'fas fa-compass',
            onClick: function (btn) {
                jumpToLocation = false;
                btn.state('location-disabled');
            }
        }, {
            stateName: 'location-disabled',
            icon: 'far fa-compass',
            onClick: function (btn) {
                jumpToLocation = true;
                btn.state('location-enabled');
            }
        }]
    }).addTo(map);
}
```

## The final code
```
<!DOCTYPE html>
<html>

<head>

    <title>Crowd Connected Sample Map</title>

    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css"
        integrity="sha512-xodZBNTC5n17Xt2atTPuE1HxjVMSvLVW9ocqUKLsCC5CXdbqCmblAshOMAS6/keqq/sMZMZ19scR4PsZChSR7A=="
        crossorigin="" />
    <script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"
        integrity="sha512-XQoYMqMTK8LvdxXYG3nZ448hOEQiglfqkJs1NOQV44cWnUrBc8PkAOcXy20w0vlaXaVUearIOBhiXZ5V3ynxwA=="
        crossorigin=""></script>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/leaflet-easybutton@2/src/easy-button.css">
    <script src="https://cdn.jsdelivr.net/npm/leaflet-easybutton@2/src/easy-button.js"></script>

    <style>
        html,
        body {
            height: 100%;
            margin: 0;
        }

        #map {
            width: 100%;
            height: 100%;
        }

        /* You can customise the marker to be however you like */
        .cc-marker-circle {
            width: 8px;
            height: 8px;
            border: solid 2px;
            border-radius: 50%;
            border-color: white;
            background: #3288ff;
        }
    </style>

</head>

<body>
    <div id='map'></div>

    <script>
        let map;
        let surfaces;
        let currentSurfaceID;
        let mapLayers = {};

        let blueDotLayer;
        let blueDot;
        let jumpToLocation = true;

        initMap = function (surfacesToInit) {
            // Sort the surfaces based on their floor number
            surfacesToInit = surfacesToInit.sort((a, b) => a.floor - b.floor);

            // Surfaces need to be global so when positions are added we can correctly plot them and change the map view
            surfaces = surfacesToInit;

            // Create the Leaflet map, note the CRS must be Simple
            map = L.map('map', {
                crs: L.CRS.Simple
            });

            // Calculate the bounds and zoom for the map
            calculateBoundsZoom();

            // This is where our public tiles are hosted
            const url = 'https://public-map-data.crowdconnected.com/{appKey}/{surfaceID}/{v}/{z}/{x}/{y}.png';

            // Iterate through the surfaces and now add them to the map, this cannot be done before map and bound creation
            for (surface of surfacesToInit) {
                const tileLayer = L.tileLayer(url, {
                    appKey: surface.appKey,
                    surfaceID: surface.surfaceID,
                    v: surface.versions,
                    tileSize: 512,                      // We generate tiles that are 512px
                    minZoom: 0,
                    maxZoom: surface.zoom + 4,          // This allows for zooming beyond the native tile zoom level
                    maxNativeZoom: surface.zoom,        // This is how many tile zoom levels have been generated
                    bounds: calculateBounds(surface.max_x, surface.min_y, surface.zoom)
                });

                mapLayers[surface.name] = tileLayer;
            }

            // Add a layer control to do simple floor selection
            L.control.layers(mapLayers).addTo(map);

            // Create a layer for holding the blue dot
            blueDotLayer = new L.LayerGroup();
            blueDotLayer.addTo(map);

            // Listen for the baselayer changing so the blue dot is not rendered on the wrong layer
            map.on('baselayerchange', function (change) {
                if (blueDot) {
                    blueDot.removeFrom(blueDotLayer);
                    blueDot = null;
                }
                currentSurfaceID = change.layer.options.surfaceID;
            });

            // Add a button to disable the automatic changing of the level if you wish to just navigate the map
            L.easyButton({
                states: [{
                    stateName: 'location-enabled',
                    icon: 'fas fa-compass',
                    onClick: function (btn) {
                        jumpToLocation = false;
                        btn.state('location-disabled');
                    }
                }, {
                    stateName: 'location-disabled',
                    icon: 'far fa-compass',
                    onClick: function (btn) {
                        jumpToLocation = true;
                        btn.state('location-enabled');
                    }
                }]
            }).addTo(map);
        }

        calculateBoundsZoom = function () {
            // Leaflet needs the bounds for the map which is from the southWest corner to the northEast
            // The north west corner is always 0,0, so we just need to find the most southerly point (min_y) and the most easterly point (max_x)
            // Leaflet has it's Y axis inverted e.g. 0 to -100 so the further south you go, the more negative it becomes which is why it was called min_y and not max_y
            // The zoom is also needed for Leaflet to correctly be able to render the tiles.
            let max_x = Math.max(...surfaces.map((s) => s.max_x));
            let min_y = Math.min(...surfaces.map((s) => s.min_y));
            let maxZoom = Math.max(...surfaces.map((s) => s.zoom));

            const bounds = calculateBounds(max_x, min_y, maxZoom);

            // Make the map fit the bounds
            map.fitBounds(bounds);
        }

        calculateBounds = function (max_x, min_y, zoom) {
            // Calculate the bounds, and change the projection of the numbers based on the maxZoom
            const southWest = map.unproject([0, Math.abs(min_y)], zoom);
            const northEast = map.unproject([max_x, 0], zoom);
            return L.latLngBounds(southWest, northEast);
        }

        addPositionWithSurface = function (surfaceID, timestamp, y, x) {
            const surface = surfaces.find((s) => s.surfaceID === surfaceID);

            if (surface && (jumpToLocation || currentSurfaceID === surfaceID)) {
                // Change the surface that is currently being displayed if it is not the existing surface
                if (currentSurfaceID !== surfaceID) {
                    currentSurfaceID = surfaceID;

                    Object.keys(mapLayers).forEach((key) => {
                        mapLayers[key].removeFrom(map);
                    });

                    mapLayers[surface.name].addTo(map)
                }

                // The positions that come from the Library are in metres, we need to convert them the map scale
                y = y / (surface.yScalingFactor || 1);
                x = x / (surface.xScalingFactor || 1);
                const latLng = new L.LatLng(y, x);

                // Only create a new marker if one doesn't already exist, otherwise just update its latLng
                if (blueDotLayer && !blueDot) {
                    blueDot = L.marker(latLng, {
                        icon: L.divIcon({
                            className: 'cc-marker-container',
                            html: '<div class="cc-marker-circle"></div>'
                        })
                    }).addTo(blueDotLayer);
                } else if (blueDot) {
                    blueDot.setLatLng(latLng);
                }
            }
        }
    </script>
</body>

</html>
```

# Using our hosted webview
We have a hosted version of the webview available at
```
http://colocator-v3-sample-map.s3-website-us-east-1.amazonaws.com/
```
If you opt to use our webview you must call the following Javascript Methods:
&nbsp; 
&nbsp; 
Initialising the map
```
initMap([{
    "appKey": "MTzvMnNj",
    "surfaceID": "d0b1f8b3-b76c-4aff-b628-1a54f478b918",
    "name": "Floor 1",
    "floor": 1,
    "versions": 0,
    "zoom": 1,
    "max_x": 892,
    "min_y": -1000,
    "xScalingFactor": 0.16232449599785742,
    "yScalingFactor": 0.16232449599785742
}])
```
Adding a blue dot for the users current location
```
addPositionWithSurface('d0b1f8b3-b76c-4aff-b628-1a54f478b918', 1636026270871, -20, 10)
```