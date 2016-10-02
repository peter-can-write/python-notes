# Reduce

`functools.reduce(function, iterable[, initializer])`

`reduce` takes a function and an iterable range and applies the function to the first two values, then applies the function to the result and the third value etc. `reduce` also takes an optional third argument `start`, which defaults to `None`, which you can pass as the initial value of the reduction:

```Python
from functools import reduce

l = [1, 2, 3, 4]

reduce(lambda a, b: a + b, l) # n(n + 1)/2 = 10
```

It can be implemented like so:

```Python
#/usr/bin/env python
# -*- coding: utf-8 -*-

def reduce(function, iterable, initial=None):
	iterator = iter(iterable)
	if initial is None:
		try:
			# try to set to the first value
			initial = next(iterator)
		except StopIteration:
			raise RuntimeError("Cannot reduce empty "
			                   "iterable without initial value!")
	result = initial
	while True:
		try:
			result = function(result, next(iterator))
		except StopIteration:
			return result

def main():
	print(reduce(lambda a, b: a + b, [1, 2, 3, 4], 10))

if __name__ == '__main__':
	main()
```
