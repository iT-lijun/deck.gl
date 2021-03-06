# RFC: Using Binary Data

Authors: Ib Green, ...
Date: June 13, 2018
Status: **Implemented**


## Overview

This RFC is written as an article for the deck.gl Developer's Guide, rather than in the standard format. The intention is to copy this text to the developer's guide once it matures.

> This documents references some in-progress or hypothetical functionality, not everything is implemented yet. The intent is to show a "complete story" that will help us understand how each piece fits in to the big picture.


## Main Text

While all deck.gl layers are designed to work with classic JavaScript arrays of objects as `data`, it is sometimes desirable for applications to work directly with binary data and to be able to pass such binary data to layers. The motivation is often performance-related but it could just be that the application happens to have data in a binary format.

This article focused on the following use cases:

* **Using typed arrays as layer input data** - Using The input data is in binary form, perhaps it is delivered this way from the back-end, and it would be preferable to not have to unpack it.
* **Using typed arrays or GPU buffers directly as layer attributes** - The input data is already formatted in the memory format expected by the GPU and deck.gl's shaders.

## Using Flattend Data (Typed Arrays as layer input data

In some cases the application may receive all or parts of a layer's geometry in the form of binary typed arrays instead of Javascript arrays. Typed arrays can sometimes be transported more efficiently in binary form than classic arrays that tend to be JSON stringified and then parsed.

One of the biggest differences that makes it hard to use binary arrays directly is that they are flat, whereas JavaScript arrays tend to have a sub-array for each vertex or color.

TBA...

deck.gl layers carefully formats data in memory so that it can be accessed efficiently by the GPU...


## Passing Typed Arrays directly to layer attributes

deck.gl layers will look for each attribute name in the `props` and use a provided typed array (and upload it to a GPU Buffer) instead of attempting to build its own buffer from data.

## Passing GPU Buffers directluy to layer attributes

deck.gl layers will look for each attribute name in the `props` and use a provided GPU Buffer instead of attempting to build its own buffer from data.


## Binary Layer Formats

deck.gl provides the ability to supply binary data directly to layers.

This is an overview of the binary formats of layers in the core deck.gl layer catalog. This is not intended to cover all layers. Ultimately, each layer needs to specify its own binary format. When in doubt, consult the layer source code.

> Note that directly feeding in binary data is considered "experimental" in the sense that the binary format is not guaranteed to stay unchanged between minor releases. While changes to the binary format will generally be avoided, it is sometimes necessary for e.g. optimization reasons or to implement a new feature in the best way.


### One-to-one, instanced layers

These layers have very straight-forward binary representations and it is very easy to generate the required attributes, even on a server and send pre-formatted data to the client application which can then be uploaded to GPU and passed in as attributes.

| `Layer`            | `Type`     | Accessors |
| ---                | ---        | --- |
| `PointCloudLayer`  | 1-to-1     | `instancePositions` `instanceColors`, ... |
| `ScatterPlotLayer` | 1-to-1     | `instancePositions` `instanceColors`, ... |
| `LineLayer`        | 1-to-1     | `instancePositions` `instanceColors`, ... |
| `ArcLayer`         | 1-to-1     | |
| `GridCellLayer`    | 1-to-1     | |
| `HexagonCellLayer` | 1-to-1     | |
| `IconLayer`        | 1-to-1     | |
| `TextLayer`        | 1-to-1     | |


### Custom Geometry layers

* Main geometry (positions and supplementary attributes) - for some layers, this layout is more complicated and described here.
* Per-vertex copies of values: A number of attributes are per-vertex copies of some per-object value (color, elevation, etc) and can be generated in an automated way.
* May use custom number of vertices per object, or custom number of instances per object.

| `Layer`            | `Type`     | Accessors |
| ---                | ---        | --- |
| `PathLayer`        |            | |
| `PolygonLayer`     |            | |
| `SolidPolygonLayer`|            | |
| `GeoJsonLayer`     |            | |


### Aggregating Layers

TBD: Binary data for aggregating layers is not currently considered, but would focus on providing the input data in binary form and extending the aggregation algorithms to work on flattened data.

| `Layer`            | `Type`      | Accessors |
| ---                | ---         | --- |
| `HexagonLayer`     | aggregating | See `HexagonCellLayer` |
| `GridLayer`        | aggregating | See `GridCellLayer` |
| `ScreenGridLayer`  | aggregating | ... |


## One Value Per-Instance Layers

Layers like scatterplot layer, point cloud layer, line layer have very simple data structures.


##  PathLayer

| `startPos`   | v0.x | v0.y | v0.z | v1.x | v1.y | v1.z | ... | vn-2.x | vn-2.y | vn-2.z |
| `endPos`     | v1.x | v1.y | v1.z | v2.x | v2.y | v2.z | ... | vn-1.x | vn-1.y | vn-1.z |
| `leftDelta`  |    v0 - vn-2       |      v1 - v0       | ... | vn-2 - vn-3 |
| `rightDelta` |    v1 - v0         |      v2 - v1       | ... | vn-2 - vn-3 |



## Background Information

If you plan to work with binary data, you will want to make sure you are up to speed on JavaScript typed arrays as well as GPU buffers, both in terms of general concepts as well as a basic graps of the API.


### CPU vs GPU memory

Ultimately all deck.gl rendering is done on the GPU, and all memory must be available to the GPU by being "uploaded" into GPU memory `Buffers`. By pre-creating GPU buffers and passing these to deck.gl layers you exercise the maximum amount of control of memory management.

```js
import {Buffer} from 'luma.gl';
const buffer = new Buffer(gl, {data: });
```

Note that GPU buffers have many advanced features. They can be interleaved etc. The luma.gl `Accessor` class lets you describe how GPU Buffers should be accessed.


## Binary array manipulation utilities

luma.gl offers a suite of array manipulation utilities

* `copyArray`
* `fillArray`
* `flattenArray`


### Ways to pack, transport and unpack binary data

A standard that can be of interest when working with binary data is glTF (or more specifically, the GLB container part of glTF).


# GLB Binary Container Protocol Format

TBA: luma.gl comes with parsing support for the glTF / GLB binary container format. In this format, each message is packaged in a binary container or "envelope", that contains two "chunks":
* a JSON chunk containing the JSON encoding of semantic parts of the data
* a BIN chunk containing compact, back-to-back binary representations of large numeric arrays, images etc.

An intended benefit of the binary format is that large segments of data such as encoded images or raw buffers can be sent and processed natively rather than via JSON. Also Typed Array views can be created directly into the loaded data, minimizing copying.


# Parsing Support

GLB parsing functions will decode the binary container, parse the JSON and resolve binary references. The application will get a "patched" JSON structure, with the difference from the basic JSON protocol format being that certain arrays will be compact typed arrays instead of classic JavaScript arrays.

Typed arrays do not support nesting so all numbers will be laid out flat and the application needs to know how many values represent one element, for instance 3 values represent the `x, y, z` coordinates of a point.


## Details on Binary Container Format

The container format is an implementation of the GLB binary container format defined in the glTF specification. However the `accessor`/`bufferView` tables specified by `glTF` are quite verbose in the case where many small arrays (buffers) are used and apps may want to consider use custom tables for more compact messages.

References:
* [glTF 2 Poster](https://raw.githubusercontent.com/KhronosGroup/glTF/master/specification/2.0/figures/gltfOverview-2.0.0a.png)
* [glTF 2 Spec](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0)


## Endianness

In most cases, when working with binary data, care must be taken to properly define and respect "endianness" which essentially deswc
* glTF is little endian: GLB header is little endian. glTF Buffer contents are specified to be little endian.
* Essentially all of the current web is little endian, so potential big-endian issues are igored for now.
