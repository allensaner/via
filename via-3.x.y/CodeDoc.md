# Code Documentation for VIA Version 3.x.y
The full VIA3 software application fits in a single HTML file of size less than 
300 Kilobyte containing nearly 8000 lines of carefully amalgamated HTML, CSS 
and Javascript code. This single HTML file can run as an offline application 
in most modern web browsers to provide the functionality of a fully featured 
manual annotation software application for image, audio and video. In this 
report we explain different aspects of the software code powering the VIA3 
software with the aim of nurturing future code contributors to this open source 
project.

In this report, we will explain the main concepts of VIA3 using [_via_video_annotator.html](via-3.x.y/src/html/_via_video_annotator.html)
as an example. To help readers guide through the code, we have removed all the details
and included a simplified view of the show a simplfied view of [_via_video_annotator.html](via-3.x.y/src/html/_via_video_annotator.html) 
file below:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>VIA Video Annotator</title>
  <style>
  [CSS style definitions]
  </style>
</head>

<body onresize="via._hook_on_browser_resize()">
  <svg style="display:none;" version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
    [Definition of SVG icons]
  </svg>
  <div id="via_info_page_container">
    [Definition of information pages (e.g. About, Help, Keyboard Shortcuts, etc)]
  </div>
  <div id="via_start_info_content" class="hide">
    [Definition of usage information shown when VIA start up]
  </div>

  <div class="via_container" id="via_container" tabindex="0">
    [VIA container which is dynamically populated by VIA Javascript codebase]
  </div>

  <script src="../js/_via_util.js"></script>
  <script src="../js/_via_const.js"></script>
  <script src="../js/_via_config.js"></script>
  <script src="../js/_via_event.js"></script>
  <script src="../js/_via_control_panel.js"></script>
  <script src="../js/_via_metadata.js"></script>
  <script src="../js/_via_file.js"></script>
  <script src="../js/_via_attribute.js"></script>
  <script src="../js/_via_view.js"></script>
  <script src="../js/_via_data.js"></script>
  <script src="../js/_via_video_thumbnail.js"></script>
  <script src="../js/_via_file_annotator.js"></script>
  <script src="../js/_via_temporal_segmenter.js"></script>
  <script src="../js/_via_view_annotator.js"></script>
  <script src="../js/_via_view_manager.js"></script>
  <script src="../js/_via_editor.js"></script>
  <script src="../js/_via_debug_project.js"></script>
  <script src="../js/_via.js"></script>

  <script>
    var via_container = document.getElementById('via_container');
    var via = new _via(via_container);
  </script>
</body>
</html>+
```
To understand the HTML and CSS portions of the above file, follow this 
[excellent tutorial](https://developer.mozilla.org/en-US/docs/Web/HTML) developed 
by Mozilla.


## Core Data Structure : _via_data.js
> "Bad programmers worry about the code. Good programmers worry about
> data structures and their relationships."
> [Linus Torvalds](https://lwn.net/Articles/193245/)

An instance of [_via_data.js](via-3.x.y/src/js/_via_data.js) function defines and 
manages the core data of the VIA3 application. For this tutorial, we will assume
that `data` is an object of [_via_data.js](via-3.x.y/src/js/_via_data.js) defined
as follows:
```
var data = new _via_data();
```

All the other modules in VIA3 are designed to show or update this data based on
user interactions with the VIA3 application. For example, when a user adds a new
video file by selecting a local file, VIA3 updates the `data` using an instance
of [_via_file.js](via-3.x.y/src/js/_via_file.js) function as follows:
```
var file_id = '1';
var file = new _via_file(file_id,
                         'Alioli.ogv',
                         _VIA_FILE_TYPE.VIDEO,
                         _VIA_FILE_TYPE.URIHTTP,
                         'https://upload.wikimedia.org/wikipedia/commons/4/47/Alioli.ogv'
                        );
data.store['file'][file_id] = file;
```

Most manual annotation projects require annotation of a single file (i.e. image,
video or audio). However, there are some use cases which requires the human annotators to
annotate a pair of images. For example, a computer vision researcher working to
understand how humans quantify the quality of images will create a manual 
annotation project in which human annotators are shown a pair of images and are 
required to decide which of the two images has better quality. To address such 
use cases, VIA3 uses a concept of View which can contain any number of files.
For annotation of a single file, each view will contain a single file while for
annotation of a pair of files, each view will contain two files. Views are 
defined using the [_via_view.js](via-3.x.y/src/js/_via_view.js) function as 
follows:
```
var view_id = '1';
var file_id_list = ['1']; // a view containing a single file with file_id = '1'
var view = new _via_view(file_id_list);
data.store['view'][view_id] = view;
```

The sequential ordering of each view is defined by `data.store['vid_list']`. For 
example, if we want the user to first see view id `1`, then `3` and finally view
id `2`, we can defined `vid_list` as follows
```
data.store['vid_list'] = ['1', '3', '2'];
```

VIA3 allows human annotators to define and describe spatial regions
associated with an image or still frame from a video and temporal
segments associated with audio or video. Spatial regions are defined
using standard region shapes such as rectangle, circle, ellipse,
point, polygon, polyline, freehand drawn mask, etc.\ while the temporal
segments are defined by delineating start and end timestamps
(e.g.\ video segment from 3.1 sec. to 9.2 sec.).
Such spatial regions and temporal segments are described using textual
metadata. The function [_via_metadata.js](via-3.x.y/src/js/_via_metadata.js),
shown below, defines the data structure to hold all these variations of metadata:
```
const _VIA_RSHAPE  = { 'POINT':1, 'RECT':2, 'CIRCLE':3, 'ELLIPSE':4, 'LINE':5, 'POLYLINE':6, 'POLYGON':7 };
const _VIA_METADATA_FLAG = { 'VISIBLE':0, 'DELETED':1, 'HIDDEN':2, 'RESERVED1':4, 'RESERVED2':8 }

function _via_metadata(vid, z, xy, av) {
  this.vid = vid;   // view id
  this.flg = 0;     // flags: [deleted, hidden, ...]
  this.z   = z;     // [t0, ..., tn] (temporal coordinate e.g. time or frame index)
  this.xy  = xy;    // [shape_id, shape_coordinates, ...] (i.e. spatial coordinate)
  this.av  = av;    // attribute-value pair e.g. {attribute_id : attribute_value, ...}
}
```
Each metadata is associated with a view and this information is stored in `this.vid`.
The variable `this.z` is an array and stores the temporal coordinates associated 
with a metadata. For example, `this.z = [7.2,11.6]` defines a temporal segment 
from 7.2 sec to 11.6 sec. The spatial coordinates are stored in `this.xy`. For 
example, a rectangular bounding box of size (40,50) located at (10,20)
in an image is denoted by `this.xy = [_VIA_RSHAPE.RECT, 10, 20, 40, 50]`. The
`z` and `xy` variables store the location information about a metadata while
`this.av` stores the textual metadata defined by the user as a (key,value) pair.
For example, if a user defines a rectangular bounding box in a video frame at
time 8.3 sec as containing a white cat, then the metadata is stored as follows:

```
var metadata_id = '1_mR8ks-Aa'; // a unique metadata id
var metadata = new _via_metadata('1',
                                 0,
                                 [7.2, 11.6],
                                 [2, 10, 20, 40, 50],
                                 {
                                   'object':'cat',
                                   'colour':'white',
                                 }
                                );
data.store['metadata'][metadata_id] = metadata;
```

The attributes (e.g. `object` and `colour` in the above example) are defined
using the [_via_attribute.js](via-3.x.y/src/js/_via_attribute.js) function:
```
var attribute_id = '1'
var attribute = new _via_attribute('object',
                                   'FILE1_Z1_XY1',
                                   _VIA_ATTRIBUTE_TYPE.TEXT,
                                   'Name of Object');
data.store['attribute'][attribute_id] = attribute;
```
The `anchor_id` of `FILE1_Z1_XY1` indicates that this attribute will be defined
for a spatial region in a video frame. There are many more anchor types defined 
in [_via_attribute.js](via-3.x.y/src/js/_via_attribute.js) file. By default, 
each attribute is visible as a text box to human annotators. To reduce the 
cognitive load of human annotators, the attribute can be a dropdown, radio or
checkbox.

## Packing VIA as a Single HTML File
All the Javascript modules, CSS style files, HTML code can be packed into a 
single self contained and standalone HTML file as follows:
```
cd via-3.x.y/scripts
python3 pack_via.py video_annotator
python3 pack_via.py audio_annotator

python3 pack_demo.py video_annotator
python3 pack_demo.py audio_annotator
```

```
Author: Abhishek Dutta
Version: 16 May 2019
```