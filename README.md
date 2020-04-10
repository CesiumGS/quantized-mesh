# quantized-mesh-1.0 terrain format

Have a question? Discuss the quantized-mesh specification on the [Cesium forum](http://cesiumjs.org/forum.html).

A terrain tileset in quantized-mesh-1.0 format is a simple multi-resolution quadtree pyramid of heightmaps. All tiles have the extension .terrain. So, if the Tiles URL for a tileset is:

```
http://example.com/tiles
```

Then the two root files of the pyramid are found at these URLs:

* (-180 deg, -90 deg) - (0 deg, 90 deg) - http://example.com/tiles/0/0/0.terrain
* (0 deg, -90 deg) - (180 deg, 90 deg) - http://example.com/tiles/0/1/0.terrain

The eight tiles at the next level are found at these URLs:

* (-180 deg, -90 deg) - (-90 deg, 0 deg) - http://example.com/tiles/1/0/0.terrain
* (-90 deg, -90 deg) - (0 deg, 0 deg) - http://example.com/tiles/1/1/0.terrain
* (0 deg, -90 deg) - (90 deg, 0 deg) - http://example.com/tiles/1/2/0.terrain
* (90 deg, -90 deg) - (180 deg, 0 deg) - http://example.com/tiles/1/3/0.terrain
* (-180 deg, 0 deg) - (-90 deg, 90 deg) - http://example.com/tiles/1/0/1.terrain
* (-90 deg, 0 deg) - (0 deg, 90 deg) - http://example.com/tiles/1/1/1.terrain
* (0 deg, 0 deg) - (90 deg, 90 deg) - http://example.com/tiles/1/2/1.terrain
* (90 deg, 0 deg) - (180 deg, 90 deg) - http://example.com/tiles/1/3/1.terrain

When requesting tiles, be sure to include the following HTTP header in the request:
```
Accept: application/vnd.quantized-mesh,application/octet-stream;q=0.9
```

Otherwise, some servers may return a different representation of the tile than the one described here.

Each tile is a specially-encoded triangle mesh where vertices overlap their neighbors at tile edges. In other words, at the root, the eastern-most vertices in the western tile have the same longitude as the western-most vertices in the eastern tile.

Terrain tiles are served gzipped. Once extracted, tiles are little-endian, binary data. The first part of the file is a header with the following format. Doubles are IEEE 754 64-bit floating-point numbers, and Floats are IEEE 754 32-bit floating-point numbers.

```C++
struct QuantizedMeshHeader
{
    // The center of the tile in Earth-centered Fixed coordinates.
    double CenterX;
    double CenterY;
    double CenterZ;

    // The minimum and maximum heights in the area covered by this tile.
    // The minimum may be lower and the maximum may be higher than
    // the height of any vertex in this tile in the case that the min/max vertex
    // was removed during mesh simplification, but these are the appropriate
    // values to use for analysis or visualization.
    float MinimumHeight;
    float MaximumHeight;

    // The tile’s bounding sphere.  The X,Y,Z coordinates are again expressed
    // in Earth-centered Fixed coordinates, and the radius is in meters.
    double BoundingSphereCenterX;
    double BoundingSphereCenterY;
    double BoundingSphereCenterZ;
    double BoundingSphereRadius;

    // The horizon occlusion point, expressed in the ellipsoid-scaled Earth-centered Fixed frame.
    // If this point is below the horizon, the entire tile is below the horizon.
    // See http://cesiumjs.org/2013/04/25/Horizon-culling/ for more information.
    double HorizonOcclusionPointX;
    double HorizonOcclusionPointY;
    double HorizonOcclusionPointZ;
};
```

Immediately following the header is the vertex data. An `unsigned int` is a 32-bit unsigned integer and an `unsigned short` is a 16-bit unsigned integer.

```C++
struct VertexData
{
    unsigned int vertexCount;
    unsigned short u[vertexCount];
    unsigned short v[vertexCount];
    unsigned short height[vertexCount];
};
```

The `vertexCount` field indicates the size of the three arrays that follow. The three arrays contain the delta from the previous value that is then zig-zag encoded in order to make small integers, regardless of their sign, use a small number of bits. Decoding a value is straightforward:

```javascript
var u = 0;
var v = 0;
var height = 0;

function zigZagDecode(value) {
    return (value >> 1) ^ (-(value & 1));
}

for (i = 0; i < vertexCount; ++i) {
    u += zigZagDecode(uBuffer[i]);
    v += zigZagDecode(vBuffer[i]);
    height += zigZagDecode(heightBuffer[i]);

    uBuffer[i] = u;
    vBuffer[i] = v;
    heightBuffer[i] = height;
}
```

Once decoded, the meaning of a value in each array is as follows:

| Field | Meaning |
| ----- | ------- |
| u | The horizontal coordinate of the vertex in the tile. When the u value is 0, the vertex is on the Western edge of the tile. When the value is 32767, the vertex is on the Eastern edge of the tile. For other values, the vertex's longitude is a linear interpolation between the longitudes of the Western and Eastern edges of the tile. |
| v | The vertical coordinate of the vertex in the tile. When the v value is 0, the vertex is on the Southern edge of the tile. When the value is 32767, the vertex is on the Northern edge of the tile. For other values, the vertex's latitude is a linear interpolation between the latitudes of the Southern and Nothern edges of the tile. |
| height | The height of the vertex in the tile. When the height value is 0, the vertex's height is equal to the minimum height within the tile, as specified in the tile's header. When the value is 32767, the vertex's height is equal to the maximum height within the tile. For other values, the vertex's height is a linear interpolation between the minimum and maximum heights. |

Immediately following the vertex data is the index data. Indices specify how the vertices are linked together into triangles. If tile has more than 65536 vertices, the tile uses the `IndexData32` structure to encode indices. Otherwise, it uses the `IndexData16` structure.

To enforce proper byte alignment, padding is added before the IndexData to ensure 2 byte alignment for `IndexData16` and 4 byte alignment for `IndexData32`.

```C++
struct IndexData16
{
    unsigned int triangleCount;
    unsigned short indices[triangleCount * 3];
}

struct IndexData32
{
    unsigned int triangleCount;
    unsigned int indices[triangleCount * 3];
}
```

Indices are encoded using the high water mark encoding from [webgl-loader](https://code.google.com/p/webgl-loader/). Indices are decoded as follows:

```javascript
var highest = 0;
for (var i = 0; i < indices.length; ++i) {
    var code = indices[i];
    indices[i] = highest - code;
    if (code === 0) {
        ++highest;
    }
}
```

Each triplet of indices specifies one triangle to be rendered, in counter-clockwise winding order. Following the triangle indices is four more lists of indices:

```C++
struct EdgeIndices16
{
    unsigned int westVertexCount;
    unsigned short westIndices[westVertexCount];

    unsigned int southVertexCount;
    unsigned short southIndices[southVertexCount];

    unsigned int eastVertexCount;
    unsigned short eastIndices[eastVertexCount];

    unsigned int northVertexCount;
    unsigned short northIndices[northVertexCount];
}

struct EdgeIndices32
{
    unsigned int westVertexCount;
    unsigned int westIndices[westVertexCount];

    unsigned int southVertexCount;
    unsigned int southIndices[southVertexCount];

    unsigned int eastVertexCount;
    unsigned int eastIndices[eastVertexCount];

    unsigned int northVertexCount;
    unsigned int northIndices[northVertexCount];
}
```

These index lists enumerate the vertices that are on the edges of the tile. It is helpful to know which vertices are on the edges in order to add skirts to hide cracks between adjacent levels of detail.

## Tiling Scheme and Coordinate System

By default, the data is tiled according to the [Tile Map Service (TMS)](http://wiki.osgeo.org/wiki/Tile_Map_Service_Specification) layout and global-geodetic system. These defaults can be varied by specifying the `projection` and `scheme`.

Allowed values for the projection are `EPSG:3857` ([Web Mercator](https://en.wikipedia.org/wiki/Web_Mercator_projection) as used by Google Maps) and `EPSG:4326` (Lat/Lng coordinates in the [Global-Geodetic System](https://en.wikipedia.org/wiki/World_Geodetic_System)). It is worth noting that the `EPSG3857` projection has only 1 tile at the root (zoom level 0) while `EPSG:4326` has 2.

The options for the tiling scheme are `tms` and `slippyMap`. The Y coordinates are numbered from the south northwards (eg. latitudes) in the `tms` standard whereas `slippyMap` coordinates have their origin at top left (NW).

For Cesium terrain layers these options can be set in the `layer.json` manifest file. If not specified, they default
to `EPSG:4326` and `tms`.

## Extensions
Extension data may follow to supplement the quantized-mesh with additional information. Each extension begins with an `ExtensionHeader`, consisting of a unique identifier and the size of the extension data in bytes. An `unsigned char` is a 8-bit unsigned integer.

```C++
struct ExtensionHeader
{
    unsigned char extensionId;
    unsigned int extensionLength;
}
```

As new extensions are defined, they will be assigned a unique identifier. If no extensions are defined for the tileset, an `ExtensionHeader` will not included in the quanitzed-mesh. Multiple extensions may be appended to the quantized-mesh data, where ordering of each extension is determined by the server.

Multiple extensions may be requested by the client by delimiting extension names with a `-`. For example, a client can request vertex normals and watermask using the following Accept header:

```
Accept : 'application/vnd.quantized-mesh;extensions=octvertexnormals-watermask'
```

The following extensions may be defined for a quantized-mesh:

### Terrain Lighting

__Name:__ Oct-Encoded Per-Vertex Normals

__Id:__ 1

__Description:__ Adds per vertex lighting attributes to the quantized-mesh. Each vertex normal uses oct-encoding to compress the traditional x, y, z 96-bit floating point unit vector into an x,y 16-bit representation. The 'oct' encoding is described in "A Survey of Efficient Representations of Independent Unit Vectors", Cigolle et al 2014: http://jcgt.org/published/0003/02/01/

__Data Definition:__
```C++
struct OctEncodedVertexNormals
{
    unsigned char xy[vertexCount * 2];
}
```

__Requesting:__ For oct-encoded per-vertex normals to be included in the quantized-mesh, the client must request this extension by using the following HTTP Header:
```
Accept : 'application/vnd.quantized-mesh;extensions=octvertexnormals'
```

__Comments:__ The original implementation of this extension was requested using the extension name `vertexnormals`. The `vertexnormals` extension identifier is deprecated and implementations must now request vertex normals by adding `octvertexnormals` in the request header extensions parameter, as shown above.

### Water Mask
__Name:__ Water Mask

__Id:__ 2

__Description:__ Adds coastline data used for rendering water effects. The water mask is either 1 byte, in the case that the tile is all land or all water, or it is `256 * 256 * 1 = 65536` bytes if the tile has a mix of land and water. Each mask value is 0 for land and 255 for water. Values in the mask are defined from north-to-south, west-to-east; the first byte in the mask defines the watermask value for the northwest corner of the tile. Values between 0 and 255 are allowed as well in order to support anti-aliasing of the coastline.

__Data Definition:__

A Terrain Tile covered entirely by land or water is defined by a single byte.
```C++
struct WaterMask
{
    unsigned char mask;
}
```

A Terrain Tile containing a mix of land and water define a 256 x 256 grid of height values.

```C++
struct WaterMask
{
    unsigned char mask[256 * 256];
}
```

__Requesting:__ For the watermask to be included in the quantized-mesh, the client must request this extension by using the following HTTP Header:
```
Accept : 'application/vnd.quantized-mesh;extensions=watermask'
```

### Metadata
__Name:__ Metadata

__Id:__ 4

__Description:__ Adds a JSON object to each tile that can store extra information about the tile. Potential uses include storing the what type of land is in the tile (eg. forest, desert, etc) or availability of child tiles.

__Data Definition:__

```C++
struct Metadata
{
    unsigned int jsonLength;
    char json[jsonLength];
}
```

__Requesting:__ For the metadata to be included in the quantized-mesh, the client must request this extension by using the following HTTP Header:
```
Accept : 'application/vnd.quantized-mesh;extensions=metadata'
```
