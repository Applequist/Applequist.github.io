---
title: 'Geometry modelling.'
description: 'A study of ISO 19107.'
pubDate: 'Mar 3 2025'
---

In this post, we will look at geometry modelling: how do you describe 
the spatial characteristics (location, size, shape...) of geographical features
using coordinates and coordinates reference systems.

These are just notes for myself. You might still find them useful as a
summary of the ISO 19107.

## From point sets to geometric objects

So, let's say we have to describe the size and shape of countries so we can 
solve border dispute (is this land in country A or neighbouring country B?).

One way would be to have the **set** of all positions that are **inside** country A
and the **set** of all positions that are **inside*** country B. 

In case of a dispute where country A claims a plot of land for which we also have the set of all positions,
we could just check whether all positions of the plot of land are **all** included in the 
set of positions from country A and if so everyone can agree that country A rules over
the said plot of land.

Unfortunately, given the countinuous nature of space, it is impossible to exhaustively 
enumerate all positions of a spatial extent using coordinates except for point,

But coordinates and math provide us with tools to **model** spatial extent as set of positions 
We can start with point which represent a single position.
With 2 positions, we can define an oriented curve segment using interpolation (linear or other)
Patching curve segments together, we can define oriented curves.

Now that we have oriented curves, we can use a curve that closes on itself (a cycle) 
to delimit a surface as a subset of an existing surface by saying that all positions
that are to the left of the curve are **in** the delimited surface. 
If we don't have a *reference* surface, we could use 3 positions and interpolation to 
define a planar triangle, and for more complex surface we could pacth a bunch of triangles
together as we did for curves.

Finally using surfaces we can delimit bounded volumes of space.

So using finite representations using points, curves and surfaces we have been 
able to model or describe infinite set of points.
During this little *though experiment*:
- we have been able to model/describe infinite sets of points with finite representations using 
  points, curves and surfaces defined using coordinates and math,
- a few important concepts have emerged:
    - interior, i.e the set of position that are *in* a spatial extent,
    - boundary, i.e. the set of positions that delimit a spatial extent. 
    - interpolation, i.e. a way to mathematically describes a (infinite) set of positions from a finite 
      set of positions

The rest of this post goes over the different concepts and terms and define them more precisely.


## Glossary

boundary:
> **Set** that represents the limit of an entity.

closure:
> Union of the **interior** and **boudary** of a **topological** or **geometric object**.

connected:
> Property of a **geometric object** saying that between any 2 points in the object
there is **curve** that is entirelly contained in the object.

convex set: 
> A **geometric set** in which any 2 points are connected by a **straight line** segment 
entirelly contained in the object.

curve: 
> A 1-dimensional **geometric primitive** that is the image of an interval by a continuous
mapping.

cycle: 
> A **spatial object** without a **boundary**. Cycles are used to describe boundary components.
A cycle has no boundary because it closes on itself but it does not have infinite extent, eg
a circle (1-dimensional cycle) or a sphere (2-dimensional cycle).

function: 
> Mapping from elements of a domain to a unique element in another domain (co-domain or range).

geometric aggregate: 
> A collection of **geometric objects** that has no internal structure.

geometric boundary: 
> **Boundary** represented by a **set** of **geometric primitive** of smaller **geometric dimension**
that limits the extent of a **geometric object**.

geometric complex: 
> **Set** of disjoint **geometric primitives** (no point is interior to more than 1 geometric primitive)
where the **boundary** of each **geometric primitive** can be represented as the union of other **geometric primitives** 
of smaller dimension within the same **set**. A geometric complex is then closed under the boundary operations.

geometric dimension:
> Largest number n such that each point in a **geometric set** can be associated to a subset (i.e. neighbourhood)
that has the point in its interior ans is isomorphic to R^n. Eg each point on a curve can be associated to a subset 
of the curve containing the point and is isomorphic to R^1.

geometric object:
> A **spatial object** representing a **geometric set**.

geometric primitive:
> A **geometric object** representing a single, connected, homogeneous element of space.
>
> **Geometric primitives** include **points**, **curves**, **surfaces** and **solids**.

geometric set:
> **Set** of point in space.

interior:
> *Set* of all positions that are on/in a **geometric object** but which are not on/in its **boundary**.

point:
> 0-dimensional **geometric primitive**, representing a position in space.

ring:
> **Simple curve** which is a cycle. Rings are used to describe **boundary** components of surfaces in 2D and 3D CRS.

shell:
> **Simple surface** which is a cycle.

simple:
> Property of a **geometric object** that its **interior** is everywhere locally **isomorphic** to an open 
subset of of a Euclidian space of the appropriate dimension. I.e it has no self-intersection or self-tangency.

spatial object:
> **Object** used to represent the spatial characteristic of a feature.

