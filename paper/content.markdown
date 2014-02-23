---
title: Dynamic language bridges using remote procedure call
author: Benjamin Pringle; Mukkai Krishnamoorthy
numbersections: true
bibliography: references.bib
---

#### Abstract {-}

A simple strategy is presented for dynamically interpreting remote procedure
calls in scripting languages, resulting in the ability to transparently use
libraries from both languages in a single program. The protocol described
handles complex, nested arguments and object-oriented libraries.

Example implementations show two useful bridges: one between Ruby and Python,
and another between two different Python runtimes.

# Introduction

A **language bridge** allows a program written in one language to use functions
or libraries from a different language. For example, a web server written in
Ruby might use a bridge to call backend functions written in Java.

The use of languages bridges often introduces significant overhead, either in
performance or in code complexity. However, if the destination language is
faster or has better libraries, the benefits often outweigh those costs.

For example, in Ruby, matrix multiplication is done via the
[Matrix#*](http://www.ruby-doc.org/stdlib-2.0.0/libdoc/matrix/rdoc/Matrix.html#method-i-2A)
method:

```ruby
require 'matrix'
a = Matrix[[1, 2], [3, 4]]
b = Matrix[[4, 3], [2, 1]]
puts a * b
# => Matrix[[8, 5], [20, 13]]
```

This works for small matrices, but since it is implemented in pure Ruby, it can
be very slow. For example, multipling 256x256 matrices:

    $ cd ruby
    $ time ruby mm-native.rb ../inputs/512.txt
    ruby mm-native.rb ../inputs/512.txt  45.45s user 0.04s system 99% cpu 45.502 total

How can we speed this up?

We could write the function in C, which Ruby supports, but this is both tedious
and error-prone. We're using a scripting language for a reason!

Fortunately, Ruby has a fast *and* high-level matrix implementation in the
works -- it's called [NMatrix](http://sciruby.com/nmatrix/) from
[SciRuby](http://sciruby.com/). Unfortunately, this is still alpha software.

On the other hand, Python has [NumPy](http://www.numpy.org/), which is both
stable and feature-rich.

For a quick project, it would be really nice to use NumPy in Ruby.

## Existing techniques

The concept of a language bridge is not a new one. Current techniques vary in
overhead and complexity.

### Common Intermediate Language

Languages within the same runtime are typically capable of bridging with little
to no overhead. For example, a Jython program can use Java methods and objects.

### Foreign Function Interface

### Remote Procedure Call

## Desired features

-   cross-runtime (mix Java and C libraries)
-   dynamic (no tedious "glue code")

# Protocol

## JSON over IPC

Client:

-   Fork server process in other language

    In Ruby:

    ```ruby
    command = 'python3 -mpybifrost.server'
    pipes = Open3.popen3(command)
    ```

## Dynamic introspection

## Object Proxies

# Implementation

On the Ruby (client) side,

```ruby
def get_result request
  @stdin.write JSON.generate(request) + "\n"
  @stdin.flush
  line = @stdout.gets
  result = JSON.parse line
  raise RuntimeError, result['error'] if result.has_key? 'error'
  raise RuntimeError, 'no result' if not result.has_key? 'result'
  proxify result['result']
end
```

On the Python (server) side,

```python
def handle_json(self, json_line):
    try:
        request = json.loads(json_line)
    except ValueError:
        return json.dumps({"error": "could not parse json"})
    response = self.handle(request)
    return json.dumps(response)
```

## Ruby to Python

## Jython to CPython

# Analysis

On the 512 by 512 matrix, we saw a 30x speedup using the bridge!

However, on that same matrix, the bridge is still 6 times slower than just
using straight numpy:

    $ cd py
    $ time python mm-numpy.py ../inputs/512.txt 
    python mm-numpy.py ../inputs/512.txt  0.25s user 0.04s system 100% cpu 0.285 total

Also, the time spent marshalling JSON data between processes is wasted. This
means on smaller matricies, native Ruby beats the bridge.

The table below shows a breakdown of runtimes for native Ruby, the bridge, and
straight NumPy on matrices with various sizes:

    +--------+-------+---------+---------+
    |        | 2x2   | 128x128 | 512x512 |
    +========+=======+=========+=========+
    | Native |  .08s |    .79s |  45.50s |
    +--------+-------+---------+---------+
    | Bridge |  .19s |    .27s |   1.48s |
    +--------+-------+---------+---------+
    | Numpy  |  .09s |    .10s |    .28s |
    +--------+-------+---------+---------+

## Edge cases

## Speed

# Conclusion
