# Profiling

An essential part of producing high-performance Python programs is profiling. That is, keeping track of memory usage and runtime behavior of your code at critical locations. In Python, there exist various possibilities for both kinds of profiling (memory and runtime).

## Runtime Profiling

Runtime profiling, in the most basic case, means simply recoding the current time before your program, recording it after and finally comparing the delta between those two measurements. In Python, this can be done with the `time` and `timeit` module.

### `time`

The `time.time()` function gives us the number of elapsed seconds since the epoch. We can use it like so to time some critical function of interest to us:

```Python
import time

start = time.time()
critical_function()
end = time.time()

delta = end - start

print('{0} took {1} seconds'.format(critical_function.__name__, delta))
```

We can extend this into a simple decorator that performs these measurements automatically:

```Python
def benchmark(function):

    # The start timestamp
    start = None

    @functools.wraps(function)
    def proxy(*args, **kwargs):
        # Need to reference the non-local start because we want to assign to it
        nonlocal start

        # Keep track of the very first call so that we only
        # benchmark the outer call for recursive functions
        is_first_call = start is None
        if is_first_call:
            start = time.time()

        # Call the actual function we want to benchmark
        result = function(*args, **kwargs)

        # Benchmark only at the end of the first (outer most) call
        if is_first_call:

            # Compute the elapsed time
            end = time.time()
            delta = end - start

            print('{0} took {1} seconds'.format(function.__name__, delta))

            # Reset for further calls to this function
            start = None

        return result
    return proxy
```

### `timeit`

`timeit` is a Python built-in module and command-line utility that provides simple timing functionality. It has two main functions: `timeit` and `repeat`. The latter is a generalized version of `timeit` in the sense that it will run `timeit` multiple times and return a list of the result of each execution. But first of all, let's look at `timeit`'s interface:

* `statement`: A string statement to time.
* `setup`: A setup statement string to execute once before the timing loop.
* `number`: The number of times to run the statement in a loop.
* `globals`: A dictionary of globals to include in the scope of the statement during execution.

Calling `timeit` will then first run the setup statement string (more or less `eval` it) and then, `number` times, execute the `statement`. It then returns the *total* execution time across *all iterations* (not that of each individual execution/iteration).

An example would be:

```python
import timeit

x = 0

t = timeit.timeit(stmt='print(x)', setup="x = 5", number=100, globals=dict(x=x))
print(t) # 0.0005621060263365507
```

It is noteworthy that the `timeit` module will temporarily disable Python's garbage collector utility via `gc.disable()` (see the `gc` module) to ensure multiple runs of `timeit` produce somewhat deterministic results. To change this, you can pass `gc.enable()` as the `setup` statement string.

`timeit.repeat` has the same interface, except for the `repeat` parameter, which determines how often the entire `timeit.timeit()` loop is executed. For each loop execution (for each invocation of `timeit.timeit()`), the result of that execution is returned:

```python
import timeit
print(timeit.repeat('x = 5; x += 1', repeat=2, number=1000))
```

We can again write a useful decorator to add `timeit` functionality to any function:

```python
def enable_timeit(number=1000, repeat=1):
    def inner_enable_timeit(function):

        # The actual wrapped function (identity function)
        def proxy(*args, **kwargs):
            return function(*args, **kwargs)

        def execute_timeit(*args, **kwargs):
            # Stringify the positional and keyword arguments
            args = ', '.join(map(str, args))
            kwargs = ', '.join('{0}={1}'.format(k,v) for k,v in kwargs.items())
            arguments = ', '.join((args, kwargs))
            statement = '{0}({1})'.format(function.__name__, arguments)

            result = timeit.repeat(
                statement,
                number=number,
                repeat=repeat,
                globals={function.__name__: function}
            )

            # Unpack the single result for convenience
            if repeat == 1:
                return result[0]
            return result

        # Add the "member function"
        proxy.timeit = execute_timeit

        return proxy

    return inner_enable_timeit
```

### `line_profiler`

A non-built-in but very useful module and command-line utility is `line_profiler` (also known as `kernprof`), developed [here](https://github.com/rkern/line_profiler). It allows for a great deal of insight into the performance of every line in a Python function.

Install it with:

```bash
$ pip install line_profiler
```

Then, you must annotate (decorate) the function you wish to profile with `@profile`. Note you don't have to import that decorator from anywhere, just write it above the target function and the command line utility will do the rest:

```python
@profile
def foo():
  pass
```

Lastly, simply call `kernprof -lv module_to_profile.py`. This will pick up all functions with the `@profile` decorator and output a detailed line-by-line analysis of the runtime of the function:


Note that because the `profile` decorator will not actually be defined anywhere in your code, you will want to add some "shims" for testing or other cases where you are not profiling the function:

```python
import builtins

if 'profile' not in dir(builtins):
    def profile(function):
        def identity(*args, **kwargs):
            return function(*args, **kwargs)
        return identity
```

## Memory

Next to runtime profiling, we are often, of course, also interested in the memory consumption of our programs. For this, there exists one main tool of interest: `memory_profiler`, which acts much like `line_profiler` (a.k.a. `kernprof`). Install it:

```bash
$ pip install memory_profiler
```

Then, just like with `line_profiler`, annotate the functions you want to investigate with the `@profile` decorator:

```python
@profile
def foo():
  l = [x for x in range(1000)]
  return l[1]
```

Then, running `python -m memory_profiler module_to_profile.py` will output a similar analysis as `line_profiler`. However, additionally, we can run `mprof run module_to_profile.py`. This will record and store a statistics file. If we then run `mprof plot`, the last such statistics file generated will be plotted using `matplotlib` (you'll want to have that installed).

Furthermore, for precisely this functionality, you can add checkpoints to your program using `profile.timestamp('name_of_the_checkpoint')`. This will then be shown on the timeline in the plot:

```python
@profile
def foo():
  with profile.timestamp('creating_big_list'):
    l = [x for x in range(1000)]
  with profile.timestamp('about_to_return'):
  return l[1]
```

Note that the names of the checkpoints must not contain spaces (hyphens or underscores are OK).
