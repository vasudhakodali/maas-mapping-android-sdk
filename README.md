#MaaS Mapping

[Android MaaS Mapping Documentation](http://phunware.github.io/maas-mapping-android-sdk/)

**v 1.2.0**
________________

##Overview
MaaS Mapping is an all-inclusive Android SDK for Mapping, Blue Dot, and Navigation services provided by Phunware. 
###Build requirements
* Latest MaaS Core
* OkHttp 1.2.1
* AndroidSVG 2.1.1

##Prerequisites
Install the module in the `Application` `onCreate` method before registering keys. For example:
``` Java
@Override
public void onCreate() {
    super.onCreate();
    /* Other Code */
    PwCoreSession.getInstance().installModules(PwAnalyticsModule.getInstance(), ...);
    /* Other code */
}
```

##Mapping
Mapping starts with the `PwMapView` class. It is a custom view that can be defined in an XML layout or in code. The view will not draw anything by default however it will respect the bounds given to it.

####Life cycle
Users of this class must forward some life cycle methods from the activity or fragment containing this view to the corresponding methods in this class. In particular the following methods must be forwarded:

* `onCreate(android.app.Activity, android.os.Bundle)`
* `onSaveInstanceState(android.os.Bundle)`
* `onDestroy(android.app.Activity)`
* `onLowMemory()`

####Building Data
Get building data for the map with `PwMappingModule.getBuildingDataByIdInBackground(context, long, PwOnBuildingDataListener)`. Building data contains all of the meta data for a `PwBuilding`, it's `PwFloor`s, and each LOD (level of detail) for the floors. LODs are referred to as `PwFloorResource`s. A `PwMapView` uses this data to draw a map and otherwise to function.

Once building data is obtained, use the map view's method `setMapData(pwBuilding)` to pass the data to the map. It will begin loading assets and resources immediately. 

An example of retrieving and using building data:
```Java
PwMappingModule.getInstance().getBuildingDataByIdInBackground(this, BUILDING_ID, new PwOnBuildingDataListener() {
    @Override
    public void onSuccess(PwBuilding pwBuilding) {
        if (pwBuilding != null) {
            // Building data exists!
            pwMapView.setMapData(pwBuilding);
        } else {
            // No building data found or a network error occured.
            Toast.makeText(this, "Error loading building data.", Toast.LENGTH_SHORT).show();
        }
    }
});
```

####Point of Interest Data
Get a list Points of Interest (POI) with `getPOIDataInBackground(context, long, PwOnPOIDataListener)`. Once this list is given to a `PwMapView` it can draw points on the map when relevant. This means that if a building has multiple floors then the map will only display the POIs that are on the current floor. POIs can also have a minimum zoom level specified so that they will show only once that zoom level has been reached.

Once POI data is obtained, use the map view's method `setPOIList(pois)` to pass the data to the map.

An example of retreiving and using POI data:
```Java
PwMappingModule.getInstance().getPOIDataInBackground(this, BUILDING_ID, new PwOnPOIDataListener() {
    @Override
    public void onSuccess(List<PwPoint> pois) {
        if (pois != null) {
            pwMapView.setPOIList(pois);
        } else {
            Toast.makeText(SVGCanvas.this, "Error loading poi data.", Toast.LENGTH_SHORT).show();
        }
    }
});
```

####Event Callbacks
This view provides the option to set callbacks for certain events:

* `setOnMapViewStateChangedListener(PwOnMapViewStateChangedListener)` signals when the view has changed floors or zoom levels.
* `setOnPOIClickListener(PwOnPOIClickListener)` is called any time a point of interest is clicked.
* `setOnMapLoadCompleteListener(PwOnMapLoadCompleteListener)` is called once the first image has loaded and is drawn.

####Layers
Layers can be added to the map through `addLayer(PwMapViewLayer)` and removed with `removeLayer(PwMapViewLayer)`. Predefined layers include `PwRouteLayer` and `PwBlueDotLayer`. Custom layers can be added by using extending `PwMapViewLayer`. Some lifecycle events are forwarded to layers however currently the only way to restore state is by making sure layers are added again after calling `onCreate(activity, bundle)`. In `onDestroy(activity)` layers are asked to save state and are then removed from the view.

#####PwMarkerLayer
`PwMarkerLayer` is a special layers that allows developers to add markers anywhere on the map. `PwMarker` objects can be added to or removed from this layer with `PwMarkerLayer#addMarker(pwMarker)` and `PwMarkerLayer#removeMarker(pwMarker)` respectively.
An example of creating a `PwMarker`:

```Java
// Create a marker to show on top of a POI
final PwMarker marker = new PwMarker(
        R.drawable.circle_marker,
        poi.getFloorId(),
        poi.x,
        poi.y);
// Calculate and set X and Y position offset for the marker.
final BitmapDrawable drawable = (BitmapDrawable) getResources().getDrawable(marker.getDrawableResId());
marker.xOffset = -drawable.getBitmap().getWidth() / 2f;
marker.yOffset = -drawable.getBitmap().getHeight() / 2f;
mPwMarkerLayer.addMarker(marker);
```

It is suggested to calculate a X and Y offset for a marker, in pixels. By default the offsets are 0. Because any image can be set it is up to the developer to set the appropriate offset. With no offset the top left corner of the image will be drawn at the specified X and Y.

##Blue Dot
A user's location in a venue can be retrieved with `PwBlueDotLayer`. This is a layer that should be added to a `PwMapView`. Once added the layer manages polling for a location. A Blue Dot will be displayed on the map automatically, if available, representing the end-user's location.

An example of using `PwBlueDotLayer`:
```Java
PwBlueDotLayer pwBlueDotLayer = new PwBlueDotLayer();

public void toggleBlueDot(final boolean enabled) {
    if(enabled) {
        pwMapView.addLayer(pwBlueDotLayer);
    }
    else /*disabled*/ {
        pwMapView.removeLayer(pwBlueDotLayer);
    }
}
```

Receive a callback once Blue Dot has been acquired:
```java
pwBlueDotLayer.setBlueDotListener(new PwBlueDotLayer.PwBlueDotListener() {
    @Override
    public void onBlueDotAvailable() {
        Toast.makeText(SVGCanvas.this, "Blue Dot is available!", Toast.LENGTH_SHORT).show();
    }
});
```

##Navigation
The `PwMapView` can display routes by using a `PwRouteLayer`. Routes can be a path between any two points that are defined in the MaaS Portal. A route can go from a Point of Interest (POI) to a Point of Interest, or from any way-point to a POI.

Get data for a route (if available) with `PwMappingModule#getRoute(context, long, long)` or one of it's overloaded methods. There are also methods to help get route data in the background.
There is a special method to help get a route from a point on the map; `getRouteFromLocation(context, float, float, long, long)`. This will find a route with the starting point near an `X` and `Y` coordinate. The actual starting point depends on the closest way-point that is set on the MaaS portal.

An example of retrieving and using route data:
```java
// Note that the start and end point Ids should come from a call to PwPoint#getId()
PwMappingModule.getInstance().getRouteInBackground(this, startingPointId, endingPointId, new PwRouteCallback() {
    @Override
    public void onSuccess(ArrayList<PwRoute> routes) {
        // A list of potential routes may be returned. We're only interested in the first one.
        mRouteLayer = new PwRouteLayer(routes.get(0));
        pwMapView.addLayer(mRouteLayer);
    }

    @Override
    public void onFail(int errorCode, String errorMessage) {
        Toast.makeText(this, errorCode + ": " + errorMessage, Toast.LENGTH_SHORT).show();
    }
});
```

Routes are comprised of segmen


----------


ts. In the Mapping SDK, a segment is a path between two points that are marked as an exit or a portal. Use the methods `nextRouteSegment()` and `previousRouteSegment()` on a `PwRouteLayer` to show a highlighted path along the route. The highlighted path will progress to the next segment if one exists, or the previous segment if one exists.

Attribution
-----------
MaaS Mapping uses the following 3rd party components. 

| Component     | Description   | License  |
| ------------- |:-------------:| -----:|
| [OkHttp](https://github.com/square/okhttp)      | An HTTP & SPDY client for Android and Java applications. | [Apache 2.0](https://github.com/square/okhttp/blob/master/LICENSE.txt) |
| [Picasso](https://github.com/square/picasso)      | A powerful image downloading and caching library for Android      |   [Apache 2.0](https://github.com/square/picasso/blob/master/LICENSE.txt) |
| [AndroidSVG](https://code.google.com/p/androidsvg/)      | A SVG parser and renderer for Android      |   [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0) |