# TopoJSON

<a href="http://bl.ocks.org/4090870"><img src="/mbostock/topojson/wiki/example.png" width="960" height="500" alt="U.S. County mesh"></a>

TopoJSON is an extension of GeoJSON that encodes topology. Rather than representing geometries discretely, geometries in TopoJSON files are stitched together from shared line segments called *arcs*. This technique is similar to [Matt Bloch’s MapShaper](http://www.cartogis.org/docs/proceedings/2006/bloch_harrower.pdf
) and the [Arc/Info Export format, .e00](http://indiemaps.com/blog/2009/02/e00parser-an-actionscript-3-parser-for-the-arcinfo-export-topological-gis-format/).

TopoJSON eliminates redundancy, allowing related geometries to be stored efficiently in the same file. For example, the shared boundary between California and Nevada is represented only once, rather than being duplicated for both states. A single TopoJSON file can contain multiple feature collections without duplication, such as states and counties. Or, a TopoJSON file can efficiently represent both polygons (for fill) and boundaries (for stroke) as two feature collections that share the same arc mesh.

As a result, TopoJSON is substantially more compact than GeoJSON. The above shapefile of U.S. counties is 2.2M as a GeoJSON file, but only [436K](http://bl.ocks.org/4090870) as a boundary mesh, a reduction of 80.4% even without simplification. TopoJSON can also be more efficient to render since shared control points need only be projected once.

To further reduce file size, TopoJSON uses fixed-precision delta-encoding for integer coordinates rather than floats. This eliminates the need to round the precision of coordinate values (e.g., [LilJSON](https://github.com/migurski/LilJSON)), without sacrificing accuracy. Like GeoJSON, TopoJSON files are easily modified in a text editor and amenable to gzip compression.

Lastly, encoding topology has numerous useful applications for maps and visualization. It facilitates geometry simplification that preserves the connectedness of adjacent features; this applies even across feature collections, such as simultaneous consistent simplification of state and county boundaries. Topology can also be used for [Dorling cartograms](http://www.ncgia.ucsb.edu/projects/Cartogram_Central/types.html) and other techniques that need shared boundary information.

## Implementation

TopoJSON introduces a new object type, "Topology". A Topology has an array `objects` which can contain either features or geometry objects, and an array `arcs`. Each *arc* is a sequence of coordinates; thus, a single arc is equivalent to a LineString's coordinates, and the array of arcs is equivalent to a MultiLineString's coordinates. The arcs are stitched together to form the geometry, rather than storing the geometry on each object separately.

As such, geometry objects differ from the GeoJSON specification in terms of how their coordinates are specified. Any geometry object contained inside a Topology defines its coordinates in terms of a sequence of the Topology's arcs, referenced by zero-based index. For example, a LineString geometry might be defined as

```js
{"type": "LineString", "coordinates": [42]}
```

where *42* refers to the arc `topology.arcs[42]`. If multiple arcs need to be concatenated:

```js
{"type": "LineString", "coordinates": [42, 43]}
```

Similarly, a Polygon with a hole might be defined as

```js
{"type": "Polygon", "coordinates": [[42, 43], [44]]}
```

When stitching together arcs to form geometries, the last coordinate of the arc must be the same as the first coordinate of the subsequent arc, if any. Thus, for all arcs except the last arc, the last coordinate of the arc should be skipped while rendering. For example, if arc 42 represents the point sequence A → B → C, and arc 43 represents the point sequence C → D → E, then the line string [42, 43] represents the point sequence A → B → C → D → E.

A negative index indicates that the sequence of coordinates in the arc should be reversed before stitching. To avoid ambiguity with zero, the two's complement is used; -1 (~0) represents the reversed arc 0, -2 (~1) represents the reversed arc 1, and so on.

A Topology has an array `arcs` which is an array of line strings. Each point in the line string is specified at least two dimensions: x and y. Each coordinate is represented as an integer value relative to the previous point, or relative to the origin ⟨0,0⟩ for the first point. To convert to latitude and longitude (or absolute coordinates), the topology also defines a `transform`. For example:

```js
"transform": {
  "scale": [0.035896033450880604, 0.005251163636665131],
  "translate": [-179.14350338367416, 18.906117143691233]
}
```

To convert from relative integer coordinates to fixed integer coordinates, keep a running sum while iterating over the arc. To convert from fixed integer coordinates to absolute coordinates, use the following expression:

```js
function fixedToAbsolute(point) {
  return [
    point[0] * transform.scale[0] + transform.translate[0],
    point[1] * transform.scale[1] + transform.translate[1]
  ];
}
```

An optional z-dimension can be used for dynamic simplification; the visual importance of the given control point as computed by the simplification algorithm (e.g., Visvalingham) is stored in TopoJSON so that the geometry can be rapidly simplified as needed for different zoom levels.

## Under Consideration

* Make feature `properties` optional; alternatively prefer using `id` on geometry objects.
* How to represent Points? (MultiPoint could reference a line string.) In what situations would points be affected by topology-preserving simplification?
