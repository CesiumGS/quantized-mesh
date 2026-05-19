
# JSON Format Specification

This section describes the format of the `layer.json` manifest file. The format is derived from [tilejson](https://github.com/mapbox/tilejson-spec), but is not a strict subset of this format. The JSON schema for this file can be found in [`layer.schema.json`](./schema/layer.schema.json). The following table summarizes the properties that can be contained in this JSON file, their data types, default values, and interpretation:


| *Property* | *Type* | *Description* | *Required* |
|---|---|---|---|
| name | `string` | The name of this terrain tileset. | No. Default: `"Terrain"` |
| description | `string` | The description of this terrain tileset. | No. Default: `""` |
| attribution | `string` | The attribution (credit) string for the terrain. | No. Default: `""` |
| version | `string` | The version of this tileset. | No. Default: `"1.0.0"` |
| format | `string` | The format of the terrain tiles. Should be `"quantized-mesh-1.0"`. | No. Default: `"quantized-mesh-1.0"` |
| scheme | `string` | The tiling scheme. The only valid value is `"tms"`. | No. Default: `"tms"` |
| extensions | `string[]` | The extensions available for this tileset. | No. Default: `undefined` |
| projection | `string` | The map projection of this tileset. Valid values are `"EPSG:4326"` and `"EPSG:3857"`. | No. Default: `"EPSG:4326"` |
| parentUrl | `string` | The URL of the parent layer.json that this one is layered on top of. When this is defined, then availability information and the presence of extensions should also be looked up in the parent. Attributions from parents should be integrated into the overall attribution. | No. Default: `undefined` |
| minzoom | `integer` | The minimum level for which there are any available tiles. | No. Default: `0` |
| maxzoom | `integer` | The maximum level for which there are any available tiles. | Yes |
| bounds | `number[4]` | The bounding box of the terrain, expressed as `[west, south, east, north]`, in degrees. This is not required for calculating tile positions or downloads. Implementations may choose to ignore this property. | No. Default: `[-180,-90,180,90]` |
| tiles | `string[]` | The URL templates from which to obtain tiles. | Yes |
| available | `availabilityRectangle[][]` | The tile availability information. The outer array is indexed by tile level. The inner array is a list of rectangles of availability at that level. Tiles themselves may also contain further availability information for their subtree. If `metadataAvailability` is present, then this property should be ignored. | No. Default: `undefined` |
| metadataAvailability | `integer` | The levels at which metadata is found in tiles. A value of `n` indicates that metadata can be found at every `n`th level, starting at 0. For example, if this value is 10, then metadata is found at levels 0, 10, 20, etc. If this property is present, then the `available` property should be ignored.| No. Default: `undefined` |

## About `available` and `metadataAvailability`

The `available` property is a predecessor of what is now stored in the `metadataAvailability`. When the `metadataAvailability` is present, then the `available` property should be ignored. Specifically: When the `metadataAvailability` is present, then the structure of the `available` property may not properly reflect the actual availability. 

## About `tiles`

The `tiles` property is an array of template URLs. This array should have a length of exactly 1. Clients may choose to ignore all but the first array element. The template URLs can contain variables `{x}`, `{y}`, `{z}`, and `{version}`. The `{z}` variable indicates the level of the tile. The `{x}` and `{y}` variables indicate the x- and y coordinate of the tile. The `version` indicator is a hint for clients or users to indicate whether the underlying data changed over time. When clients or users are caching the data in any form, then a change in the version number should cause that cache to be invalidated.

An example of the entry of the `tiles` array is 
`"{z}/{x}/{y}.terrain?v={version}"`
At runtime, this will be converted into an actual URL by substituting the template variables accordingly - for example:
`5/46/21.terrain?v=1.2.0`

## About `extensions`

The `extensions` is an array of strings that indicate different flavors of the data that may be requested. The elements of this array can be added as query parameters in the request, with the key being `extensions`, and the value being the names of the desired extensions, separated by a `-` dash. For example, when the layer JSON indicates extensions
```json
"extensions": [
    "watermask",
    "metadata",
    "octvertexnormals"
]
```
then the query parameter can be
`extensions=octvertexnormals-watermask-metadata`
The request URL (including the substituted template variables) and these parameters would therefore be
`12/6074/2684.terrain?extensions=octvertexnormals-watermask-metadata&v=1.2.0`

## About `minzoom`

While the value of `minzoom` can technically be greater than 0, some details of the behavior are not clearly specified for this case. Many clients will ignore the `minzoom` value, and assume the availability information to be present for layer 0.

## About the refinement process

_This section is non-normative!_

Client implementations will usually traverse the tile hierarchy for a quantized mesh, replacing the rendered representation of one tile with its child tiles, until the desired refinement level is achieved. During this process, the availability of child tiles is checked with the `available`- or `metadataAvailability` properties. The availability will be looked up either directly, or via the layer information that is found under the `parentUrl`.

During this process, it can happen that only _some_ of the four children of a given parent tile are marked as being available.

Clients have different options for handling this case:

- Clients _could_ omit the child tiles that are not available. This would cause gaps in the visual representation of the tiles, and is therefore not recommended
- Clients can take the geometry of the parent tile, split it into four parts, and use the respective parts from the parent geometry to fill the gaps that would otherwise be caused by children that are not available
- Clients can generate artificial "fill tiles" to fill the gaps. These tiles will usually be generated at runtime, by aligning the borders with the geometry of the surrounding tiles, and interpolating the vertex heights bilinearly between the heights of the border vertices.

