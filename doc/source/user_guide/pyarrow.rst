.. _pyarrow:

{{ header }}

*********************
PyArrow Functionality
*********************

pandas can utilize `PyArrow <https://arrow.apache.org/docs/python/index.html>`__ to extend functionality and improve the performance
of various APIs. This includes:

* More extensive `data types <https://arrow.apache.org/docs/python/api/datatypes.html>`__ compared to NumPy
* Missing data support (NA) for all data types
* Performant IO reader integration
* Facilitate interoperability with other dataframe libraries based on the Apache Arrow specification (e.g. polars, cuDF)

To use this functionality, please ensure you have :ref:`installed the minimum supported PyArrow version. <install.optional_dependencies>`


Data Structure Integration
--------------------------

A :class:`Series`, :class:`Index`, or the columns of a :class:`DataFrame` can be directly backed by a :external+pyarrow:py:class:`pyarrow.ChunkedArray`
which is similar to a NumPy array. To construct these from the main pandas data structures, you can pass in a string of the type followed by
``[pyarrow]``, e.g. ``"int64[pyarrow]""`` into the ``dtype`` parameter

.. ipython:: python

   ser = pd.Series([-1.5, 0.2, None], dtype="float32[pyarrow]")
   ser

   idx = pd.Index([True, None], dtype="bool[pyarrow]")
   idx

   df = pd.DataFrame([[1, 2], [3, 4]], dtype="uint64[pyarrow]")
   df

For PyArrow types that accept parameters, you can pass in a PyArrow type with those parameters
into :class:`ArrowDtype` to use in the ``dtype`` parameter.

.. ipython:: python

   import pyarrow as pa
   list_str_type = pa.list_(pa.string())
   ser = pd.Series([["hello"], ["there"]], dtype=pd.ArrowDtype(list_str_type))
   ser

   from datetime import time
   idx = pd.Index([time(12, 30), None], dtype=pd.ArrowDtype(pa.time64("us")))
   idx

   from decimal import Decimal
   decimal_type = pd.ArrowDtype(pa.decimal128(3, scale=2))
   data = [[Decimal("3.19"), None], [None, Decimal("-1.23")]]
   df = pd.DataFrame(data, dtype=decimal_type)
   df

If you already have an :external+pyarrow:py:class:`pyarrow.Array` or :external+pyarrow:py:class:`pyarrow.ChunkedArray`,
you can pass it into :class:`.arrays.ArrowExtensionArray` to construct the associated :class:`Series`, :class:`Index`
or :class:`DataFrame` object.

.. ipython:: python

   pa_array = pa.array([{"1": "2"}, {"10": "20"}, None])
   ser = pd.Series(pd.arrays.ArrowExtensionArray(pa_array))
   ser

To retrieve a pyarrow :external+pyarrow:py:class:`pyarrow.ChunkedArray` from a :class:`Series` or :class:`Index`, you can call
the pyarrow array constructor on the :class:`Series` or :class:`Index`.

.. ipython:: python

   ser = pd.Series([1, 2, None], dtype="uint8[pyarrow]")
   pa.array(ser)

   idx = pd.Index(ser)
   pa.array(idx)

Operations
----------

PyArrow data structure integration is implemented through pandas' :class:`~pandas.api.extensions.ExtensionArray` :ref:`interface <extending.extension-types>`;
therefore, supported functionality exists where this interface is integrated within the pandas API. Additionally, this functionality
is accelerated with PyArrow `compute functions <https://arrow.apache.org/docs/python/api/compute.html>`__ where available. This includes:

* Numeric aggregations
* Numeric arithmetic
* Numeric rounding
* Logical and comparison functions
* String functionality
* Datetime functionality

The following are just some examples of operations that are accelerated by native PyArrow compute functions.

.. ipython:: python

   ser = pd.Series([-1.545, 0.211, None], dtype="float32[pyarrow]")
   ser.mean()
   ser + ser
   ser > (ser + 1)

   ser.dropna()
   ser.isna()
   ser.fillna(0)

   ser_str = pd.Series(["a", "b", None], dtype="string[pyarrow]")
   ser_str.str.startswith("a")

   from datetime import datetime
   pa_type = pd.ArrowDtype(pa.timestamp("ns"))
   ser_dt = pd.Series([datetime(2022, 1, 1), None], dtype=pa_type)
   ser_dt.dt.strftime("%Y-%m")

I/O Reading
-----------

PyArrow also provides IO reading functionality that has been integrated into several pandas IO readers. The following
functions provide an ``engine`` keyword that can dispatch to PyArrow to accelerate reading from an IO source.

* :func:`read_csv`
* :func:`read_json`
* :func:`read_orc`
* :func:`read_feather`

.. ipython:: python

   import io
   data = io.StringIO("""a,b,c
      1,2.5,True
      3,4.5,False
   """)
   df = pd.read_csv(data, engine="pyarrow")
   df

By default, these functions and all other IO reader functions return NumPy-backed data. These readers can return
PyArrow-backed data by specifying the parameter ``use_nullable_dtypes=True`` **and** the global configuration option ``"mode.dtype_backend"``
set to ``"pyarrow"``. A reader does not need to set ``engine="pyarrow"`` to necessarily return PyArrow-backed data.

.. ipython:: python

    import io
    data = io.StringIO("""a,b,c,d,e,f,g,h,i
        1,2.5,True,a,,,,,
        3,4.5,False,b,6,7.5,True,a,
    """)
    with pd.option_context("mode.dtype_backend", "pyarrow"):
        df_pyarrow = pd.read_csv(data, use_nullable_dtypes=True)
    df_pyarrow.dtypes

To simplify specifying ``use_nullable_dtypes=True`` in several functions, you can set a global option ``nullable_dtypes``
to ``True``. You will still need to set the global configuration option ``"mode.dtype_backend"`` to ``pyarrow``.

.. code-block:: ipython

    In [1]: pd.set_option("mode.dtype_backend", "pyarrow")

    In [2]: pd.options.mode.nullable_dtypes = True

Several non-IO reader functions can also use the ``"mode.dtype_backend"`` option to return PyArrow-backed data including:

* :func:`to_numeric`
* :meth:`DataFrame.convert_dtypes`
* :meth:`Series.convert_dtypes`
