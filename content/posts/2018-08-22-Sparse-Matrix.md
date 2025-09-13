---
title: Sparse Matrix Manipulation in Libigl
toc: true
tags:
  - Computer Graphics
  - Geometry Processing
  - Libigl
  - Eigen
categories:
  - Computer Graphics
date: 2018-08-22 00:18:05
---

Here I've compiled a list of functions to manipulate `Eigen::SparseMatrix` in [Libigl](https://github.com/libigl/libigl), due to the lack of documentation.

Up-to-date with commit [1f5a0c1](https://github.com/libigl/libigl/commit/1f5a0c100a70ae487f673e3a76110cad4fb983d3).

<!-- more -->

## Creation

### [cat.h](https://github.com/libigl/libigl/blob/master/include/igl/cat.h)

Concatenate two `SparseMatrix`.

### [diag.h](https://github.com/libigl/libigl/blob/master/include/igl/diag.h)

Extract the diagonal entries of a  `SparseMatrix` into a `Vector` and vice-versa. This is superceded by

```cpp
VectorXd V = X.diagonal() and
SparseVector<double> V = X.diagonal().sparseView()
SparseMatrix<double> X = V.asDiagonal().sparseView()
```

### [invert_diag.h](https://github.com/libigl/libigl/blob/master/include/igl/invert_diag.h)

Invert the diagonal entries of a `SparseMatrix`.

### [repdiag.h](https://github.com/libigl/libigl/blob/master/include/igl/repdiag.h)

Repeat a `SparseMatrix` on the diagonal n times to form a new `SparseMatrix`.

### [repmat.h](https://github.com/libigl/libigl/blob/master/include/igl/repmat.h)

Repeat a `SparseMatrix` along the columns and rows to form a r by c `SparseMatrix`.

### [sparse.h](https://github.com/libigl/libigl/blob/master/include/igl/sparse.h)

Create a `SparseMatrix` from indices and values. It's a simple wrapper to the `Eigen::SparseMatrix::setFromTriplets` method.

### [sparse_cached.h](https://github.com/libigl/libigl/blob/master/include/igl/sparse_cached.h)

Faster version of `sparse` using cached data.

### [speye.h](https://github.com/libigl/libigl/blob/master/include/igl/speye.h)

Create a sparse identity matrix.

## Predicate

### [is_sparse.h](https://github.com/libigl/libigl/blob/master/include/igl/is_sparse.h)

Determine if a matrix is `SparseMatrx`.

### [is_symmetric.h](https://github.com/libigl/libigl/blob/master/include/igl/is_symmetric.h)

Determine if a `SparseMatrix` is symmetric.

### [is_diag.h](https://github.com/libigl/libigl/blob/master/include/igl/is_diag.h)

Determine if a `SparseMatrix` is diagonal.

## Coefficient-wise Operation

### [for_each](https://github.com/libigl/libigl/blob/master/include/igl/for_each.h)

Apply a function for each non-zero element. Could be replaced with [`Eigen::SparseMatrix::unaryExpr`](http://eigen.tuxfamily.org/dox/classEigen_1_1SparseMatrixBase.html#af9bed5dea96bdaf17ffd1a76ab0aedb1).

Note that `Eigen::SparseMatrix` also provides a lot of coefficient-wise operations to use.

## Reduction and Visitor

### [all.h](https://github.com/libigl/libigl/blob/master/include/igl/all.h)

Return true if all coefficients in a row/column of a `SparseMatrix` are true.

### [any.h](https://github.com/libigl/libigl/blob/master/include/igl/any.h)

Return true if any coefficient in a row/column of a `SparseMatrix` is true.

### [count.h](https://github.com/libigl/libigl/blob/master/include/igl/count.h)

Return the number of coefficients that are true in each row/column in a given `SparseMatrix`.

### [find.h](https://github.com/libigl/libigl/blob/master/include/igl/find.h)

Get the values and indices of non-zero elements in a `SparseMatrix`. It's a convinient wrapper to:

```cpp
SparseMatrix<double> mat(rows,cols);
for (int k=0; k<mat.outerSize(); ++k)
  for (SparseMatrix<double>::InnerIterator it(mat,k); it; ++it)
  {
    it.value();
    it.row();   // row index
    it.col();   // col index (here it is equal to k)
    it.index(); // inner index, here it is equal to it.row()
  }
```

### [find_zero.h](https://github.com/libigl/libigl/blob/master/include/igl/find_zero.h)

Return the first zero rows/columns of a `SparseMatrix`.

### [max.h](https://github.com/libigl/libigl/blob/master/include/igl/max.h)

Return the values and indices of the maximum elements of each row/column in a `SparseMatrix`.

### [min.h](https://github.com/libigl/libigl/blob/master/include/igl/min.h)

Return the values and indices of the minimum elements of each row/column in a `SparseMatrix`.

### [redux.h](https://github.com/libigl/libigl/blob/master/include/igl/redux.h)

Run a function to reduce the `SparseMatrix` colwise/rowwise.

### [sum.h](https://github.com/libigl/libigl/blob/master/include/igl/sum.h)

Return the colwise/rowwise sum of a `SparseMatrix`.

## Slicing

### [slice.h](https://github.com/libigl/libigl/blob/master/include/igl/slice.h)

Slice a `SparseMatrix` into a sub-matrix.

### [slice_cached.h](https://github.com/libigl/libigl/blob/master/include/igl/slice_cached.h)

Faster version of `slice` using cached data.

### [slice_into.h](https://github.com/libigl/libigl/blob/master/include/igl/slice_into.h)

Copy a `SparseMatrix` into a slice of another one.

### [slice_mask.h](https://github.com/libigl/libigl/blob/master/include/igl/slice_mask.h)

Slice using mask instead of indices.

## Decomposition

### [eigs.h](https://github.com/libigl/libigl/blob/master/include/igl/eigs.h)

Eigen value decomposition. See [Spectra](https://spectralib.org/) for a more sophisticated solution.

## Other Utilities

### [print_ijv.h](https://github.com/libigl/libigl/blob/master/include/igl/print_ijv.h)

Print the indices and values of non-zero entries.
