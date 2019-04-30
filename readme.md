# BLOSPRAY - OSPRay as a Blender render engine

...

Note that this project is developed for Blender 2.8x, as that
will become the new stable version in a couple of months.
Blender 2.7x is not supported at the moment and probably won't be,
as the Python API isn't fully compatible with 2.8x.

## Usage

BLOSPRAY is a Blender add-on, but is not being distributed separately
as it is in heavy development.
Currently, the way to install it is to clone this repository and then
make a symlink to the `render_ospray` directory in the Blender addon directory:

```
$ cd ..../blender-2.8/2.80/scripts/addons
$ ln -sf <blospray-repo>/render_ospray render_ospray
```

## Goals and non-goals

The focus of BLOSPRAY is to provide rendering of scientific
scenes in Blender, with a specific focus on volumetric data. XXX other OSPRAY features

Goals:

- Provide rendering through OSPRAY for scientific datasets

Non-goals:

- Compete with Cycles. Cycles has focus on production rendering of animation,
  artistic, etc.
    

## Features

### Plugins

There is a rudimentary plugin system that can be used to set up
custom scene elements in OSPRay directly. This is especially useful when working with large scientific datasets for which it is infeasible or unattractive to load into Blender. Instead, one can use a proxy object, such as a cube mesh, and attach a plugin to it. During rendering BLOSPRAY will call the plugin to load the actual scene data associated with the proxy. In this way Blender scene creation, such as camera animation or lighting, can be done as usual as the proxy object shows the bounding box of the data and can even be transformed.

The original use case for plugins (and even BLOSPRAY itself) was
to make it easy to use volume rendering in a Blender scene. Blender's
own volume rendering support is geared towards the built-in 
smoke and fire simulations and isn't really a good fit for
scientific datasets, plus it is very hard to get volume data
into Blender anyway for that use case. 

Apart from volumes, plugins can be used for other types of scene contact as well, like polygonal geometry.
But this is currently not supported yet in BLOSPRAY.

Note that BLOSPRAY plugins are different from OSPRay's own "extensions", that
are also loadable at run-time. The latter are meant for extending OSPRay itself
with, for example, a new geometry type. BLOSPRAY plugins serve to extend *Blender*
with new scene elements that can then be rendered in OSPRay.

### Render server

BLOSPRAY consists of two parts:

1. a Python addon (directory `render_ospray`) that implements the Blender render engine. It handles scene export, showing the rendered result, etc.
2. a render server (`ospray_render_server`) that receives the scene from Blender, calls OSPRay routines to do the actual rendering and sends back the image result.

The original reason for this two-part setup is that there currently is no Python API for OSPRay, so direct integration in Blender is not straightforward. Neither is there a command-line OSPRay utility that takes as input a scene description in some format and outputs a rendered image.

Plus, the client-server setup also has some advantages:

- The separate render server can be run on a remote system, for example an HPC system that holds a large scientific dataset to be rendered. This offloads most of the compute-intensive rendering workload and necessity to hold data to be rendered locally away from the system running Blender.
- It should be feasible to use OSPRay's [Parallel Rendering with MPI](http://www.ospray.org/documentation.html#parallel-rendering-with-mpi) mode, by providing a variant of the render server as an MPI program.
- BLOSPRAY development becomes slightly easier as both Blender and the render server can be independently restarted in case of crashes or bugs.
- The network protocol is (currently) not strongly tied to Blender, so the render server can be used in other situations as well.

Of course, this client-server setup does introduce some overhead, in terms of network latency and data (de)serialization. But in practice this overhead is small compared to actual render times. 
In future, caching of data on the server can help in reducing the overhead even further.

Note that the render server currently doesn't support multiple different Blender scenes,
or even different users, at the same time. 

## Supported elements

| Meshes | |
| ------ |-|
| Normals           | :heavy_check_mark: |
| Vertex colors     | :heavy_check_mark: |
| Modifiers         | :heavy_check_mark: |
| Materials         | :x: |
| Textures          | :x: |
| UI panels | :x: |

| Volumes | | |
| ------- |-|-|
| Isosurfaces | :heavy_check_mark: | Support through custom property |
| Slice planes | :o: | Support for 1 plane through custom property |
| AMR | :o: | No specific support, atm, but plugins can create AMR volumes |
| Transfer functions | :x: | No UI yet |
| UI panels | :x: |

| Lights | |
| ------ |-|
| Point | :heavy_check_mark: |
| Sun | :heavy_check_mark: |
| Spot | :heavy_check_mark: |
| Area | :heavy_check_mark: |
| HDRI | :x: |
| UI panels | :x: |

| Cameras | | |
| ------- |-|-|
| Perspective | :heavy_check_mark: | |
| Orthographic | :x: | |
| DOF | :heavy_check_mark: | |
| UI panels | :x: | |

| Render settings | |
| --------------- |-|
| Resolution | :heavy_check_mark: |
| Percentage | :heavy_check_mark: |
| UI panels | :o: |


## Notes on volumetric dataset usage

XXX These notes are work-in-progress.

Within OSPray there's three levels at which volumetric data and its rendering 
is handled:

1. The raw volume data. In OSPRay these are stored in data buffers creating using `ospNewData()`.
   These buffers only contain basic properties relating to the data, such as the type
   of value (e.g. `float`), number of values (but not volume dimension) and data sharing flags.
   
2. The volume object, i.e. an `OSPVolume`. This object references the data buffer
   holding the raw volume data and also holds volume-specific properties. These
   properties relate both to the volume itself (e.g. grid spacing and data range), 
   but also to rendering the volume (e.g. sampling rate to use and the
   transfer function to apply). An `OSPVolume` can directly be added to an
   OSPRay scene to have the volume get rendered in the output image.

3. Geometry derived from volumetric data, specifically isosurfaces and slice planes.
   In OSPRay these two types of derived geometry are rendered implicitly based on
   ray traversal of the volumetric data, so no explicit isosurface or slice plane geometry
   is created. As such, these geometry objects contain a reference to a volume
   object but are of type `OSPGeometry` themselves. These can also be added
   to the OSPRay scene, independent from the addition of the `OSPVolume` object.
   
When working with volumetric objects in Blender it makes sense to have some
flexibility in handling these three levels, mostly to allow sharing
of data were possible. For example, one might want to load a volumetric dataset
and extract multiple isosurfaces that are shown side-by-side. Or one might 
want to use both a volume rendered representation, as well as a slice plane, 
based on the same volumetric data. 

As two of the levels refer to another level sharing is conceptually easy, but in 
practice a bit suboptimal. This has to do with the embedding of the three levels
within Blender in a way that fits Blender's concepts and workflow. Blender itself
only uses two levels: "objects" which refer to "object data" [^1]. Here, the object holds general
information, such as the 3D transformation to apply, while the object data holds the
per-type data. For example, mesh object data holds the 3D geometry and has a reference
to the material to use. As such, the mesh data is the natural place to 
query data on the 3D extent of a 3D surface, while the object referencing
the mesh data determines where in the 3D scene the mesh is located based on
the object's transformation.

Matching Blender's two levels to OSPRay's three is somewhat cumbersome, but we
can try to come up with some guiding principles and corresponding design choices:

- Reusing a volume object (level 2) is probably more common than reusing
  raw volume data (level 1). 
  

- Having a 3D representation of a volume in the scene is a necessity. This representation
can be as simple as a cube mesh for rectangular domains or a more complex shape
for unstructured volumetric grids. The most natural 3D object to use for this is
a Blender mesh. 

- Transformations of volumes should use existing 3D concepts and interaction 
(e.g. translate, scale), and so must work on objects, not object data.

- Re-using data is best done with Blender's existing method of object
instancing, in which two objects refer to the same object data. This allows 
the two objects to use independent transformations. This implies that 
volume-specific information (level 2) should be stored with the object
data (i.e. mesh as determined above).

Taken all together:

- We use a Blender cube mesh object to add an OSPRay volume/isosurface/slice plane to a scene
- At the object level a choice is made wether to render the volume directly,
  or show it as an isosurface or slice plane
- The size of the cube mesh is adjusted to match the extent of the underlying volume
- The object transformation is used to place the volume in the scene

But there are some ugly consequences of this scheme:

- To have both a volume representation as well as (say) an isosurface representation
of the same volumetric data involves having two objects (one volume rendered, one
isosurface rendered) refer to the same object data. But this implies that setting
the type of representation is done *at the object level*, which is counterintuitive 
in Blender as the representation is normally based on the type of object data linked 
to an object and not dependent on any object-level settings.

- Volumes currently don't support an arbitrary affine transformation in OSPRay.
Only for a structured grid is there a way to influence volume size and placement (but
not orientation) at the `OSPVolume` level. But this requires passing transformation
data from the (Blender) object level to the mesh level. Furthermore, pushing the transformation
down to the mesh level makes it impossible to share the volume in multiple objects,
as the mesh can only have one transformation.

- There's no natural place to specify the raw volume data to use. Setting this
at the mesh level does not allow sharing XXX.
Using an Empty and setting custom properties on it to specify the raw data
might be a workaround. But our current plugin API expects plugins to 

- Loading of volume data and construction of a `OSPVolume` are handled by
plugins, as Blender does not have any methods to offer for this. But object-level
transformation data needs to be available to the plugin if it needs to 
transform the volume. So the xform either needs to be passed (duplicate) with
the object data and object itself, OR the object needs to be transmitted
(including object data it references) before object data. Which means more
bookkeeping on the server. For geometry we use 1) pass mesh, 2) pass object referencing
that mesh.



There is actually a fourth level, which is to use volumetric data to color
geometry that it is located within the volume's extent, i.e. a form of
volumetric texturing. This is currently not supported in BLOSPRAY.

[^1]: Note that parenting could be used to provide more levels, but this doesn't
solve the actual problem.

XXX blender object nodes will make all of this much easier

XXX povray addon DOES add custom types (but uses existing blender objects
to represent the custom povray types)

## Known limitations and bugs

* The addon provides some UI panels to set OSPRay specific settings, but in other cases we use Blender's [custom properties](https://docs.blender.org/manual/en/dev/data_system/custom_properties.html)
  to pass information to OSPRay. These can even be animated, with certain limitations, but are not a long-term solution. Note also that some builtin UI panels are disabled when the render engine
  is set to OSPRay as those panels can't directly be used with OSPRay (e.g. they contain Cycles-specific settings).

* Scene management on the render server is non-existent. I.e. memory usage increases after each render.  

* Caching of scene data on the server, especially for large data loaded by plugins, is planned to be implemented, but isn't there yet. I.e. currently all scene data is re-sent when rendering a new image.

* Only a single (hard-coded) transfer function for volume rendering is supported

* Only final renders (i.e. using the F12 key) are supported. Preview rendering is technically feasible, but is not implemented yet.

* Error handling isn't very good yet, causing a lockup in the Blender script in case the BLOSPRAY server does something wrong (like crash ;-))

* BLOSPRAY is only being developed on Linux at the moment, on other platforms it might only work after some tweaks

* Command-line (batch) rendering isn't supported in a nice way yet, as the lifetime of the BLOSPRAY server needs to be managed manually.

* Volumes can only use point-based values, so no cell-based values

* All meshes are converted to triangle meshes before being passed to OSPRay


OSPRay itself also has some limitations, some of which we can work around, some of which we can't:

* Only the scivis renderer supports volume rendering, the pathtrace renderer currently does not

* In OSPray structured grid volumes currently cannot be transformed with an 
  arbitrary affine transformation (see [this issue](https://github.com/ospray/ospray/issues/159)
  and [this issue](https://github.com/ospray/ospray/issues/48)). 
  We work around this limitation in two ways in the `volume_raw` plugin:
  
  - Custom properties `grid_spacing` and `grid_origin` can be set to influence the
    scaling and placement of a structured volume (these get passed to the `gridOrigin`
    and `gridSpacing` values of an `OSPVolume`). See **tests/grid.blend** for an example.
    
  - By setting a custom property `unstructured` to `1` on the volume object 
    the structured grid is converted to an unstructured one, whose vertices *can* be transformed. 
    Of course, this increases memory usage and decreases render performance, but makes 
    it possible to handle volumes in the same general was as other 
    Blender scene objects. See **tests/test.blend** for an example.
    
* Volumes are limited in their size, due to the relevant ISPC-based
  code being built in 32-bit mode. See [this issue](https://github.com/ospray/ospray/issues/239).
  Development work on OSPRay is currently underway to turn the ISPC-based
  parts into regular C++, which should lift this restriction.
  
* Unstructured volumes can only contain float values (i.e. not integers).
  See [here](https://github.com/ospray/ospray/issues/289)
  
* The OSPRay orthographic camera does not support depth-of-field
  
* Blender supports multiple colors per vertex (basically one per face loop),
  while OSPRay only supports a single value per vertex. XXX need to double-check this
  
* OSPRay does not support motion blur

* All rendering is done on the CPU, because, well ... it's OSPRay ;-)


## Dependencies

For building:

* OpenImageIO
* Google protobuf (C/C++ libraries)

* Uses https://github.com/nlohmann/json/blob/develop/single_include/nlohmann/json.hpp
  (included in the sources)

For running:

* Numpy
* Google protobuf (Python modules)

  To make Blender find the necessary protobuf packages add symlinks to
  `google` and `six.py` in Blender's python library dir:

  ```
  $ cd ~/blender-2.80-...../2.80/python/lib/python3.7
  $ ln -sf /usr/lib/python3.7/site-packages/six.py six.py
  $ ln -sf /usr/lib/python3.7/site-packages/google google
  ```

