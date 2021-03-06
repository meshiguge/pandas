.. currentmodule:: pandas
.. _sparse:

.. ipython:: python
   :suppress:

   import numpy as np
   np.random.seed(123456)
   from pandas import *
   import pandas.util.testing as tm
   randn = np.random.randn
   np.set_printoptions(precision=4, suppress=True)
   import matplotlib.pyplot as plt
   plt.close('all')
   options.display.mpl_style='default'
   options.display.max_rows = 15

**********************
Sparse data structures
**********************

We have implemented "sparse" versions of Series, DataFrame, and Panel. These
are not sparse in the typical "mostly 0". You can view these objects as being
"compressed" where any data matching a specific value (NaN/missing by default,
though any value can be chosen) is omitted. A special ``SparseIndex`` object
tracks where data has been "sparsified". This will make much more sense in an
example. All of the standard pandas data structures have a ``to_sparse``
method:

.. ipython:: python

   ts = Series(randn(10))
   ts[2:-2] = np.nan
   sts = ts.to_sparse()
   sts

The ``to_sparse`` method takes a ``kind`` argument (for the sparse index, see
below) and a ``fill_value``. So if we had a mostly zero Series, we could
convert it to sparse with ``fill_value=0``:

.. ipython:: python

   ts.fillna(0).to_sparse(fill_value=0)

The sparse objects exist for memory efficiency reasons. Suppose you had a
large, mostly NA DataFrame:

.. ipython:: python

   df = DataFrame(randn(10000, 4))
   df.ix[:9998] = np.nan
   sdf = df.to_sparse()
   sdf
   sdf.density

As you can see, the density (% of values that have not been "compressed") is
extremely low. This sparse object takes up much less memory on disk (pickled)
and in the Python interpreter. Functionally, their behavior should be nearly
identical to their dense counterparts.

Any sparse object can be converted back to the standard dense form by calling
``to_dense``:

.. ipython:: python

   sts.to_dense()

.. _sparse.array:

SparseArray
-----------

``SparseArray`` is the base layer for all of the sparse indexed data
structures. It is a 1-dimensional ndarray-like object storing only values
distinct from the ``fill_value``:

.. ipython:: python

   arr = np.random.randn(10)
   arr[2:5] = np.nan; arr[7:8] = np.nan
   sparr = SparseArray(arr)
   sparr

Like the indexed objects (SparseSeries, SparseDataFrame, SparsePanel), a
``SparseArray`` can be converted back to a regular ndarray by calling
``to_dense``:

.. ipython:: python

   sparr.to_dense()

.. _sparse.list:

SparseList
----------

``SparseList`` is a list-like data structure for managing a dynamic collection
of SparseArrays. To create one, simply call the ``SparseList`` constructor with
a ``fill_value`` (defaulting to ``NaN``):

.. ipython:: python

   spl = SparseList()
   spl

The two important methods are ``append`` and ``to_array``. ``append`` can
accept scalar values or any 1-dimensional sequence:

.. ipython:: python
   :suppress:

.. ipython:: python

   from numpy import nan
   spl.append(np.array([1., nan, nan, 2., 3.]))
   spl.append(5)
   spl.append(sparr)
   spl

As you can see, all of the contents are stored internally as a list of
memory-efficient ``SparseArray`` objects. Once you've accumulated all of the
data, you can call ``to_array`` to get a single ``SparseArray`` with all the
data:

.. ipython:: python

   spl.to_array()

SparseIndex objects
-------------------

Two kinds of ``SparseIndex`` are implemented, ``block`` and ``integer``. We
recommend using ``block`` as it's more memory efficient. The ``integer`` format
keeps an arrays of all of the locations where the data are not equal to the
fill value. The ``block`` format tracks only the locations and sizes of blocks
of data.

.. _sparse.scipysparse:

Interaction with scipy.sparse
-----------------------------

Experimental api to transform between sparse pandas and scipy.sparse structures.

A :meth:`SparseSeries.to_coo` method is implemented for transforming a ``SparseSeries`` indexed by a ``MultiIndex`` to a ``scipy.sparse.coo_matrix``.

The method requires a ``MultiIndex`` with two or more levels.

.. ipython:: python
   :suppress:


.. ipython:: python

   from numpy import nan
   s = Series([3.0, nan, 1.0, 3.0, nan, nan])
   s.index = MultiIndex.from_tuples([(1, 2, 'a', 0),
                                     (1, 2, 'a', 1),
                                     (1, 1, 'b', 0),
                                     (1, 1, 'b', 1),
                                     (2, 1, 'b', 0),
                                     (2, 1, 'b', 1)],
                                     names=['A', 'B', 'C', 'D'])

   s
   # SparseSeries
   ss = s.to_sparse()
   ss

In the example below, we transform the ``SparseSeries`` to a sparse representation of a 2-d array by specifying that the first and second ``MultiIndex`` levels define labels for the rows and the third and fourth levels define labels for the columns. We also specify that the column and row labels should be sorted in the final sparse representation.

.. ipython:: python

   A, rows, columns = ss.to_coo(row_levels=['A', 'B'],
                                column_levels=['C', 'D'],
                                sort_labels=True)

   A
   A.todense()
   rows
   columns

Specifying different row and column labels (and not sorting them) yields a different sparse matrix:

.. ipython:: python

   A, rows, columns = ss.to_coo(row_levels=['A', 'B', 'C'],
                                column_levels=['D'],
                                sort_labels=False)

   A
   A.todense()
   rows
   columns

A convenience method :meth:`SparseSeries.from_coo` is implemented for creating a ``SparseSeries`` from a ``scipy.sparse.coo_matrix``.

.. ipython:: python
   :suppress:

.. ipython:: python

   from scipy import sparse
   A = sparse.coo_matrix(([3.0, 1.0, 2.0], ([1, 0, 0], [0, 2, 3])),
                               shape=(3, 4))
   A
   A.todense()

The default behaviour (with ``dense_index=False``) simply returns a ``SparseSeries`` containing
only the non-null entries.

.. ipython:: python

   ss = SparseSeries.from_coo(A)
   ss

Specifying ``dense_index=True`` will result in an index that is the Cartesian product of the
row and columns coordinates of the matrix. Note that this will consume a significant amount of memory
(relative to ``dense_index=False``) if the sparse matrix is large (and sparse) enough.

.. ipython:: python

   ss_dense = SparseSeries.from_coo(A, dense_index=True)
   ss_dense

