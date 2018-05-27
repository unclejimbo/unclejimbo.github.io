---
title: Some Notes on OpenGL Vertex Specification
date: 2017-08-13 11:34:09
categories:
- Computer Graphics
tags:
- Computer Graphics
- OpenGL
---

The idea of vertex specification is pretty simple, after you submit data to OpenGL, you need to tell OpenGL how to interpret those data. But I'm always quite confused by the names related to the vertex specification thing, for example,

- Vertex Array Object
- Vertex Buffer Object
- `GL_VERTEX_ARRAY_BUFFER`
- `glVertexAttribPointer`
- `glEnableVertexAttribArray`

What's the fuss about those funny names? It really needs more conceptual explanations before those words could make any sense.

<!--- more --->

# Vertex Attribute

Suppose you need vertex positions and vertex normals to correctly render an image. Thus your vertex shader might look something like

```glsl
(layout location = 0) in vec3 position;
(layout location = 1) in vec3 normal;

void main()
{
    //...
}
```

The vertex position and vertex normal are called **vertex attributes**, which OpenGL requires as input for your shader to correctly render a result. Actually, in the old days of OpenGL, the input specifier wasn't `in` but indeed `attribute`, so the shader should look like

```glsl
attribute vec3 position;
attribute vec3 normal;

void main()
{
    //...
}
``` 

Although `attribute` was deprecated since OpenGL 3.3 (as well as `varying`), this term still remains in the API, such as `glVertexAttribPointer` and `glEnableVertexAttribArray`. Now let's talk more about them.

## glVertexAttribPointer

As is often the case, we pack the vertex attribute data into **Vertex Buffer Objects** and stream the data to OpenGL. For the previous shader, there are several common ways to store the data.

- Into separate VBOs
    - VBO1 = P1 P2 P3 ...
    - VBO2 = N1 N2 N3 ...
- Concatenated
    - VBO = P1 P2 P3 ... N1 N2 N3 ...
- Interleaved
    - VBO = P1 N1 P2 N2 P3 N3 ...

In any case, you need to bind the correct vertex buffer object to the `GL_ARRAY_BUFFER` binding point and tell OpenGL how to interpret the data using `glVertexAttribPointer`. It actually works quite like a pointer, moving around the buffer to find the desired value for the specific attribute.

## glEnableVertexAttribArray

After the specification of one attribute, you need to enable this **vertex attribute array** for the shader to use the data. It's called this way because the vertex attribute data are indeed stored in an array. But why the hell do I need to enable this array? Because when you don't, OpenGL let you set a static vertex attribute value at that location using `glVertexAttrib*`. This is called the **non-array attribute** of OpenGL and is associated to an OpenGL context. Its job is pretty similar to an `uniform` and you might only want to use static vertex attribute in certain cases for the sake of efficiency. Otherwise, use `uniform` for attribute that remain constant across vertex and use vertex attribute array for the per-vertex data.

# Vertex Array Object

As describted in the previous section, the common work flow before you could draw with a shader is to first use `glBindBuffer` to bind a VBO to `GL_ARRAY_BUFFER`, then use `glVertexAttribPointer` to specify the vertex format and finally use `glEnableVertexAtrribArray` for this data to be operated. You repeat this procedure for every vertex attribute required by the shader and then you could draw with OpenGL. This seems quite cubersome, so OpenGL let you use a VAO to record those specifications then you don't need to repeat those procedures whenever you want to draw with the set of data. However, VAO does not record the data in the vbo, so changing the data will cause VAO to update automatically.

---
That's pretty much about it. Still these terms are quite similar to each other yet with different meanings and purposes. Try not to get confused. For more advanced usage of vertex specification, such as instancing, separate attribute format, e.g., please read the wiki page in the reference.

# Reference

<https://www.khronos.org/opengl/wiki/Vertex_Specification>
