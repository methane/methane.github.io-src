---
title: "Faster unicode access from Python 3.3/C API"
date: "2014-12-26"
categories:
    - "python"
---

Python 3.3 intorduces [PEP 393 Flexible String Representation](https://www.python.org/dev/peps/pep-0393/).
This is not only memory efficient, but also make it fast especially extension module.
Before Python 3.3, there are no way to build unicode by byte-to-byte operation.  Since Python 3.3,
you can do it for ASCII or Latin-1 string.


## PEP 393 summarized

There are some internal representation for unicode.

### Compact (non ASCII)

Compact representation is basic form of unicode.

```
+--------+--------+------+-------+--------------+-------+-------------+--------------+
| Header | length | hash | *wstr | utf-8 length | *utf8 | wstr_length | DATA ... NUL |
+--------+--------+------+-------+--------------+-------+-------------+--------------+
```

DATA is array of codepoints.  Each code point have size of 1byte, 2byte or 4byte (KIND).
When unicode only contains latin-1 chars (and contains no-ASCII chars), each codepoint is represented 1byte.

wstr and utf8 is cache for Python/C API.  When calling `PyUnicode_AsUTF8()` or `PyUnicode_AsUTF8AndSize()`, utf8 cache is created.
This is how it encode utf-8:

```c
    if (PyUnicode_UTF8(unicode) == NULL) {
        assert(!PyUnicode_IS_COMPACT_ASCII(unicode));
        bytes = _PyUnicode_AsUTF8String(unicode, "strict");
        if (bytes == NULL)
            return NULL;
        _PyUnicode_UTF8(unicode) = PyObject_MALLOC(PyBytes_GET_SIZE(bytes) + 1);
        if (_PyUnicode_UTF8(unicode) == NULL) {
            PyErr_NoMemory();
            Py_DECREF(bytes);
            return NULL;
        }
        _PyUnicode_UTF8_LENGTH(unicode) = PyBytes_GET_SIZE(bytes);
        Py_MEMCPY(_PyUnicode_UTF8(unicode),
                  PyBytes_AS_STRING(bytes),
                  _PyUnicode_UTF8_LENGTH(unicode) + 1);
        Py_DECREF(bytes);
    }

    if (psize)
        *psize = PyUnicode_UTF8_LENGTH(unicode);
    return PyUnicode_UTF8(unicode);
```

As you can see, this is slower and inefficient than `PyUnicode_AsUTF8String()` since it has additional malloc and memcpy.
You should prefer `PyUnicode_AsUTF8String()` except that you know cache is used later.


### Compact ASCII

ASCII string is special:

* utf-8 length == length
* utf-8 data == data
* wstr length == length

So it is more compact than compact representation.

```
+--------+--------+------+-------+--------------+
| Header | length | hash | *wstr | DATA ... NUL |
+--------+--------+------+-------+--------------+
```

`PyUnicode_AsUTF8()` and `PyUnicode_AsUTF8AndSize()` doesn't create cache.  You can use it free.

### Legacy

Legacy representation is only for support deprecated APIs.


## Case 1: [UltraJSON](https://github.com/esnme/ultrajson)

UltraJSON is fast JSON encoder and decoder.

Typical JSON contains many ascii strings.  Encoding to utf-8 can be skipped when unicode is Compact ASCII.
Otherwise, I don't use `PyUnicode_AsUTF8AndSize()` since it can be slower and memory inefficient.

```
# https://github.com/esnme/ultrajson/pull/159
diff --git a/python/objToJSON.c b/python/objToJSON.c
index e56aa9b..a5b2f62 100644
--- a/python/objToJSON.c
+++ b/python/objToJSON.c
@@ -145,7 +145,17 @@ static void *PyStringToUTF8(JSOBJ _obj, JSONTypeContext *tc, void *outValue, siz
 static void *PyUnicodeToUTF8(JSOBJ _obj, JSONTypeContext *tc, void *outValue, size_t *_outLen)
 {
   PyObject *obj = (PyObject *) _obj;
-  PyObject *newObj = PyUnicode_EncodeUTF8 (PyUnicode_AS_UNICODE(obj), PyUnicode_GET_SIZE(obj), NULL);
+  PyObject *newObj;
+#if (PY_VERSION_HEX >= 0x03030000)
+  if(PyUnicode_IS_COMPACT_ASCII(obj))
+  {
+    Py_ssize_t len;
+    char *data = PyUnicode_AsUTF8AndSize(obj, &len);
+    *_outLen = len;
+    return data;
+  }
+#endif
+  newObj = PyUnicode_AsUTF8String(obj);
   if(!newObj)
   {
     return NULL;
```

Before this patch:
```console
$ python3.4 -m timeit -n 10000 -s 'import ujson; x = ["a"*10]*100' 'ujson.dumps(x)'
10000 loops, best of 3: 15.8 usec per loop
```

After this patch:
```console
$ python3.4 -m timeit -n 10000 -s 'import ujson; x = ["a"*10]*100' 'ujson.dumps(x)'
10000 loops, best of 3: 7.14 usec per loop
```


## Case 2: [meinheld](https://github.com/mopemope/meinheld)

meinheld is fast WSGI server.  It can be used with [gunicorn](http://gunicorn.org/) or [circus](http://circus.readthedocs.org/) + [chaussette](https://chaussette.readthedocs.org/).

HTTP header field name is ASCII.  And [PEP 3333](https://www.python.org/dev/peps/pep-3333/) specifies field value should be decoded by latin-1.

`PyUnicode_New()` accepts maxchar as second argument. When maxchar<128, it creates Compact ASCII unicode.

Since field name is ASCII, environment name like `"HTTP_ACCEPT_ENCODING"` can be created as Compact ASCII.

field value may contain non-ASCII characters.  So I use `PyUnicode_DecodeLatin1()`.  It checks maxchar and create best unicode representation.

[Here](https://github.com/mopemope/meinheld/blob/c47cdba7a1fe6a80dc8a4f9f2ac5288104f1102f/meinheld/server/response.c#L44-L59)
is how to create header string in `char*` from unicode:

```python
static int
wsgi_to_bytes(PyObject *value, char **data, Py_ssize_t *len)
{
#ifdef PY3
    if (!PyUnicode_Check(value)) {
        PyErr_Format(PyExc_TypeError, "expected unicode object, value "
                     "of type %.200s found", value->ob_type->tp_name);
        return -1;
    }
    if (PyUnicode_READY(value) == -1) {
        return -1;
    }
    if (PyUnicode_KIND(value) != PyUnicode_1BYTE_KIND) {
        PyErr_SetString(PyExc_ValueError,
                        "unicode object contains non latin-1 characters");
        return -1;
    }
    *len = PyUnicode_GET_SIZE(value);
    *data = (char*)PyUnicode_1BYTE_DATA(value);
    return 0;
#else
    return PyBytes_AsStringAndSize(value, data, len);
#endif
}
```

Previously, it convert unicode to bytes with `PyUnicode_AsLatin1String()`.

Before this change:

```console
$ ./wrk http://localhost:8000/
Running 10s test @ http://localhost:8000/
  2 threads and 10 connections
^T^N  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   385.68us   25.67us 843.00us   74.12%
    Req/Sec    12.42k   632.30    13.78k    59.64%
  234475 requests in 10.00s, 39.13MB read
Requests/sec:  23448.18
Transfer/sec:      3.91MB
$ ./wrk http://localhost:8000/
Running 10s test @ http://localhost:8000/
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   379.78us   24.96us   0.90ms   73.86%
    Req/Sec    12.46k   639.91    14.00k    52.45%
  235688 requests in 10.00s, 39.33MB read
Requests/sec:  23569.38
Transfer/sec:      3.93MB
$ ./wrk http://localhost:8000/
Running 10s test @ http://localhost:8000/
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   388.15us   24.65us 524.00us   73.31%
    Req/Sec    12.43k   623.66    13.78k    55.26%
  234899 requests in 10.00s, 39.20MB read
Requests/sec:  23490.25
Transfer/sec:      3.92MB
```

After this change:

```console
$ ./wrk http://localhost:8000/
Running 10s test @ http://localhost:8000/
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   339.98us   65.87us   4.33ms   92.45%
    Req/Sec    13.56k     1.43k   15.44k    81.82%
  253189 requests in 10.00s, 42.26MB read
Requests/sec:  25319.67
Transfer/sec:      4.23MB
$ ./wrk http://localhost:8000/
Running 10s test @ http://localhost:8000/
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   364.82us   78.85us   1.01ms   84.10%
    Req/Sec    12.99k     1.81k   15.44k    80.63%
  243685 requests in 10.00s, 40.67MB read
Requests/sec:  24368.90
Transfer/sec:      4.07MB
$ ./wrk http://localhost:8000/
Running 10s test @ http://localhost:8000/
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   329.00us   22.40us 464.00us   73.99%
    Req/Sec    13.66k   760.36    15.44k    61.95%
  258730 requests in 10.00s, 43.18MB read
Requests/sec:  25873.60
Transfer/sec:      4.32MB
```
