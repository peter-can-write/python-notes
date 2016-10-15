# Slicing

Slicing is a feature in Python that gives a *view* of a sequence.

## What can thy slice?

Any sequence is sliceable if it has a `__setitem__`/`__getitem__` method (usually used as subscript operator), since the slicing syntax `sequence[start:end:step]` is really nothing else than passing a `slice` object to the subscript (`__setitem__`/`__getitem__`) operator.

For example, we can write our own `list` wrapper that supports slicing simply by giving it a `__getitem__` method that then checks if the element passed is a slice and if so, returns does something special or else just accesses an index:

```python
class Wrapper(object):
  def __init__(self, sequence):
    self._sequence = sequence

  def __getitem__(self, index_or_slice):
    if isinstance(index_or_slice, slice):
      print('slice requested!')
      # Create the same slice, but with double the stride
      new_slice = slice(
          index_or_slice.start,
          index_or_slice.stop,
          index_or_slice.step * 2 if index_or_slice.step else 2
      )
      return self._sequence[new_slice]
    else:
      assert isinstance(index_or_slice, int)
      return self._sequence[index_or_slice]
```

Then:

```python
In [0]: w = Wrapper([1, 2, 3, 4, 5])
In [1]: w[1]
Out [1]: 2
In [2]: w[:]
slice requested!
Out [2]: [1, 3, 5]
```

## Slice objects

As you can see, slices are explicit objects in Python. They also have a constructor, `slice`, so you can even instantiate and store/pass around slices! This constructor takes a `start`, `stop` (end) and `step` (stride) parameter. It then makes these available via attributes. Moreover, `slice`s have an `indices()` method that takes a length (such as that of a sequence) and returns a "sanitized" version of the slice as a triple (start, stop and stride). With "sanitized", we mean one that is valid w.r.t to a sequence of such length.

```python
In [0]: s = slice(1, None, 2)

In [1]: s.start
Out[1]: 1

In [2]: s.stop
# None

In [3]: s.step
Out[3]: 2

In [4]: s.indices(5)
Out[4]: (1, 5, 2)
```

## Slice assignment

One interesting feature you can achieve with slices is in-place modifications of ranges within sequences. For example, say we have the list `l = [1, 2, 3, 4, 5]`. Then writing `l[1:3] = 'ab'` will assign the characters 'a' and 'b' to the list at indices 1 and 2. More generally, assigning to a slice of a list (or anything with `__setitem__`, see above) will assign whatever iterable is on the right side to that part of the list.

Most interestingly, the iterable to the right and the slice to the left need not be of equal length. This allows you to perform insertion, deletion and replacement. For this, let `l = [1, 2, 3]`. Then:

* Insertion:
  + `l[1:3] = [4, 5, 6]` produces `[1, 4, 5, 6]`
  + `l[1:1] = [4, 5, 6]` produces [1, 4, 5, 6, 2, 3]

* Deletion:
  + `l[:] = []` clears the list.
  + `l[1:] = []` removes all but the first element.
  + `l[:2] = []` removes all but the last element.

* Replacement:
  + `l[0:2] = [5, 6]` produces `[5, 6, 3]`
  + `l[:] = [1]` produces `[1]`

Note that replacement can be equated with tuple assignment, to each referenced index, when the slice and assigned iterable are of the same length. That is, `l[1:3] = 'ab'` is the same as `l[1], l[2] = 'ab'`. However, `l[1] = 'ab'` would not work, because the iterable on the right has a greater length than the slice.

Also note, especially, how `l = []` differs from `l[:] = []` for some list `l`. The former will simply assign the reference `l` to a newly allocated, empty list (and decrement the reference count of whatever `l` pointed to before). The latter will clear whatever was in `l` before.

Resources:

* http://stackoverflow.com/questions/10623302/how-assignment-works-with-python-list-slice

## Recipes

Interesting recipes and things you can do with slicing:

* (Shallow) copy a list: `copy = my_list[:]` (same as `my_list.copy()`)
