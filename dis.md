# Disassembling Python Programs

In Python, the `dis` module allows disassembly of Python code into the individual instructions executed by the Python interpreter (usually cPython) for each line. Passing a module, function or other piece of code to the `dis.dis` function will return a human-readable representation of the underlying, disassembled bytecode. This is useful for analyzing and hand-tuning tight loops or perform other kinds of necessary, fine-grained optimizations.

## Basic Usage

The main function you will interact with when wanting to disassemble Python code is `dis.dis`. It takes either a function, method, class, module, code string, generator or byte sequence of raw bytecode and prints the disassembly of that code object to `stdout` (if no explicit `file` argument is specified). In the case of a class, it will disassemble each method (also static and class methods). For a module, it disassembles all functions in that module.

Let's see this in practice. Take the following code:

```python
import dis

class Foo(object):
  def __init__(self):
    pass

  def foo(self, x):
    return x + 1

def bar():
  x = 5
  y = 7
  z = x + y
  return z

def main():
  dis.dis(bar) # disassembles `bar`
  dis.dis(Foo) # disassembles each method in `Foo`
```

This will print:

```
14           0 LOAD_CONST               1 (5)
             3 STORE_FAST               0 (x)

15           6 LOAD_CONST               2 (7)
             9 STORE_FAST               1 (y)

16          12 LOAD_FAST                0 (x)
            15 LOAD_FAST                1 (y)
            18 BINARY_ADD
            19 STORE_FAST               2 (z)

17          22 LOAD_FAST                2 (z)
            25 RETURN_VALUE

Disassembly of __init__:
 8           0 LOAD_CONST               0 (None)
             3 RETURN_VALUE

Disassembly of foo:
11           0 LOAD_FAST                1 (x)
             3 LOAD_CONST               1 (1)
             6 BINARY_ADD
             7 RETURN_VALUE
```

Also, we can disassemble an entire module from the command line using `python -m dis module_file.py`. Either way, at this point, we should probably discuss the format of the disassembly output. The columns returned are the following:

1. The original line of code the disassembly is referencing.
2. The address of the bytecode instruction.
3. The name of the instruction.
4. The index of the argument in the code block's name and constant table.
5. The human-friendly mapping from the argument index (4) to the actual value or name being referenced.

For (4), it is important to understand that all *code objects* in Python, that is, isolated code blocks like functions, have internal *name and constant tables*. These tables are simply lists, where the constant table would hold constants such as string literals, numbers or special values such as `None` that appear at least once in the code block, while the name table will hold a list of variable names. These variable names are then, further, keys into a dictionary mapping such symbols to actual values. The reason why instruction arguments are indices into tables and not the values stored in those tables is so that arguments can have uniform length (always two bytes). As you can imagine, storing variable-length strings in the bytecode directly makes advancing a program counter a great deal more complex.

## Code Objects

I just mentioned the concept of [*code objects*](https://docs.python.org/3.5/c-api/code.html). But what are these? Simply put, code objects are the compiled representation of a piece of code. This representation can then be understood by the cPython interpreter, which is the most common and popular implementation of the Python programming language. It is written in C and uses these code objects to execute Python code line-by-line. Next to cPython, there exist a number of other Python compilers and interpreters, including [PyPy](http://pypy.org) (which uses a JIT-compiler, often making it much faster than cPython) and [Jython](http://www.jython.org).

There are a few ways to generate code objects. The easiest is to access the `__code__` member that all functions and methods (remember that methods are just functions with an additional instance argument):

```python
def foo():
  pass


In [1]: foo.__code__
Out [1]: <code object foo at 0x109d419c0, file "<string>", line 1>
```

Alternatively, you can just compile any piece of code yourself, using the `compile` builtin. This function will return a code object. It takes the following parameters:

1. A string or [`ast`](https://docs.python.org/3.5/library/ast.html) object.
2. The name of the file in which the code is written, or a descriptive string if no such file exists (e.g. `"<string>"`).
3. The compilation mode, which can be one of the following three strings:
  * `'exec'`: Allows multiple expressions to be parsed.
  * `'eval'`: Allows only a single expression to be parsed (no spaces).
  * `'single'`: Interprets and evaluates an expression like an interactive shell. This means the code will be evaluated and any non-`None` result will be printed.

For example:

```python
code_object = compile('x = 5; x += 1', '<string>', 'exec') # OK
code_object = compile('x = 5; x += 1', '<string>', 'eval') # Error
```

These code objects provide all the information a Python interpreter such as cPython needs to execute the code represented by the object. For this, the code object has several interesting attributes:

* `co_code`: The actual bytecode byte string.
* `co_consts`: The constants available to the instructions ("the constants table").
* `co_names`: The global symbols available to the instructions.
* `co_nlocals`: The number of local symbols (functions and variables).
* `co_varnames`: The list of local symbols (functions and variables).
* `co_argcount`: If the code object is that of a function, the number of arguments it takes.

For example, we can see that when we create the following function:

```python
def foo():
  x = 5
  def bar():
    pass
  return len([x])
```

The code object of `foo` will have the constants (`co_consts`) `None` and `5`, the local variable names (`co_varnames`) `bar` and `x` and available symbols (`co_names`) `len`:

```python
In [1]: foo.__code__.co_consts
Out [1]: (None, 5)
In [2]: foo.__code__.co_varnames
Out [2]: ('bar', 'x')
In [3]: foo.__code__.co_names
Out [3]: ('len', )
```

## Understanding Disassembled Code

Now that we know what code objects are and that they have name and constant tables, we can better understand the output of `dis.dis()`. Let us take the disassembly of the following function:

```python
y = 5
def function():
  x = len([1, 2, 3])
  z = x + y
  return z
```

which is:

```
In [1]: dis.dis(function)
Out [1]:
  3           0 LOAD_GLOBAL              0 (len)
              3 LOAD_CONST               1 (1)
              6 LOAD_CONST               2 (2)
              9 LOAD_CONST               3 (3)
             12 BUILD_LIST               3
             15 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             18 STORE_FAST               0 (x)

  4          21 LOAD_FAST                0 (x)
             24 LOAD_GLOBAL              1 (y)
             27 BINARY_ADD
             28 STORE_FAST               1 (z)

  5          31 LOAD_FAST                1 (z)
             34 RETURN_VALUE
```

For example, the very first line shows the `LOAD_GLOBAL` instruction. It is the first bytecode instruction of the third line in the Python function, as indicated by the second and first column, respectively. As we can also see, it loads the name at index 0. Recall that this index is referencing the `co_names` list associated with the disassembled code object. The last column `len`, is simply a friendly hint by `dis.dis()` as to what the value at `co_names[0]` is.

But what does `LOAD_GLOBAL` do? Well, it loads a global. In this case the global function (symbol) `len`. That was obvious, but what is more interesting is *where* it loads the global to. The answer to this question lies in how the cPython interpreter is structured: it is entirely stack-based. This means that any function or method symbols, constants or variable names are added to a stack. Operations such as binary addition (the `BINARY_ADD` bytecode instruction you see) will then operate on this stack, popping two values and pushing the result back onto the stack. Function call instructions (`CALL_FUNCTION`) will similarly pop the function symbol and arguments from the stack, execute the function (which pushes a new *frame* onto the stack) and ultimately push the return value of the called function back onto the stack.

A complete list of instructions understood by cPython can be found [here](https://docs.python.org/3.5/library/dis.html#python-bytecode-instructions). Some interesting ones include:

* `LOAD_FAST <index>`: Pushes the local variable at the given index onto the stack.
* `STORE_GLOBAL <index>`: Pops the top of the stack into the variable with the name specified at the given index.
* `BINARY_ADD`: Pops the top two values from the stack, adds them and pushes the result back onto the stack.
* `BUILD_LIST <size>`: Pops `size` many elements off the stack and allocates a new list.
* `CALL_FUNCTION <#arguments>`: Pops `#arguments` many arguments off the stack. The lower byte of this number specifies the number of positional arguments, the high byte the number of keyword arguments. Then pops the function symbol (itself) off the stack, makes the call and pushes the return value of the function back onto the stack.
* `MAKE_FUNCTION <#arguments>`: To create a function object, consumes:
  * All default arguments in positional order,
  * The code object of the function,
  * The qualified name of the function,
  from the stack and pushes the created function object onto the stack.
* `POP_TOP`: Simply pops (as in, "removes") the value at the top of the stack.

## Practical examples

Let's now look at practical examples of how looking at the disassembly of our programs can help you optimize and better understand your code.

### Investigating Optimizations

One simple case is investigating when your [Python compiler](http://programmers.stackexchange.com/questions/24558/is-python-interpreted-or-compiled) performs certain optimizations and when not. For example, take the following code:

```python
i = 1 + 2
f = 3.4 * 5.6
s = 'Hello,' + ' World!'

I = i * 3 * 4
F = f / 2 / 3
S = s + ' Bye!'
```

Doing a disassembly shows us (comments mine):

```python
### i
2           0 LOAD_CONST              10 (3)
            3 STORE_FAST               0 (i)

### f

3           6 LOAD_CONST              11 (19.04)
            9 STORE_FAST               1 (f)

### s

4          12 LOAD_CONST              12 ('Hello, World!')
           15 STORE_FAST               2 (s)

### I

6          18 LOAD_FAST                0 (i)
           21 LOAD_CONST               7 (3)
           24 BINARY_MULTIPLY
           25 LOAD_CONST               8 (4)
           28 BINARY_MULTIPLY
           29 STORE_FAST               3 (I)

### F

7          32 LOAD_FAST                1 (f)
           35 LOAD_CONST               2 (2)
           38 BINARY_TRUE_DIVIDE
           39 LOAD_CONST               7 (3)
           42 BINARY_TRUE_DIVIDE
           43 STORE_FAST               4 (F)

### S

8          46 LOAD_FAST                2 (s)
           49 LOAD_CONST               9 (' Bye!')
           52 BINARY_ADD
           53 STORE_FAST               5 (S)
           56 LOAD_CONST               0 (None)
```

As you can see, the compiler will readily perform simple arithmetic operations at "compile time", such as adding `1 + 2` or reducing the expression `3.4 * 5.6` to the constant `19.04`. The compiler will even perform string concatenation immediately. However, as soon as a single variable (see `I` or `f`) is involved, the compiler stops optimizing and loads the variable name as well as all constants individually and performs the binary operations one-by-one in the natural order of the operation.

### When Rolling-Your-Own is Bad

Now, let's look at a more realistic case where looking at the disassembly would have really given you insight. In Python, we usually use `range` and iterators to loop over a sequence of numbers. However, we may think that calling `range` and using the iterator may be too costly compared to a roll-your-own C-like loop. That is, we want to compare:

```python
for i in range(x):
  pass
```

with

```python
i = 0
while i < x:
  i += 1
```

Let's do just that and profile them:

```python
In [1]: timeit.timeit('for i in range(x): pass', globals=dict(x=100))
Out[1]: 1.4965313940192573

In [2]: timeit.timeit('i = 0\nwhile i < x: i += 1', globals=dict(x=100))
Out[2]: 7.19753862300422
```

As we can see, the built-in version is much faster. Let's disassemble the code and try to understand why. First for the `range` loop:

```
1           0 SETUP_LOOP              20 (to 23)
            3 LOAD_NAME                0 (range)
            6 LOAD_NAME                1 (x)
            9 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
           12 GET_ITER
      >>   13 FOR_ITER                 6 (to 22)
           16 STORE_NAME               2 (i)
           19 JUMP_ABSOLUTE           13
      >>   22 POP_BLOCK
      >>   23 LOAD_CONST               0 (None)
           26 RETURN_VALUE
```

Then for the roll-your-own version:

```
1           0 LOAD_CONST               0 (0)
            3 STORE_NAME               0 (i)

2           6 SETUP_LOOP              26 (to 35)
      >>    9 LOAD_NAME                0 (i)
           12 LOAD_NAME                1 (x)
           15 COMPARE_OP               0 (<)
           18 POP_JUMP_IF_FALSE       34
           21 LOAD_NAME                0 (i)
           24 LOAD_CONST               1 (1)
           27 INPLACE_ADD
           28 STORE_NAME               0 (i)
           31 JUMP_ABSOLUTE            9
      >>   34 POP_BLOCK
      >>   35 LOAD_CONST               2 (None)
           38 RETURN_VALUE
```

As we can see, the version using the built-in `range` function has fewer instructions than the roll-your-own variant. Ultimately, what this boils down to is that the version using `range` will make better use of internal C imrplementations of iterators, while our custom version requires repeated interpretation of the `INPLACE_ADD`, `COMPARE_OP`, `POP_JUMP_IF_FALSE` and other operations, which takes more time.

#### Dynamic Lookup

Lastly, inspecting the disassembly of code can give us quite interesting insight into how Python's dynamic language properties affect its performance. Recall that a symbol in Python must be re-evaluated on each invocation at runtime. This means that even if we use the exact same global variable twice in the same line of code, it will have to be retrieved from the global namespace twice, individually, irrespective of previous or future loads, given that it may very well change value and/or type between evaluations. Ultimately, this means that caching "deeply qualified" names (with lots of dots) and storing them into local variables can improve the performance of our code quite a bit.

As a simple example, say we want to sum the sines of a range of values -- lots of values. The naive way to do so would be the following:

```python
def first():
  total = 0
  for x in range(1000):
    total += math.sin(x)
  return total
```

When we disassemble this using `dis.dis(first)`, we get the following:

```
2           0 LOAD_CONST               1 (0)
            3 STORE_FAST               0 (total)

3           6 SETUP_LOOP              39 (to 48)
            9 LOAD_GLOBAL              0 (range)
           12 LOAD_CONST               2 (1000)
           15 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
           18 GET_ITER
      >>   19 FOR_ITER                25 (to 47)
           22 STORE_FAST               1 (x)

4          25 LOAD_FAST                0 (total)
           28 LOAD_GLOBAL              1 (math)
           31 LOAD_ATTR                2 (sin)
           34 LOAD_FAST                1 (x)
           37 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
           40 INPLACE_ADD
           41 STORE_FAST               0 (total)
           44 JUMP_ABSOLUTE           19
      >>   47 POP_BLOCK

5     >>   48 LOAD_FAST                0 (total)
           51 RETURN_VALUE
```

The important bit is line 4. As you can see, since we fully qualify `math.sin`, we have to:

1. `LOAD_GLOBAL` the `math` module symbol.
2. `LOAD_ATTR` the `sin` symbol.

Where (2) ultimately equates to a dictionary (hash table) lookup. Understanding the dynamic nature of Python, we may hypothesize that caching and storing the `math.sin` symbol in a local variable may greatly improve the performance of our code in the tight inner loop. Let's see:

```python
def second():
  sin = math.sin
  total = 0
  for x in range(1000):
    total += sin(x)
  return total
```

and disassembled:

```
2           0 LOAD_GLOBAL              0 (math)
            3 LOAD_ATTR                1 (sin)
            6 STORE_FAST               0 (sin)

3           9 LOAD_CONST               1 (0)
           12 STORE_FAST               1 (total)

4          15 SETUP_LOOP              36 (to 54)
           18 LOAD_GLOBAL              2 (range)
           21 LOAD_CONST               2 (1000)
           24 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
           27 GET_ITER
      >>   28 FOR_ITER                22 (to 53)
           31 STORE_FAST               2 (x)

5          34 LOAD_FAST                1 (total)
           37 LOAD_FAST                0 (sin)
           40 LOAD_FAST                2 (x)
           43 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
           46 INPLACE_ADD
           47 STORE_FAST               1 (total)
           50 JUMP_ABSOLUTE           28
      >>   53 POP_BLOCK

6     >>   54 LOAD_FAST                1 (total)
           57 RETURN_VALUE
```

If we now look at the inner loop (line 5), we can see that we've replaced the `LOAD_GLOBAL`, `LOAD_ATTR` sequence with `LOAD_FAST` -- a single local variable lookup. As the name implies, this will be a lot faster (supposedly). Let's check:

```python
In [1]: timeit.timeit('f()', globals=dict(f=first), number=100000)
Out[1]: 27.191193769976962

In [2]: timeit.timeit('f()', globals=dict(f=second), number=100000)
Out[2]: 15.568848820985295
```

Well, I don't know about your definition of "a lot faster", but a factor of (almost) two is quite a lot faster in my book.

## Interpreting Bytecode

Disassembled bytecode instructions are already quite low-level (a.k.a. cool). However, we can go even deeper and understand the *byte* code itself -- i.e. the binary or hexadecimal representation of the instructions in compiled and assembled bytecode. For this, let's define a function and mess a little more with its `__code__` property:

```python
def function():
  x = 5
  l = [1, 2]
  return len(l) + x
```

Through `function.__code__` we can gain access to the code object associated with the function. Furthermore, `function.__code__.co_code` returns the actual bytecode:

```python
In [1]: function.__code__.co_code
Out[1]: b'd\x01\x00}\x00\x00d\x02\x00d\x03\x00g\x02\x00}\x01\x00t\x00\x00|\x01\x00\x83\x01\x00|\x00\x00\x17S'
```

Yes! Bytes! Just what I like for breakfast. But what can we actually make of these delicious bites of bytecode? Well, we know that these bytes specify instructions, some taking arguments and some not. Each instruction will occupy a single byte and arguments (such as the indices into the name and constants table) will occupy further bytes. Furthermore, fortunately enough, the `dis` module (as well as the `opcode` module) provides an `opname` table and an `opmap` map. The former is a simple list, laid out such that indexing it with the opcode of an instruction will return the name (*mnemonic*) of that instruction. The latter, `dis.opmap`, maps instruction mnemonics to their bytecode numbers:

```python
In [1]: dis.opname[69]
Out[1]: 'GET_YIELD_FROM_ITER'

In [2]: dis.opmap['LOAD_CONST']
Out[2]: 100
```

So, if we know the byte value describing a certain instruction, we now know how to get the instruction name. All that's left is interpreting the arguments of these instructions. For this we need to know whether or not the instruction takes arguments in the first place. To get this information, we can make use of the `dis.hasconst`, `dis.hasname`, `dis.hasjrel` and `dis.hasjabs` and others. Each of these are lists in the `dis` module that contain the bytecodes either taking a a constant argument, a name argument, relative/absolute jump target or other kind of parameter. For example, `dis.hasnargs` is also such a list, containing all opcodes related to function calls, such as `CALL_FUNCTION`, `CALL_FUNCTION_VAR` (for functions taking `*args`) or `CALL_FUNCTION_KW` (for functions taking `**kwargs`). It is noteworthy that if an instruction takes arguments at all, it can only take a single argument occupying exactly 16 bits (two bytes).

### Writing Our Own Bytecode

For practice and fun, we could now actually write some bytecode ourselves. Say, for example, we wanted to write `x = 1 + 2` in bytecode. This equates to first loading the constants `1` and `2` onto the stack using `LOAD_CONST` with opcode `100` (`0x64`). Then, we perform a `BINARY_ADD` (opcode `23`/`0x17`), which will pop those two values off the stack and push the result (3) back on top. Then we'll do a `STORE_NAME` (opcode `125`/`0x7d`) to pop the result off the stack and store it in the only variable we'll have, i.e. at index 0. This equates to:

```
0 LOAD_CONST 0 # 0x64
3 LOAD_CONST 1 # 0x64
6 BINARY_ADD   # 0x17
7 STORE_NAME 0 # 0x7d

consts = (1, 2)
names = ('x',)
```

or, in (decimal) bytes:

```
100 0 0 # consts[0]
100 1 0 # consts[1]
23      # no arguments
125 0 0 # names[0]
```

Note that the compiler would actually optimize `1 + 2` to the constant `3`, as you'll note if you run `dis.dis('x = 1 + 2')`. Also, when you run `dis.dis`, you'll see that the code seems to be returning `None` at the end. This is simply an implementation detail of cPython, related to it being written in C and always requiring a return value (even for code outside functions).

The disassembly also tells us that a single `LOAD_CONST` instruction consumes three bytes. One for the opcode and two for the 16-bit parameter. Lastly, it is important to mention that arguments are stored in *little endian* format. That's why `LOAD_CONST 1` is `100 1 0`, sine `0b00000001 0b00000000` is `1` in little endian representation.

### Writing a Disassembler

Given all this information, we're now more than set to write our own little bytecode disassembler. We'll simply take a bytestring, such as returned by a code object's `co_code` attribute, and loop through it's bytes. We'll have to read the bytes to decode instructions and pick out the right number of subsequent bytes for arguments. Also, we'll want to make use of `co_lnotab` and `co_firstlineno` to cross-reference between the bytecode and the original, disassembled code. You can find a small implementation (~300 lines) following these ideas [here](https://github.com/goldsborough/dispy).

## Outro

The one thing many people love about Python is the speed of development you get from its high-level, high-abstraction programming interface. However, every high-level programming language has a great depth of low-level implementation mysteries that give you insight into the inner workings of the language and allow you to make more educated decisions about performance-critical optimizations. That said, don't forget that there's a plethora of other languages such as modern C++, Rust, Go and even Java that will be faster than even the most optimized Python code (usually) and provide reasonably intuitive interfaces. As such, see what you just learnt as useful knowledge for rare edge cases and fun trivia to talk about with you grandmother for Sunday tea, but not an invitation to disassemble your entire code base and optimize individual instructions.

## Resources

Here some further resources and links I took inspiration and knowledge from:

* https://late.am/post/2012/03/26/exploring-python-code-objects.html
* http://www.aosabook.org/en/500L/a-python-interpreter-written-in-python.html
* http://unpyc.sourceforge.net/Opcodes.html
* https://github.com/nedbat/byterun
* http://stackoverflow.com/questions/12673074/how-should-i-understand-the-output-of-dis-dis
* https://docs.python.org/3.5/library/dis.html#python-bytecode-instructions
