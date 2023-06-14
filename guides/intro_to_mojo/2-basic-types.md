---
title: '2: Basic Types'
category: '1: Setup'
usage: Some basic types to get you started with Mojo
---

# Basic Types
_This is in very early stages and under heavy development_

## PythonObject
Run code through the Python interpreter to get a [PythonObject](https://docs.modular.com/mojo/MojoPython/PythonObject.html) back:


```mojo
x = Python.evaluate('5 + 10')
print(x)
```

    15


The first line is executed the exact same way as if we ran this in Python:

```python
x = 5 + 10
```

The code is parsed and compiled to byte code, then CPython executes the instruction, `x` is actually a pointer to `heap` allocated memory.

::: tip CS Fundamentals
`stack` and `heap` memory are really important concepts to understand, [this YouTube video](https://www.youtube.com/watch?v=_8-ht2AKyH4) does a fantastic job of explaining it visually. 

If the video doesn't make sense, for now you can use the mental model that:

- `stack` memory is very fast but small, the size of the values must be known at runtime
- `pointer` is an address to lookup the value somewhere else in memory
- `heap` memory is huge and the size can change at runtime, but needs a pointer to access the data which is slow

We'll be revisiting these concepts a lot, don't worry if it's not clicking yet.
:::

We can access all the Python keywords by importing `builtins`:


```mojo
let py = Python.import_module("builtins")

py.print("using python keywords")
```

    using python keywords


We can now use the `type` builtin from Python to see what the dynamic type of `x` was:


```mojo
py.print(py.type(x))
```

    <class 'int'>


We can also read the address that is stored on the `stack` which allows us to read memory on the `heap` with the Python `id` builtin:


```mojo
py.print(py.id(x))
```

    140411432053024


This is pointing to a C object in Python, and Mojo behaves the same when using a `PythonObject`, accessing the value actually uses the address to lookup the data on the `heap` which comes with a performance cost. 

Lets dive into this a little more.

## Stack
This is the fast access section of memory that is allocated to your computers RAM, take a simple program:

_the `%%python` below means the cell runs through the Python interpreter_


```mojo
%%python
def double(a):
    return a * 2 

def quad(a):
    return a * 4 

a = 1

a = double(a)
a = quad(a)
```

If we represent the instructions in pseudo code, this is a simplified version of what your `stack` memory would look like as the program runs:


```mojo
%%python
from pprint import pprint

stack = []
stack.append({"frame": "main", "a": 1, "function_calls": ["double(a)", "quad(a)"]})
stack.append({"frame": "add", "a": 1})

pprint(stack)
```

    [{'a': 1, 'frame': 'main', 'function_calls': ['double(a)', 'quad(a)']},
     {'a': 1, 'frame': 'add'}]


The program starts by allocating variables from `main` to the `stack` memory, the first function is `add` so it is then appended to the stack.

When it's finished running and returns the result, all the variables in `add` are popped off the stack:


```mojo
%%python
stack.pop()
stack[0]["a"] *= 2
print(stack)
```

    [{'frame': 'main', 'a': 2, 'function_calls': ['double(a)', 'quad(a)']}]


This is why a `stack` is called Last In First Out (LIFO), because the `last` function to be allocated is the first one `out` of the stack.

The next function calls `quad` and the variable is appended to the stack memory and runs, it then returns while updating `a` and being popped off the stack, then `main` is popped off the stack as there are no more instructions to run which ends the program:


```mojo
%%python
stack.append({"frame": "quad", "a": 2})
stack[0]["a"] *= 4
stack.pop()
stack.pop()
print(stack)
```

    []


## Heap

The Heap memory is huge, it can use the remainder of the available RAM on your OS, Python uses it for every object to provide us with conveniences, `a` in the previous example doesn't actually contain the value `1` at the start of the program, it contains an address to another place in memory on the heap:


```mojo
%%python

heap = {
    44601345678945: {
        "type": "int",
        "ref_count": 1,
        "size": 1,
        "digit": 8,
        #...
    }
}
```

So on the stack `a` looks more like this for each frame:


```mojo
%%python

[
    {"frame": "main", "a": 44601345678945 }
]
```

`a` contains an address that is pointing to the heap object

In Python we can change the type like:


```mojo
a = "mojo"
```

The object in C will change its representation:


```mojo
%%python
heap = {
    "a": {
        "type": "string",
        "ref_count": 1,
        "size": 4,
        "ascii": True,
        # utf-8 / ascii for "mojo"
        "value": [109, 111, 106, 111]
        # ...
    }
}
```

This allows Python to do nice convenient things for us
- once the `ref_count` goes to zero it will be de-allocated from the heap during garbage collection, so the OS can use that memory for something else
- an integer can grow beyond 64 bits by increasing `size`
- we can dynamically change the `type`
- the data can be large or small, we don't have to think about when we should allocate to the heap

However this also comes with a penalty, there is a lot more extra memory being used for the extra fields, and it also takes CPU instructions to allocate the data, retrieve it, garbage collect etc.

In Mojo we can remove all that overhead:

## Mojo 🔥


```mojo
x = 5 + 10
print(x)
```

    15


We've just unlocked our first Mojo optimization! Instead of looking up an object on the heap via an address, `x` is now just a value on the stack with 64 bits that can be passed through registers.

This has numerous performance implications:

- All the expensive allocation, garbage collection, and indirection is no longer required
- The compiler can do huge optimizations when it knows what the numeric type is
- The value can be passed straight into registers for mathematical operations
- The data can now be packed into a vector for huge performance gains

That last one is very important in today's world, let's see how Mojo gives us the power to take advantage of modern hardware.

## SIMD

SIMD stands for `Single Instruction, Multiple Data`, hardware now contains special registers that allow you do the same operation in a single instruction, greatly improving performance, let's take a look:


```mojo
from DType import DType

y = SIMD[DType.uint8, 4](1, 2, 3, 4)
print(y)
```

    [1, 2, 3, 4]


In the definition `[DType.uint8, 4]` are known as parameters which means they're compile-time known, while `(1, 2, 3, 4)` are the arguments which can be compile-time or runtime known.

This is now a vector of 8 bit numbers that are packed into 32 bits, we can perform a single instruction across all of it instead of 4 separate instructions:


```mojo
y *= 10
print(y)
```

    [10, 20, 30, 40]


::: tip CS Fundamentals
Binary is how your computer stores memory, with each bit representing a `0` or `1`. Memory is typically byte-addressable, meaning that each unique memory address points to one byte, which consists of 8 bits.

This is how the first 4 digits in a `uint8` are represented in hardware:

- 1 = `00000001`
- 2 = `00000010`
- 3 = `00000011`
- 4 = `00000100`

In RAM, binary `1` and `0` represent charged or uncharged capacitors, indicating ON or OFF states.

[Check this video](https://www.youtube.com/watch?v=RrJXLdv1i74) if you want more information on binary.
:::

We have room for 64 bits at each address though on a 64 bit CPU, So we're packing it into a single memory location with SIMD like this:

`00000001``00000010``00000011``00000100`

The register that can perform operations on this in modern CPU's is huge, let's see how big our SIMD register is:


```mojo
from TargetInfo import simd_bit_width
print(simd_bit_width())
```

    512


That means we could pack 64 8 bit numbers together and perform a single calculation on all of it.

## Scalars

Scalar just means a single value, you'll notice in Mojo all the numerics are SIMD scalars:


```mojo
var x = UInt8(1)
x = "will cause an error"
```

    error: Expression [17]:24:9: cannot implicitly convert 'StringLiteral' value to 'SIMD[ui8, 1]' in assignment
        x = "will cause an error"
            ^~~~~~~~~~~~~~~~~~~~~
    


UInt8 is just an alias for `SIMD[DType.uint8, 1]`, you can see all the [numeric SIMD types imported by default here](https://docs.modular.com/mojo/MojoStdlib/SIMD.html)

Also notice when we try and change the type it throws an error, this is because Mojo is `strongly typed`

If we use existing Python modules, it will give us back a `PythonObject` that behaves the same `loosely typed` way as it does in Python:


```mojo
np = Python.import_module("numpy")

arr = np.ndarray([5])
print(arr)
arr = "this will work fine"
print(arr)
```

    [0.   0.25 0.5  0.75 1.  ]
    this will work fine


## Strings
In Mojo the heap allocated string isn't imported by default:


```mojo
from String import String

s = String("Mojo🔥")
print(s)
```

    Mojo🔥


`String` works similar to a normal python object, where it's actually a pointer to data that's allocated on the `heap`, this means we can load a huge amount of data into it, and change the size of the data dynamically.

Let's cause a type error so you can see the data type underlying the String:


```mojo
x = s.buffer
x = 20
```

    error: Expression [20]:26:10: cannot implicitly convert 'DynamicVector[SIMD[si8, 1]]' value to 'PythonObject' in assignment
        x = s.buffer
            ~^~~~~~~
    


`DynamicVector` is similar to a Python list, here it's storing `int8` that represent the characters, let's print the first character:


```mojo
print(s[0])
```

    M


Now lets take a look at the decimal representation:


```mojo
from String import ord

print(ord(s[0]))
```

    77


That's the ASCII code [shown in this table](https://www.ascii-code.com/)

We can build our own string this way, we can put in 78 which is N and 79 which is O


```mojo
from Vector import DynamicVector

let vec = DynamicVector[Int8](2)

vec.push_back(78)
vec.push_back(79)
```

And use the pointer


```mojo
vec_str = String(vec.data, 2)

print(vec_str)
```

    NO


One thing to be aware of is that an emoji is actually four bytes, so we need a slice of 4 to have it print correctly:


```mojo
emoji = String("🔥😀")
print("fire:", emoji[0:4])
print("smiley:", emoji[4:8])
```

    fire: 🔥
    smiley: 😀


Check out [Maxim Zaks Blog post](https://mzaks.medium.com/counting-chars-with-simd-in-mojo-140ee730bd4d) for more details.

## Notes
You can also initialize SIMD with a single argument:


```mojo
z = SIMD[DType.uint8, 4](1)
print(z)
```

    [1, 1, 1, 1]


Or do it in a loop:


```mojo
for i in range(3):
    print(SIMD[DType.uint16, 4](i))
```

    [0, 0, 0, 0]
    [1, 1, 1, 1]
    [2, 2, 2, 2]


## Exercises
1. Create a SIMD of DType UInt8, 16 bytes wide and each value at 2, then multiply it by 8 and print it
2. Create a loop using SIMD that prints four rows of data that looks like this:
    [1,0,0,0]
    [0,1,0,0]
    [0,0,1,0]
    [0,0,0,1]

## Solutions
### Exercise 1


```mojo
print(SIMD[DType.uint8, 16](2) * 8)
```

    [16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16]


### Exercise 2


```mojo
for i in range(4):
    simd = SIMD[DType.uint8, 4](0)
    simd[i] = 1
    print(simd)
```

    [1, 0, 0, 0]
    [0, 1, 0, 0]
    [0, 0, 1, 0]
    [0, 0, 0, 1]
