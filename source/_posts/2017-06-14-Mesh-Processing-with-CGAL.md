---
title: Mesh Processing with CGAL
date: 2017-06-14 14:24:34
categories:
- Computer Graphics
tags:
- Computer Graphics
- CGAL
- Geometry Processing
---

[CGAL](http://www.cgal.org/index.html), which stands for the Computational Geometry Algorithms Library, is an important tool set to use and build geometric algorithms. It contains a tons of packages with various functionalities, so this time I'm only gonna explore a little part of this beast and figure out the basics to do mesh processing with CGAL.

![](http://doc.cgal.org/latest/Surface_mesh_simplification/Illustration-Simplification-ALL.jpg)
(*Mesh smoothing algorithm, picture taken from the [CGAL document](http://doc.cgal.org/latest/Surface_mesh_simplification/index.html#Chapter_Triangulated_Surface_Mesh_Simplification)*).

<!-- more -->

# Preliminaries

First of all, I assume you already know the basics of mesh processing, especially the halfedge data structure, because all the related packages and algorithms in CGAL are implemented upon this data structure. If you don't, [here](https://www.openmesh.org/Daily-Builds/Doc/a00016.html) is a concise introduction or you can resort to the [PMP](http://www.pmp-book.org/) book for a more comprehensive treatment.

And I also assume you have installed CGAL properly and run through the [hello world](http://doc.cgal.org/latest/Manual/tutorial_hello_world.html) tutorial if you haven't used this library before.

Note that the stable release version of CGAL at the time of writing is 4.10.

# The Polyhedron_3 Package

The [3D Polyhedral Surface](http://doc.cgal.org/latest/Polyhedron/index.html#Chapter_3D_Polyhedral_Surfaces) package, or Polyhedron_3 as its name in the program, is a pointer-based implementation of the halfedge data structure released since the 1.0 version of CGAL.

## Creation

There are convinient functions to create a tetrahedron though they are not very useful.

```cpp
template<typename Traits >
Halfedge_handle CGAL::Polyhedron_3< Traits >::make_tetrahedron();

template<typename Traits >
Halfedge_handle CGAL::Polyhedron_3< Traits >::make_tetrahedron(
    const Point& p1,
    const Point& p2,
    const Point& p3,
    const Point& p4
);

```

You can also read from a file, which is what we usually need in practice. Although it only support very few common file formats like off and obj. You can also write out Polyhedron_3 to a file.

```cpp
#include <CGAL/IO/Polyhedron_iostream.h>

template <class PolyhedronTraits_3>
ostream& operator<<( ostream& out,
const CGAL::Polyhedron_3<PolyhedronTraits_3>& P);

template <class PolyhedronTraits_3>
istream& operator>>( istream& in,
CGAL::Polyhedron_3<PolyhedronTraits_3>& P);
```

When you need to read some other file formats not supported by this package, or to build the mesh procedurally, then you should use the incremental builder. The following example shows how to build up a Polyhedron_3 mesh from a list of veritices and indices.

```cpp
#include <CGAL/Simple_cartesian.h>
#include <CGAL/Polyhedron_3.h>
#include <CGAL/Polyhedron_incremental_builder_3.h>
#include <vector>

template<typename Polyhedron_3>
class TriMeshBuilder : public CGAL::Modifier_base<typename Polyhedron_3::HalfedgeDS>
{
    using FT = typename Polyhedron_3::Traits::Kernel::FT;
    using HDS = typename Polyhedron_3::HalfedgeDS;

public:
    TriMeshBuilder(const std::vector<FT>& vertices, const std::vector<unsigned>& indices)
        : _vertices(vertices), _indices(indices)
    {

    }
    void operator()(HDS& hds)
    {
        CGAL::Polyhedron_incremental_builder_3<HDS> builder(hds, true);
        builder.begin_surface(_vertices.size(), _indices.size() / 3);

        for (size_t i = 0; i < _vertices.size(); i += 3) {
            builder.add_vertex(HDS::Vertex::Point(
                _vertices[i], _vertices[i + 1], _vertices[i + 2]));
        }

        for (size_t i = 0; i < _indices.size(); ++i) {
            builder.begin_facet();
            builder.add_vertex_to_facet(_indices[i++]);
            builder.add_vertex_to_facet(_indices[i++]);
            builder.add_vertex_to_facet(_indices[i]);
            builder.end_facet();
        }

        builder.end_surface();
    }

private:
    std::vector<FT> _vertices;
    std::vector<unsigned> _indices;
};

using Kernel = CGAL::Simple_cartesian<float>;
using FT = Kernel::FT;
using Polyhedron_3 = CGAL::Polyhedron_3<Kernel>;
using HDS = Polyhedron_3::HalfedgeDS;

int main()
{
    std::vector<FT> vertices;
    std::vector<unsigned> indices;
    // Fill in the data...
    Polyhedron_3 P;
    TriMeshBuilder<HDS> builder(vertices, indices);
    P.delegate(builder);
    return 0;
}
```

## Navigation

The key feature of a halfedge data structure is its flexibility to navigate on the mesh.

There are iterators you can use to iterate through vertices, halfedges, edges and facets. Keep in mind that when you dereference a iterator you'll get the corresponding handle type. Their documentation is a bit of frustrating.

```cpp
Vertex_iterator vertices_begin();
Vertix_iterator vertices_end();
Facet_iterator facets_begin();
Facet_iterator facets_end();
// etc
```

There is also a special type of iterator called the circulator, who can iterate around a target primitive. Note that the below functions are accessed from a handle, and return a halfedge handle from which the desired primitive can be resolved.

```cpp
// Iterate around all the vertices of a facet,
Halfedge_around_facet_circulator facet_begin();
// Iterate around all the vertices incident of a vertex
Halfedge_around_vertex_circulator vertex_begin();
```

Let's see a concrete example to compute the valence of a mesh, I'm only showing the key part.
```cpp
int main()
{
    Polyhedron_3 P;
    for (Vertex_iterator vit = P.vertices_begin(); vit != P.vertices_end(); ++vit) {
        auto valence = 0;
        Halfedge_around_vertex_circulator vcir = vit->vertex_begin();
        do {
            ++valence;
        } while(++vcir != vit->vertex_begin());
    }
}
```

# The Surface_mesh Package

Since version 4.6, CGAL introduces a new [Surface Mesh](http://doc.cgal.org/latest/Surface_mesh/index.html#sectionSurfaceMeshUsage) package to replace the old Polyhedron_3 package. So why do we need another halfedge data structure? This [benchmark](http://imr.sandia.gov/papers/imr20/Sieger.pdf) shows that the Surface_mesh package achieves better runtime speed with less memory footprints. The reason behind this achievement is because this new package uses an index-based approach to store the mesh, rather than the pointer-based approach taken by Polyhedron_3. So it is recommended to use this package as your first choice. Although at the time of writing there are a few packages in CGAL who still don't support Surface_mesh, so you may want to resort to Polyhedron_3 in such a situation.

The official examples and documentations for this package are much more well-organized. With the experience you already gained from Polyhedron_3, I'm sure you could head directly to the manual and learn how to use this package by yourself.

# Let's Be More Generic

From this point on, you already have enough ground to use the mesh data structures. So you could head towards to the specific mesh processing algorithm you need to use. There are quite a few packages out there, including Polygon_mesh_processing, Subdivision, Segmentation, just to name a few. But if you need to use the halfedge data structures provided by CGAL and implement your own mesh processing algorithms on top of that, then you might want to learn more on how to do this effectively.

If you go to check those algorithmic packages and you'll find out that most of them support both Polyhedron_3 and Surface_mesh using the same interface. How to achieve that? Do we have to manually convert one data structure to another? Or do we have to specialize every algorithm for these two data structures? The way CGAL deals with different halfedge implementations is through concepts and traits.

Manifold surface meshes are in essence graphs, so CGAL defines several [graph concepts](http://doc.cgal.org/latest/BGL/group__PkgBGLConcepts.html) to abstract away the behaviour a halfedge data structure should abide. By parameterizing the mesh data type using template and programming against the graph concepts, any implementation that meets the definition of those concepts can be used by the algorithm.

In order to transform the interfaces of both data structures to conform to the interface required by the graph concepts, CGAL defines a lot of trait classes, which all reside in the "CGAL/boost/graph" directory. Except for Polyhedron_3 and Surface_mesh, a lot of other data structures in CGAL could also use traits to conform to the graph concepts so as to be used by the algorithms. There is even a trait class for [OpenMesh](https://www.openmesh.org/), which is also a popular choice of halfedge implementation, defined in the file "graph_traits_TriMesh_ArrayKernelT.h". In other words, all the algorithms written against the graph concepts could also operate on OpenMesh.

That's how generic programming saves the world. In fact, the graph concepts defined by CGAL actually conform to the API defined by [BGL](http://www.boost.org/doc/libs/1_64_0/libs/graph/doc/), a general graph library, so some basic graph algorithms like shortest path or minimum spanning tree could also operate on Polyhedron_3 and Surface_mesh. As a result, as long as you write your own algorithm using the interface defined by the graph concepts, a lot of data structures could be used with only one implementation. Awesome.

## Let's Be More Concrete

Now I'll show you one concrete example on how this thing works. Why don't we just checkout the mesh fairing function from the [Polygon Mesh Processing](http://doc.cgal.org/latest/Polygon_mesh_processing/group__PMP__meshing__grp.html#gaa091c8368920920eed87784107d68ecf) package.

```cpp
template<typename TriangleMesh, typename VertexRange, typename NamedParameters>
bool CGAL::Polygon_mesh_processing::fair(
    TriangleMesh& tmesh,
    const VertexRange& vertices,
    const NamedParameters& np
)
```

This function takes in a mesh and range of vertices you want to fair, and smoothes out the mesh as much as possible. The np parameter is not important here. So how does CGAL make this function work on all the data structure? We should look into the source code in file "CGAL\Polygon_mesh_processing\internal\fair_impl.h" to see what happens.

We could see that the algorithm is implemented with the graph concept api rather than any class-specific interface.

```cpp
//...
typedef typename boost::graph_traits<PolygonMesh>::vertex_descriptor vertex_descriptor;
//...
vertex_descriptor nv = target(opposite(*circ,pmesh),pmesh);
//...
```

And the real implementations for these apis are actually template specializations for each known data structure defined in the trait classes I mentioned earlier. You could check the implementation of Polyhedron_3 for example in "CGAL/boost/graph/graph_traits_Polyhedron_3.h".

```cpp
//...
template<class Gt, class I, CGAL_HDS_PARAM_, class A>
struct graph_traits< CGAL::Polyhedron_3<Gt,I,HDS,A> >
   : CGAL::HDS_graph_traits< CGAL::Polyhedron_3<Gt,I,HDS,A> >
{
  typedef typename Gt::Point_3 vertex_property_type;
};
//...
template<class Gt, class I, CGAL_HDS_PARAM_, class A>
typename boost::graph_traits< CGAL::Polyhedron_3<Gt,I,HDS,A> const>::vertex_descriptor
target(typename boost::graph_traits< CGAL::Polyhedron_3<Gt,I,HDS,A> const>::edge_descriptor e
       , const CGAL::Polyhedron_3<Gt,I,HDS,A> & )
{
  return e.halfedge()->vertex();
}
//...
template<class Gt, class I, CGAL_HDS_PARAM_, class A>
typename boost::graph_traits< CGAL::Polyhedron_3<Gt,I,HDS,A> >::halfedge_descriptor
opposite(typename boost::graph_traits< CGAL::Polyhedron_3<Gt,I,HDS,A> >::halfedge_descriptor h
         , const CGAL::Polyhedron_3<Gt,I,HDS,A>&)
{
  return h->opposite();
}
//...
```

In this way, the compiler will automatically use the right specialization for a data structure to use. This is often called static dispatch.

# Conclusion

I hope that I have sorted out some of the mess about using this library. You may also be interested in reading the [BGL package](http://doc.cgal.org/latest/BGL/index.html#Chapter_CGAL_and_the_Boost_Graph_Library) and [Boost PropertyMap package](http://doc.cgal.org/latest/Property_map/index.html#Chapter_CGAL_and_Boost_Property_Maps) for further information.
