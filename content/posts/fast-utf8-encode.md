---
title: "Faster unicode access from Python 3.3/C API"
date: "2014-12-26"
categories:
    - "python"
---
**This post is DRAFT**

Python 3.3 intorduces [PEP 393 Flexible String Representation](https://www.python.org/dev/peps/pep-0393/).
This is not only memory efficient, but also make it fast especially extension module.
Before Python 3.3, there are no way to build unicode by byte-to-byte operation.  Since Python 3.3,
you can do it for ASCII or Latin-1 string.


## PEP 393 summarized

There are some internal representation for unicode.

### Compact (non ASCII)

Compact representation is basic form of unicode.

<pre>
+--------+--------+------+-------+--------------+-------+-------------+--------------+
| Header | length | hash | *wstr | utf-8 length | *utf8 | wstr_length | DATA ... NUL |
+--------+--------+------+-------+--------------+-------+-------------+--------------+
</pre>

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

<pre>
+--------+--------+------+-------+--------------+
| Header | length | hash | *wstr | DATA ... NUL |
+--------+--------+------+-------+--------------+
</pre>

`PyUnicode_AsUTF8()` and `PyUnicode_AsUTF8AndSize()` doesn't create cache.  You can use it free.

### Legacy

Legacy representation is only for support deprecated APIs.


## Example 1: [ujson](https://github.com/esnme/ultrajson)

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


## Example 2: [meinheld](https://github.com/mopemope/meinheld)

HTTP header field name is ASCII.  And [PEP 3333](https://www.python.org/dev/peps/pep-3333/) specifies header value should be decoded by latin-1.

`PyUnicode_New()` accepts maxchar as second argument. When maxchar<128, it creates Compact ASCII unicode.

Since field name is ASCII, environment name like `"HTTP_ACCEPT_ENCODING"` can be created as Compact ASCII.

field value may contain non-ASCII characters.  So I use `PyUnicode_DecodeLatin1()`.  It checks maxchar and create best unicode representation.
