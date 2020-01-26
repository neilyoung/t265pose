
# t265pose 1.0

## An application of the [IntelRealsense T.265](https://www.intelrealsense.com/tracking-camera-t265/) camera

The following API controls an IntelRealsense T.265 tracking camera and enables geographic tracking of persons or vehicles. The API has been tested under Linux Ubuntu, macOS and Raspbian and runs in corresponding reference designs (see [Reference designs](#reference-designs))

It uses the [IntelRealSense SDK](https://github.com/IntelRealSense/librealsense), which must be installed in some form on the target system. Installation instructions can be obtained in the repository linked above. API version 1 requires at least IntelRealSense SDK `2.32.1`.

The API was implemented in **Python 3.7**. The required additional packages are listed in `requirements.txt`. Install them by issuing

`pip3 install -r requirements.txt`

The API comes as a set of encrypted Python3 scripts. The entry point is defined in `pose_main.py`.

Wheel odometry as suggested in the IntelRealSense SDK documentation is not supported in version 1.0. More on T.265 [can be found here](https://dev.intelrealsense.com/docs/intel-realsensetm-visual-slam-and-the-t265-tracking-camera).

## Version history

- 1.0 February 2020, initial version

## Start the API

The API can be launched as like any other Python3 script from the command line. The T.265 needs to be plugged.

Get help regarding command line options:

```bash
python3 -O -u pose_main.py --help
```

Standard API Port is 6000, Log level WARNING. Launch with log level set to DEBUG:

```bash
python3 -O -u pose_main.py -ll DEBUG
```

Start in background, log to file `log.txt` in the `./logs` subfolder:

```bash
nohup python3 -O -u  pose_main.py -ll DEBUG -lf log.txt &
```

## API functions

### Start tracking operation (`START_OPERATION`)

```bash
curl localhost:6000/api/v1/start -X PUT -H "Content-Type: application/json" -d@start_config.json
```

The provided dictionary `configuration` (here in form of the file `start_config.json`) is optional and contains these properties (all optional):

| Property              | Type           | Meaning                                                                                               | Default   |
|-----------------------|----------------|-------------------------------------------------------------------------------------------------------|-----------|
| geo_refs              | Array          | Array of zero or more `geo_ref` dictionaries                                                          | None      |
| loading               | Dict `enabler` | Controls loading of a map at start                                                                    | None      |
| recording             | Dict `enabler` | Controls recording of a bag file (ROSBAG)                                                             | None      |
| saving                | Dict `enabler` | Controls saving of a map                                                                              | None      |
| reset                 | Boolean        | If true, the T265 hardware is reset at startup. 2 s delay after reset.                                | false     |
| position_update_rate  | Integer        | Number of coordinates or raw poses delivered per second                                               | 30        |
| deliver_raw_poses     | Boolean        | Controls, if the app shall provide "raw" poses via the websocket interface (non-geo-referenced poses) | false     |
| world_reference_frame | String         | Determines the world's reference frame. `NED` or `ENU` allowed                                        | "ENU"     |
| strict_pose_checking  | Boolean        | see [Geo referencing](#geo_referencing)                                                               | true      |
| start_orientation     | String         | How the cam is oriented if started. Default is `forward`, USB port to the right                       | "forward" |

The dictionary `enabler` contains these properties:

| Property  | Type    | Meaning                                        | Default |
|-----------|---------|------------------------------------------------|---------|
| enabled   | Boolean | Enables or disables the functionality          | None    |
| file_name | String  | A name of a file for the desired functionality | None    |

If `loading`, `recording` or `saving` are specified, all properties of the dictionary `enabler` are mandatory. A `reset` at startup makes the T265 forget all accumulated map points in memory. Basically the camera will be setup to keep a map during several sessions, if not reset or power-cycled in between. The app delivers `WGS84` and `XYZ`, which requires the definition of at least one `geo_ref` in the `geo_refs` array. `Bag` files recorded with the `recording` configuration can be played back with either `realsense-viewer` or the `rb.py` script or other [ROS bag_tools](http://wiki.ros.org/bag_tools). The bag file recorded just contains poses, no video is contained.

If `deliver_raw_poses` is enabled, the app starts immediately to deliver raw poses via the websocket interface (good for debugging or testing). It will switch to real world coordinates later on, if at least one geo reference is specified at start and found in the map or a geo reference will be saved to the map at a certain position during the training (for details see [Geo referencing](#geo_referencing)).

By default returned geo referenced X, Y and Z coordinates are defined with respect to the `East-North-Up (ENU)` world reference frame. This can be overwritten to be `North-East-Down (NED)` world reference frame by setting the parameter `world_reference_frame` of the start configuration to `NED`.

The parameter `strict_pose_checking` defines, if a certain pose should be strictly checked to match these criterias in case, a marker should be defined (see [Geo referencing](#geo_referencing)):

- Pose confidence is 3 (confident)
- Camera's absolute pitch (rotation around camera's X) and roll (rotation around camera's Z axis) is smaller than 2 °, which minimizes the error in the Z coordinate (altitude) to be smaller then 0.5 m over 10 m distance.

This is to minimize the error in the resulting Z coordinate (altitude).

The parameter `start_orientation`, which defaults to `forward`, defines, how the camera is oriented at start. The camera can either start looking `forward`, `backward`, `upward` or `downward`. Only `forward` and only this is supported in version 1.0 of the app.

It is necessary to provide this information to the app, since the camera's reference frame is depending on it (see )

Except the `GET_STATUS` API, which is a bit more verbose, all API functions return a dictionary `result` plus either the HTTP response code 200 (success) or 400 (error), which consists a string tagged with either `success` or `error`, describing the result of the operation.

The dictionary `result` contains these properties:

| Property              | Type                         | Meaning |
|-----------------------|------------------------------|---------|
| "success" or "errror" | The success or error message | None    |

**Example payload:**

```json
{
    "geo_refs": [{
        "name": "Doorknob",
        "latitude": 12.345678,
        "longitude": 12.345678,
        "altitude": 0,
        "heading": 270
    }],
    "loading": {
        "enabled": true,
        "file_name": "./loaded_map.map"
    },
    "recording": {
        "enabled": false,
        "file_name": "./recording.bag"
    },
    "saving": {
        "enabled": true,
        "file_name": "./saved_map.map"
    },
    "reset": false,
    "position_update_rate": 30,
    "deliver_raw_poses": false,
    "world_reference_frame": "ned",
    "strict_pose_checking": true,
    "cam_direction": "forward"
 }
```

The `geo_ref` dictionary holds these properties (all mandatory):

| Property  | Type   | Meaning                                                           |
|-----------|--------|-------------------------------------------------------------------|
| name      | String | Name of the geo reference. Maximum 127 characters, UTF-8          |
| latitude  | Float  | WGS84 latitude of the geo reference                               |
| longitude | Float  | WGS84 longitude of the geo reference                              |
| altitude  | Float  | Altitude in m of the geo reference                                |
| heading   | Float  | Heading of the geo reference (North/East/South/West 0/90/180/270) |

### Stop tracking operation (`STOP_OPERATION`)

```bash
curl localhost:6000/api/v1/stop -X PUT
```

Stope the current tracking session.

### Set a marker (`SET_MARKER`)

This operation must be performed only once, but very carefully for each map. If executed successfully, this and only this operation will allow the app to provide correct world coordinates.

```bash
curl localhost:6000/api/v1/marker/:name -X PUT
```

Sets the marker with the specified name `:name` to the current map. The marker has to be defined beforehand in the array `geo_refs` of the start configuration.

### Delete a marker (`DELETE_MARKER`)

Deletes the marker with the given `:name` from the map.

```bash
curl localhost:6000/api/v1/marker/:name -X DELETE
```

Deletes the marker with the specified name `:name` from the current map. There seems to be no real use case for this function. Basically it is just in, because it is possible :)

### Save a map (`SAVE_MAP`)

The currently accumulated map of a T.265 camera is exported to disk, if the parameter `saving` of the start configuration has enabled it.

```bash
curl localhost:6000/api/v1/map -X PUT
```

### Get Status (`GET_STATUS`)

Get the status of the running app.

```bash
curl localhost:6000/api/v1/status
```

The response is variable depending on the state and may contain these properties:

| Property              | Type             | Meaning                                                                                                                  |
|-----------------------|------------------|--------------------------------------------------------------------------------------------------------------------------|
| version               | String           | Application version, semantic versioning                                                                                 |
| running               | Boolean          | Indicates if a tracking operation is running                                                                             |
| last_known_confidence | Integer          | Confidence of the last known pose, equivalent to the T.265 `tracker_confidence`: 0 - fail, 1 - low, 2 - medium, 3 - high |
| reference_point       | Dict `geo_ref`   | The currently used geo reference                                                                                         |
| last_fix              | Dict `last_fix`  | The last position fix                                                                                                    |
| last_pose             | Dict `last_pose` | The last raw camera pose, in case `deliver_raw_poses` is `true`                                                              |

The dictionaries `last_fix` and `last_pose` are Exclusive-OR.

The `last_fix` dictionary contains these properties:

| Property | Type    | Meaning                                                                         |
|----------|---------|---------------------------------------------------------------------------------|
| ft       | String  | `Format`, here `"w"` (WGS84)                                                    |
| fn       | Integer | `Frame number`, the pose's frame number as coming from the camera               |
| ts       | Float   | `Timestamp`, the pose's timestamp in milliseconds                               |
| lat      | Float   | `Latitude`, the pose's latitude                                                 |
| lng      | Float   | `Longitude`, the pose's longitude                                               |
| alt      | Float   | `Altitude`, the pose's altitude                                                 |
| x        | Float   | `X`, the pose's X w.r.t the world reference frame in m                          |
| y        | Float   | `Y`, the pose's Y w.r.t the world reference frame in m                          |
| z        | Float   | `Z`, the pose's Z w.r.t the world reference frame in m                          |
| d        | Float   | `Distance`, the pose's distance from the reference point in m                   |
| b        | Float   | `Bearing`, the pose's bearing in degrees                                        |
| p        | Float   | `Pitch`, the pose's pitch in degrees, rotation around X                         |
| r        | Float   | `Roll`, the pose's roll in degrees, rotation around Z                           |
| w        | Float   | `Yaw`, the pose's yaw in degrees, rotation around Y                             |
| c        | Integer | `Confidence`, the pose's confidence (aka T.265 `tracker_confidence`, see above) |

The `last_pose` dictionary contains these properties and is usually only used for debugging:

| Property | Type        | Meaning                                                                         |
|----------|-------------|---------------------------------------------------------------------------------|
| ft       | String      | `Format`, here `"r"` (Raw)                                                      |
| fn       | Integer     | `Frame number`, the pose's frame number as coming from the camera               |
| ts       | Float       | `Timestamp`, the pose's timestamp in milliseconds                               |
| trn      | translation | Dictionary describing the pose's translation in m                               |
| rot      | rotation    | Dictionary describing the pose's rotation as quaternion                         |
| c        | Integer     | `Confidence`, the pose's confidence (aka T.265 `tracker_confidence`, see above) |

The `translation` dictionary contains these properties:

| Property | Type  | Meaning                                    |
|----------|-------|--------------------------------------------|
| x        | Float | Translation along the camera's X axis in m |
| y        | Float | Translation along the camera's Y axis in m |
| z        | Float | Translation along the camera's Z axis in m |

The `rotation` dictionary contains these properties:

| Property | Type  | Meaning              |
|----------|-------|----------------------|
| x        | Float | Quaternion element X |
| y        | Float | Quaternion element Y |
| z        | Float | Quaternion element Z |
| w        | Float | Quaternion element W |

## Georeferencing

In accordance with the purpose of the camera, a user will be extremely interested in finding out the location of a person, a robot, a device in general in **real world coordinates** from the camera, provided that the person, robot or device carries the camera in action. This cannot be done automatically and without the help of the user, since the camera naturally has no outer world reference at all, indeed no idea of the environment in which it is used. How should it?

It is rather the case that the camera generates its own **reference frame** for the coordinates each time it is used. The camera is always in the center of that reference frame, the origin of all is right between it's two eyes. All poses and coordinates are delivered with respect to this coordinate origin (0,0,0).

There are a lot of images around, showing the camera's reference frame, here is mine. Since I'm not that good imaging 3D coordinate systems and it's possible rotations, I created my own model by sticking three coloured pencils together.

The red axis represents X, the green Y and the blue Z in a right-handed coordinate system. The images below are not 100 % correct, since - as already said - the origin of all is somewhere "in" the camera, between left and right eye and Y is aligned with gravity, but for the illustration of the various reference frames, whe have to deal with, it should be OK to get you started.

Below we have the reference frame of a forward looking camera, USB port right: Y is determined by the gravity, X points right, Z is positive towards the back.

> Note: This axis alignment must not always be of that kind. Refer [to the original Intel documentation](https://github.com/IntelRealSense/librealsense/blob/master/doc/t265.md) for what happens, if the camera is started e.g. looking downwards.

![t265_reference_frame](https://user-images.githubusercontent.com/731020/73137688-2b3f6d00-405b-11ea-95cc-45eb2a706753.jpg)

There are a lot of world reference frames, but two very popular are the `"North-East-Down" (NED)` reference frame, used in aeronautics and for sea vehicles, and the the `"East-North-Up (ENU)` reference frame, used for land vehicles (see [Wiki](https://en.wikipedia.org/wiki/Axes_conventions)).

In both reference frames the X-axis points forward. While in NED the positive Z-axis points down, it points up in ENU. Y is positive towards right in NED and towards left in ENU.

The NED world reference frame:

![ned_reference_frame](https://user-images.githubusercontent.com/731020/73137703-5924b180-405b-11ea-9143-fad3c937e4c6.jpg)

The ENU world reference frame:

![enu_reference_frame](https://user-images.githubusercontent.com/731020/73137717-87a28c80-405b-11ea-9d43-20295d3bce7c.jpg)

The app supports both depending on the start configuration parameter `world_reference_frame`.

The app also solves the problem of the variable camera coordinate frame by stitching it together with a fix world reference coordinate held outside. This is done by utilizing the `"static_node"` APIs provided by the RealSense-SDK. The idea behind is to tell the camera at a certain point in space and time to store a static node with a pre-defined name in the map "right here". The firmware of the camera then combines this name with a specific point in its derived 3D model of the outer environment and  stores it in the map. The process of managing a static marker is pictured by the APIs `SET_MARKER` and `DELETE_MARKER` described above.

If later the app tries to retrieve the same node on the same map, the SDK delivers the aligned coordinates of this static node to the app, now w.r.t. the **current** reference frame of the camera. This alone would not help, if no instance would combine these retrieved coordinates with a geo reference stored anywhere. This is what the app does for you. Optionally you are allowed to provide the WGS84 coordinates of this reference point and the app will deliver you WGS84 for your current position.

So here are the simple steps to be done, if you try to map a new area, given you are completely starting from scratch:

- Start the tracking operation by utilizing `START_OPERATIONS`
- Walk around and try to show the camera as much as you can from your environment
- At a certain point in your area choose to `SET_MARKER`. The `geo_ref` behind the `name`, you are using for that marker, must have been made known to the app at start (dictionary `geo_refs`).
- If you have optionally provided valid WGS84, make sure you are exactly there and you let the camera look exactly into the direction, which you have provided as `heading`.
- If the marker set is successful, this point is automatically the new (0,0,0) of your chosen world coordinate system. All subsequently delivered coordinates in translation and rotation are calculated w.r.t. this point.
- If you have provided valid WGS84 and correct `heading`, valid WGS84 and `bearing` will be returned to you afterwards.
- If you have enough, you `STOP_OPERATION`.

**A simple example:**

Say, you have a floor plan of your environment. You decide to make the **lower left corner of the floor plan** your world reference frame's origin. You start with a fresh map anywhere in the room, walking around a bit. Later you move continuously mapping to the point, which is (0,0,0) in your floor plan. You align the camera for your convenience so that it looks parallel to the left wall and set a marker, which you have defined to be named "marker" with lat/lng/alt/heading all set to 0. If the marker is set, the app will immediately flood you with X, Y and Z (also with lat/lng/alt), starting at (0,0,0), which is your (0,0,0) reference point in the world reference frame chosen. You move around and you will see, that X, Y and Z now reflect the distance on each axis to the marker's world coordinate in meters. Good so far and not surprising at all.

You save the map and in your next run you make the cam forget the accumulated map (be either power cycling it or do a RESET at start by setting the `reset` parameter in the start configuration). You move to a place in your room, which can be anywhere, and start a new tracking round. If (and only if) the camera again recognizes the scene and is able to recover the marked point, the app will deliver you new coordinates, which reflect your current position w.r.t. the point set as (0,0,0). This is all of the magic. You can from then on all the time determine your relative X, Y and Z or your absolute latitude, longitude and altitude based on what you have stored only once in the map. The map, btw., will enhance with every new run, so it makes sense to store it again and again, but you can of course also use a canned map, created one time on one device, on a completely different device.
