# Unicode, str and bytes in Python 2 and 3

## Character Sets and Encodings

Generally, when dealing with text, we have two separate concepts:

* Characters sets (e.g. Unicode or ASCII) (Zeichensatz),
* Encoding (e.g. UTF-8) (Kodierungsvorschrift)

The former, character sets, signifies a data-agnostic code for describing characters. One common set of characters is the Unicode standard. Most importantly, it does not specify how characters or text are encoded in binary. They simply map text characters or symbols to a known, agreed-upon code. For example, in Unicode, characters are assigned a *code point*. Then, an encoding like UTF-8 or UTF-16 (UTF = Unicode Transformation Format) specifies how each code point is ultimately stored in binary, on the bit-sequence level.

## Python

In Python, this discrepancy is known and handled in various ways. Most importantly, there is a difference between Python versions 2 and 3

### Python 3

In Python 3, there are two relevant data types:

* `bytes`, which are sequences of 8-bit octets.
* `str`, which stores a sequence of unicode characters (code points).

Most importantly, `str`ings are agnostic of their encoding. To move between these two types, you use the `str.encode()` and `bytes.decode()` methods. The former takes an encoding format, like `'utf-8'` (as a string) and returns a `bytes` array. Conversely, the latter also takes an encoding and goes the reverse way, decoding the bytes into a character sequence.

Either way, the encoding always defaults to 'utf-8', which is an alias for 'utf8' and 'utf_8'. Other allowed encodings are:

* 'latin-1' ('so-8859-1')
* 'ascii'
* 'utf-16/32'
* 'shit_jis' (for Japanese languages)

See [here](https://docs.python.org/3.5/library/codecs.html#standard-encodings) for the complete list of supported encodings in CPython.

Most importantly, `bytes` and `str` really are two separate formats and never compare equal, even in the case of empty bytes/strings.

To ensure either a bytes or string type and convert between them, you could use the following functions:

```python
def to_str(string_or_bytes):
  if isinstance(string_or_bytes, bytes):
    return string_or_bytes.decode('utf-8')
  else:
    return string_or_bytes

def to_bytes(string_or_bytes):
  if isinstance(string_or_bytes, str):
    return string_or_bytes.encode('utf-8')
  else:
    return string_or_bytes
```

### Python 2

In Python 2, the situation is slightly different. There was no `bytes` datatype. Rather there was only `str` and `unicode`. `str` was, in fact, closer to what `bytes` is now -- a collection of octets. On the other hand, `unicode` was then what `str` is in Python3: a sequence of Unicode code points.

While plain string literals construct `str` instances, a string literal prefixed with a `u` constructs `unicode` strings in Python 2.

A very important and annoying issue with `str` and `unicode` in Python 2 was that in the case that a `str` instance contained only ASCII characters, `str` and `unicode` strings would compare equal. However, as soon as the `str` instance stored non-ASCII characters in a unicode transformation format, the `str` and corresponding `unicode` strings would no longer compare equal.

```python
# Python 2
In [0]: 'hello' == u'hello'
Out [0]: True
In [1]: 'hello' == u'bar'
Out [1]: False

# Python 3
In [0]: b'hello' == 'hello'
Out [0]: False
In [1]: b'hello' == 'bar'
Out [1]: False
```

To convert between `str` and `unicode` instances, you coul use the following helper functions:

```python
def to_str(string_or_unicode):
  if isinstance(string_or_unicode, unicode):
      return unicode.encode('utf-8')
  else:
    return string_or_unicode

def to_unicode(string_or_unicode):
  if isinstance(string_or_unicode, str):
      return unicode.decode('utf-8')
  else:
    return string_or_unicode
```

## Files

Another noteworthy situation where encodings become relevant is file I/O. In Python2, file handles as returned by `open` will `read` and `write` byte sequences (i.e. Python 3 `bytes` / Python 2 `str`), while in Python3 they expect character sequences (i.e. Python 3 `str` / Python 2 `unicode`). Moreover, the Python 3 `open` function takes an explicit `encoding` parameter, which defaults to `utf-8` and determines how the input character sequences are encoded to store them as binary. This means:

* If you want to write text in Python 3, you must pass `str` instances and optionally pass an encoding, it you don't want UTF-8.
* If you want to write text in Python 2:
  + And are using `unicode` strings, you must encode them yourself first.
  + And are using `str` strings, you can just pass them, but make sure to only have ASCII strings then.
* If you want to write `bytes` in Python 3, you must write in binary mode (`'wb'`).
* If you want to write bytes in Python 2, you *can* use `'w'` mode, but only if you are writing ASCII strings/unicode. Else you must use `'wb'` and encode/decode properly.

Most importantly, just use binary mode `'wb'` if you really expect to write bytes (`bytes` or `str`). That will work in Python 2 and 3:

```python
with open('data', 'wb') as destination:
  destination.write(os.urandom(10))
```
