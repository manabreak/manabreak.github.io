---
layout: post
title:  "Legacy of Satos Devlog #2 - Data-Driven Approach"
date:   2019-09-28 19:17:51 +0300
categories: devlog satos 3d
---
Legacy of Satos is heavily data-driven under the hood.
Almost everything from levels to game objects and from interactions
to animations are data-driven. Let's take a look how it works!

# ECS Primer

First off, a quick primer on the game's architecture.
The base architecture follows ECS pattern, which comes from
words 'entity', 'component' and 'system'. In addition,
there is the 'game world', which contains all entities,
components and systems and handles the connections between
them.

**Components** are basically "pure data" classes and contain
no logic at all (except some helper methods). Examples include
SpriteComponent, which stores information about what texture
should be used for the sprite and what are the dimensions
for the sprite.

**Entities** are basically just identifiers of, well, entities.
Each entity is represented by an integer. By itself, an entity
is rather useless, as it can't really do anything else than
exist. This is where components join the fun: Entities can have
components assigned to them. In practice, each entity is 
a bag of components.

**Systems** bind everything together. A system operates on entities
that have certain components. For instance, a sprite system
only updates the entities that have both a `Sprite` component
and a `Transform` component, so it can draw the sprite from the
`Sprite` component at the correct location.

Since all the data is contained in components, ECS lends itself
really well into data-driven design. However, before loading any
entities, there should be something that tells the engine *what*
to load.

# Level Data

For the sole purpose of creating levels for Legacy of Satos,
I wrote Ratio, which is a octree-based voxel editor with tilemapping
capabilities. It also supports Tiled-style objects with custom
properties, as well as level-specific properties.

The level geometry is a collection of layers, which are essentially
collections of octrees. For each layer, there is at least one
octree per tileset used on that layer, and if the octree gets large
enough (vertex count wise), another octree is added.

The level data thus looks like this in JSON:

{% highlight json %}

{
    "tilesets": [
        "name": "dungeon",
        "type": "normal",
        "path": "tilesets/dungeon.png",
        "tiles": [
            {
                "id": 1,
                "x": 0,
                "y": 0,
                "w": 16,
                "h": 16
            },
            {
                "id": 10,
                "x": 0,
                "y": 16,
                "w": 16,
                "h": 16
            }
        ]
    ],
    "properties": [
        {
            "key": "ambient_intensity",
            "type": "double",
            "value": 0.3
        }
    ],
    "layers": [
        {
            "name": "First layer",
            "visible": true,
            "parts": [
                {
                    "tileset": 0,
                    "size": 2048,
                    "cells": [
                        {
                            "x": 16,
                            "y": 16,
                            "z": 64,
                            "size": 16,
                            "floor": false,
                            "faces": [10, 1, 1, 10, 1, 1]
                        }
                    ]
                }
            ]
        }
    ],
    "objects": [
        {
            "name": "player",
            "type": "enter",
            "location": [64.0, 32.0, 32.0],
            "size": [16.0, 16.0, 16.0],
            "color": [1.0, 1.0, 1.0, 1.0],
            "visible": true,
            "properties": [
                {
                    "key": "id",
                    "type": "int",
                    "value": 0
                }
            ]
        },
        {
            "name": "slime",
            "type": "object",
            "location": [64.0, 32.0, 64.0],
            "size": [16.0, 16.0, 16.0],
            "color": [1.0, 1.0, 1.0, 1.0],
            "visible": true,
            "properties": [
                {
                    "key": "class",
                    "type": "string",
                    "value": "slime"
                }
            ]
        }
    ]
}

{% endhighlight %}

That's a handful of stuff there. Let's look at each of the
top-level parts individually.

**Tilesets** portion defines all the tilesets used for the
level. Each tileset has a name, type (either "normal" which
means an image-based tileset or "palette" which means pre-defined
colors), path to the image file (if not a palette), and the tile
definitions.

Each **tile** has a unique ID inside that tileset, the X and Y coordinates
of the region in the image, and width and height of the region. These
tile definitions are created only for the tiles that are actually used
in the level to keep the amount of data as small as possible.

**Properties** defines the level-specific properties. In the example,
the ambient light intensity is set to 0.3. These keys should pretty much
match the values the game engine expects, since the engine doesn't (yet)
support checking against level properties during the gameplay. That
wouldn't be a big thing to implement, though, but it's just something
I haven't needed yet.

**Layers** is an array of the level data layers. Each layer has a name
and an array of parts. These parts contain the octree data and basically
just tell A) which tileset to use, B) how large the layer is, and C)
where the cells are.

Each **cell** inside the octree has a coordinate, size, and "faces" info,
which defines which tile ID should map to which face of the voxel.
The array order is hardcoded to order [front, back, right, left, top, bottom]
(when viewed down the negative Z-axis, X-axis being on right and Y pointing upwards).

**Objects** contains the definitions of all the custom objects in the level.
Each object has a name and a type, as well as location, size and color.
Each object can also have custom properties in similar vein as the level itself.

Now, when objects are defined here, I don't go defining each and every parameter
for all the objects here. Instead, the type is quite often "object", and 
the "class" property tells which kind of object should be created in its place.
So, when the map is loaded, it will see that I should load an object of type "slime"
here. But where does that "slime" object come from? What does it look like?

# Data-driven Entities

In Legacy of Satos, the game objects are also defined in JSON.
Each game object is basically just a set of components and
their properties. An example entity JSON might look like this:

{% highlight json %}

{
    "name": "slime",
    "components": [
        {
            "type": "sprite",
            "properties": {
                "region": "slime",
                "width": 16,
                "height": 16,
                "billboard": "Y"
            }
        },
        {
            "type": "transform"
        },
        {
            "type": "animation",
            "properties": {
                "animations": "slime"
            }
        },
        {
            "type": "health",
            "properties": {
                "hp": 5,
                "max": 5
            }
        },
        {
            "type": "damageOnHit",
            "properties": {
                "min": 1,
                "max": 2
            }
        }
    ]
}

{% endhighlight %}

This is rather straight-forward. The **components** array tells
all the components the slime should have. For instance, this
game object consists of a sprite, a transform, an animation,
a health and a "damage on hit" component. Each can override any
and all of the properties those components have.

So, in the case of this "slime" object, it will create a sprite 
component, set its image to "slime", set its size to 16 by 16,
and tell the engine that the sprite should be billboarded around Y-axis.
Nothing extraordinary. However, the one special case here is
the animation component, which has a property named "animations".
This property refers to yet another JSON, which defines all the
available sprite animations for this slime.

Introducing yet another tool: Spritetools!

# Animations

Spritetools is the most recent addition to my arsenal of 
never-ending development. It allows me to draw the sprites
with GIMP while hot-reloading the changes. This way, I can
immediately see how the animation looks like and can tweak
things like per-frame animation speeds, axis flips and so on.
When ready, the animations can be exported for the game.

So, the game object JSON above referred to animation called "slime".
This animation could look like this:

{% highlight json %}

{
    "image": "slime",
    "animations": [
        {
            "name": "idle",
            "row": 0,
            "constant_speed": false,
            "speeds": [0.25, 0.13, 0.09],
            "frame_width": 16,
            "frame_height": 16,
            "frames": 3,
            "flip_x": false,
            "flip_y": false,
            "animate": true,
            "mode": "loop",
            "forward": true
        }
    ]
}

{% endhighlight %}

Here, the animation file refers to an image called "slime",
which is just either a texture image or a region in a texture atlas.
The "animations" array contains all the different animations.

Each animation entry has a name which can be referred by other
components. The other parameters define how the animation is built,
for example which "row" in the image is used, what are the per-frame
speeds (or if constant speed should be used), the size of a single
frame, frame count, axis flips and so on.

The "mode" property can have one of three different values:
it can either be "loop", which means it'll play the animation constantly,
starting from the first frame when the end is reached. Another value
is "oneshot", which means the animation is played once and then stopped.
The third option is "pingpong", which reverses the direction of the
animation once the end is reached, and is again reversed once the animation
gets to the first frame.

Now, this animation structure works for the simplest stuff there is,
but how does this lend itself to the fact that the sprites are essentially
billboards, always facing the camera AND the camera rotating around the player
constantly?

# Animations with directional variants

For example, an issue arises when there is, say, an NPC that can walk to
four directions (north, south, east, west). Each of these walk directions
has its own animation ("walk with back visible", "walk with chest visible",
"walk with side visible" and "walk with side visible, Y-flipped"). These
would work well *if* the camera was always pointing to the same direction.
However, this is not the case in Legacy of Satos.

If the NPC is walking south and camera is facing north, we should see the
character's face. Now, what if camera rotates to face east? Now we should
see the character's side, just like if it was walking east. It's quite
a hassle to change between animations whenever the camera rotates.

The camera rotation issue is tackled by having a kind of two-dimensional
approach to the animation definitions. First off, there is the animation
definition like "walk", but the directions are grouped under that main
animation definition. So, when the old structure was like this:

{% highlight json %}
[
    {
        "name": "walk_south",
        ...
    },
    {
        "name": "walk_north",
        ...
    },
    {
        "name": "walk_east",
        ...
    },
    {
        "name": "walk_west"
    }
]
{% endhighlight %}

... the new structure becomes something like this:

{% highlight json %}
[
    {
        "name": "walk",
        "north": {
            ...
        },
        "south": {
            ...
        },
        "east": {
            ...
        },
        "west": {
            ...
        }
    }
]
{% endhighlight %}

Now, the characters have a direction they are facing (globally),
and the camera has its own direction (globally). The relative direction
of the character (relative to the camera) is quite easy to obtain,
so whenever the camera rotates, it will simply change the character's
relative direction and the "direction variant" of the animation.
All the common variables for certain animation like frame speeds,
frame count and so on are common for all the direction variants
under any given animation - only the varying properties such as
the starting row and column change.

In addition, some animations might have only one or two directional
variants. In the case where there's only one used, the whole
directional thing is omitted - the sprite will always have the same
look no matter where the camera is pointing. In the case where
the animation has two variants, one will be used when the camera
points either north or south, and the other is used when the camera
points either east or west. Optionally, X and Y flipping can be done
here as well. For example, a tree might be implemented by having
two directional variants, and they are flipped around Y axis
whenever the direction is either south or west, and not flipped
when the direction is either north or east.

# Recap, please!

So, basically the flow of starting the game goes like this:

1. Initialize all the systems in the game engine
2. Load the level defined as the first level to load
3. Level loads...
    - Geometry is generated from the octree data
    - Object definitions are loaded
4. Create game objects
    - For each game object in the level data:
        - Load the required game object definition
        - Create an entity instance
        - Create component instances as per the definition
        - Populate the component properties as per the definitions
        - If the game object in the level data has overrides for properties, set those now
        - Create child entities if needed
5. Initialize animations
    - For each required animation, load the definition
    - Set the initial animations for the game objects
6. Initialize sounds and other assets
7. ?
8. Profit!
        
This looks rather simple, but there's a ton of special cases
and component-specific exceptions going on in the game engine,
as some components require the engine to "know" a bit more about
the situation than others. There are also some object types
that are handled differently, and some may bypass the normal
loading flow altogether.

This was a rather lengthy rundown of a rather complex system,
and I've only barely scratched the surface of how deep it actually
goes! Hopefully, I'll get some extra time to write more about this,
since honestly I'm quite enthusiastic about this whole sub-system
of the game.