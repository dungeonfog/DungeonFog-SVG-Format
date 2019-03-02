# DungeonFog SVG Export Format

Next to traditional bitmap image export, DungeonFog also supports exporting as SVG. This feature is currently designed for two applications:

* Using a vinyl or laser cutter to cut your dungeon exported as a bitmap to exactly the room shapes.
* Converting to light-blockers for virtual tabletop systems.

The goal is to allow the former without any custom importer for common vector graphics applications. The latter requires more information than available in any standard format, so we're extending SVG with the additional information required for this.

Note that the format is not finalized and might change in the future. This also means that if you have any suggestions on what to change, please do not hesistate to contact us.

## Basic Structure

SVG is an XML-based format and as such, supports adding additional information that's ignored by standard SVG importers in an extra namespace. DungeonFog uses two of these:

* `https://www.dungeonfog.com/1/svg` for DungeonFog-specific information (`1` is the version number).
* `http://www.inkscape.org/namespaces/inkscape` for additional information that is supported by Inkscape, but not the SVG standard.

Only the selected level will be exported to the SVG, even if lower levels are visible in the bitmap.

The width and height attributes on the root `svg` element define the size of the exported map in millimeters. This happens irrespectively of what the user has set as the unit in the exporter, it will always be converted to millimeters. The viewBox attribute defines the size in pixels (the origin always is at 0,0 at the moment).

The root element also contains the map and level id used by the DungeonFog systems. You could reconstruct the URL to open the editor for this map and this level using this information.

An example root tag might be

`<svg xmlns="http://www.w3.org/2000/svg" xmlns:inkscape="http://www.inkscape.org/namespaces/inkscape" width="1270mm" height="889mm" viewBox="0 0 3500 2450" xmlns:dgnfog="https://www.dungeonfog.com/1/svg" dgnfog:map="5b2ec6042155bd0f928245c35340ce2a" dgnfog:level="65016b43-4c42-4feb-8db3-086c22a2e19b"/>`

The XML content is split into two layers using `<g/>` tags, the rooms and the groups. The are defined as

`<g inkscape:groupmode="layer" inkscape:label="Rooms" id="rooms"/>` and

`<g inkscape:groupmode="layer" inkscape:label="Props" id="props"/>`.

Note that these layers are defined as such using the Inkscape extensions, since svg itself does not have a concept of layers (only groups).

## The Rooms Layer

The rooms layer contains a flat list of rooms (room groups are flattened into a global list). A room itself is declared as a group with the id of the room in the DungeonFog editor (so if you export the same room multiple times, this id won't change, unless you clone the map beforehand).

`<g id="d8af4528-9127-4403-ce57-ec67d5b694b4"/>`

### The Wall

This group contains a `<path/>` element that optionally contains a `<title/>` element with the room's name (if it has one). The path defines the room's wall shape using the `d` attribute with the standard SVG path definitions (using absolute coordinates). Only the following ones are used at the moment:

* M*x*,*y* for the initial starting point
* L*x*,*y* for straight walls
* A*r*,*r*,*s*,0,*d*,*x*,*y* for arcs (*r* = radius, *s* = starting angle in degrees, *d* is 0 for inwards facing arcs and 1 for outwards facing ones, *x*,*y* are the end coordinates)
* C*c1x*,*c1y*,*c2x*,*c2y*,*x*,*y* for cubic bezier curves (*c1* is the first control point, *c2* is the second control point and *x*,*y* are the end coordinates)
* Z is used for closing the path (room walls are always closed paths)

This path's fill and stroke are always the same and only used so the path is visible in vector programs. The path tag also contains a `dgnfog:hidden` attribute that contains a list of all hidden wall segments. This list is separated by a single space character. The values are non-negative integers, with 0 being the segment that directly follows the M command in the path definition. The list might be empty (if all walls are visible).

Note that if rooms overlap each other, the wall shape does not subtract that part of the other wall's shape. This means that walls of different rooms can intersect, even though this is not visible in the bitmap export.

### Doors

Doors are listed in the wall group they belong to as `<line/>` tags. If a door is on multiple walls, the one it belongs to of these is arbitrary, but it will only be listed once.

The line segment contains the attribute `dgnfog:type="door"` to indicate that it's a door. It has the coordinates `x1`,`y1` and `x2`,`y2` that define the start and end coordinates of the door. The id of the tag is the id of the door in the internal definition (and so stays constant over multiple exports).

For arcs, there is a special case for doors. If the wall cutout is enabled in the editor, the door's start and end coordinates are on the wall shape. Otherwise, the center of the door is on the wall shape (e.g. line is on a tangent).

## The Props Layer

All props of the whole level are combined into a single flat list, including flattening of all prop groups and all props in all rooms.

There are three types of props:

* image
* shape
* token

There is one `<g/>` tag per prop with a `dgnfog:type` attribute containing one of the three items listed above. It also contains the id of the prop and an optional title tag if the prop has a custom name.

### Image Props

Image props are the most common type of prop. Their group tag contains the rotation using a `transform` attribute with the content `rotate(angle,x,y)`, where the angle is the rotation in degrees and x,y is the center for this rotation, which also is the position of the prop (the rotation is always defined relative to the center of the prop's image).

The image prop's outline is defined using a standard `<rect/>` tag (using the `x`, `y`, `width`, `height` attributes) inside the group. It also has the attribute `dgnfog:type="shape"` attribute. If the prop is a light receiver (for dynamic light), its type is defined in the `dgnfog:lightreceiver` attribute. Possible values are:

* `none`: Ignored by dynamic lighting
* `flat`: Lit by dynamic light, with prop shadows shown
* `flatnoprop`: Always lit by dynamic light (props do not cast shadows onto it)

The shape tag also contains a `dgnfog:shadow` attribute with either `true` or `false` that indicates whether this prop casts a prop shadow.

If the prop is a light source, the group tag also contains a `<circle/>` tag with `dgnfog:type="light"`. This tag has the prop's center as x and y coordinates and the light radius as attribute `r`. It also uses the light's color as the fill color for the circle.

### Shape Props

All definitions from image props also apply for these, except that the shape is not a `<rect/>` but a `<path/>` with the same rules as the wall paths above.

Shapes can also not be light sources, and they don't have rotation (and so there's no `transform` attribute on the group).

### Token Props

Tokens do not participate in the dynamic light system, and so are much simpler:

Tokens contain a `<circle/>` attribute with `dgnfog:type="shape"` that contains the center of the token as the `cx`,`cy` position and the size as its radius.
