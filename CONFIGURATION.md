Configuring Tilemaker
=====================

Vector tiles contain (generally thematic) 'layers'. For example, your tiles might contain river, cycleway and railway layers.

You'll generally assign OSM data into layers by making decisions based on their tags. You might put anything with a `highway=` tag into the roads layer, anything with a `railway=` tag into the railway layer, and so on.

In Tilemaker, you achieve this by writing a short script in the Lua programming language. Lua is a simple and fast language used by several other OpenStreetMap tools, such as the OSRM routing engine and osm2pgsql.

In addition, you supply Tilemaker with a JSON file which specifies certain global settings for your tileset.

A note on zoom levels
---------------------

Because vector tiles are so efficiently encoded, you generally don't need to create tiles above (say) zoom level 14. Instead, your renderer will use the data in the z14 tiles to generate z15, z16 etc. (This is called 'overzooming'.)

So when you set a maximum zoom level of 14 in Tilemaker, this doesn't mean you're restricted to displaying maps at z14. It just means that Tilemaker will create z14 tiles, and it's your renderer's job to use these tiles to draw the most detailed maps.

JSON configuration
------------------

The JSON config file sets out the layers you'll be using, and which zoom levels they apply to. For example, you might want to include your roads layer in your z12-z14 tiles, but your buildings at z14 only.

It also includes these global settings:

* `minzoom` - the minimum zoom level at which any tiles will be generated
* `maxzoom` - the maximum zoom level at which any tiles will be generated
* `basezoom` - the zoom level for which Tilemaker will generate tiles internally (should usually be the same as `maxzoom`)
* `include_ids` - whether you want to store the OpenStreetMap IDs for each way/node within your vector tiles
* `compress` - whether to compress (deflate) vector tiles
* `name`, `version` and `description` - about your project (these are written into the MBTiles file)

A typical config file would look like this:

	{
		"layers": {
			"roads": { "minzoom": 12, "maxzoom": 14 },
			"buildings": { "minzoom": 14, "maxzoom": 14 },
			"pois": { "minzoom": 13, "maxzoom": 14 }
		},
		"settings": {
			"minzoom": 12,
			"maxzoom": 14,
			"basezoom": 14,
			"include_ids": false,
			"compress": true,
			"name": "Tilemaker example",
			"version": "0.1",
			"description": "Sample vector tiles for Tilemaker"
		}
	}

The order of layers will be carried forward into the vector tile.

Note that all options are compulsory. If Tilemaker baulks at the JSON file, check everything's included, and run it through an online JSON validator to check for syntax errors.

By default Tilemaker expects to find this file at config.json, but you can specify another filename with the `--config` command-line option.

### Additional metadata

If you need to add additional metadata fields to your .mbtiles output, include the keys/values as an (optional) "metadata" entry under "settings". These will usually be string key/value pairs. (The value can also be another JSON entity - hash, array etc. - in which case it'll be encoded as JSON when written into the .mbtiles metadata table.)

For example:

	{
		"layers": { ... },
		"settings": { ... ,
			"metadata": {
				"author": "THERE Data Inc",
				"licence": "ODbL 1.1",
				"layer_order": { "water": 1, "buildings": 2, "roads": 3 }
			}
		}
	}
	
Lua processing
--------------

Your Lua file needs to supply three things:

1. `node_keys`, a list of those OSM keys which indicate that a node should be processed
2. `node_function`, a function to process an OSM node and add it to layers
3. `way_function`, a function to process an OSM way and add it to layers

`node_keys` is a simple list (or in Lua parlance, a 'table') of OSM tag keys. If a node has one of those keys, it will be processed by `node_function`; if not, it'll be skipped. For example, if you wanted to show highway crossings and railway stations, it should be `{ "highway", "railway" }`. (This avoids the need to process the vast majority of nodes which contain no important tags at all.)

`node_function` and `way_function` work the same way. They are called with an OSM object; you then inspect the tags of that object, and put it in your vector tiles' layers based on those tags. In essence, the process is:

* look at tags
* if tags meet criteria, write to a layer
* (optionally) add attributes (= vector tile metadata/tags) 

To do that, you use these methods:

* `node:Find(key)` or `way:Find(key)`: get the value for a tag, or the empty string if not present. For example, `way:Find("railway")` might return "rail" for a railway, "siding" for a siding, or "" if it isn't a railway at all.
* `node:Holds(key)` or `way:Holds(key)`: returns true if that key exists, false otherwise.
* `node:Id()` or `way:Id()`: get the OSM ID of the current object.
* `node:Layer("layer_name", false)` or `way:Layer("layer_name", is_area)`: write this node/way to the named layer. This is how you put objects in your vector tile. is_area (true/false) specifies whether a way should be treated as an area, or just as a linestring.
* `node:Attribute(key,value)` or `node:Attribute(key,value)`: add an attribute to the most recently written layer.
* `node:AttributeNumeric(key,value)`, `node:AttributeBoolean(key,value)` (and `way:`...): for numeric/boolean columns.

The simplest possible function, to include roads/paths and nothing else, might look like this:

    function way_function(way)
      local highway = way:Find("highway")
      if highway~="" then
        way:Layer("roads", false)
        way:Attribute("name", way:Find("name"))
        way:Attribute("type", highway)
      end
    end

Take a look at the supplied process.lua for a full example. You can specify another filename with the `--process` option.

If your Lua file causes an error due to mistaken syntax, you can test it at the command line with `luac -p filename`. Three frequent Lua gotchas: tables (arrays) start at 1, not 0; the "not equal" operator is `~=` (that's the other way round from Perl/Ruby's regex operator); and `if` statements always need a `then`, even when written over several lines.

Relations
---------

Tilemaker handles multipolygon relations natively. The combined geometries are processed as ways (i.e. by `way_function`), so if your function puts buildings in a 'buildings' layer, Tilemaker will cope with this whether the building is mapped as a simple way or a multipolygon. The only difference is that they're given an artificial ID.

Multipolygons are expected to have tags on the relation, not the outer way. The vector tile spec is [slightly vague](https://github.com/mapbox/vector-tile-spec/issues/30) on multipolygon encoding. Tilemaker will enforce correct winding order, but in the case of a multipolygon with multiple outer ways, it assigns all inner ways to the first outer way.