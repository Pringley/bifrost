---
title: Dynamic language bridges using remote procedure call
author: Benjamin Pringle; Mukkai Krishnamoorthy
numbersections: true
bibliography: references.bib
---

**NOTE: citations forthcoming, having some trouble with the BibTex syntax for
Markdown.**

#### Abstract {-}

A simple strategy is presented for dynamically interpreting remote procedure
calls in scripting languages, resulting in the ability to transparently use
libraries from both languages in a single program. The protocol described
handles complex, nested arguments and object-oriented libraries.

We will first perform a proof-of-concept for matrix multiplication in Ruby
(`Grisbr`), showing the performance of a manual bridge to Python very similar
to our intended protocol.

Then, we will present our full general protocol (`Bifrost`). Example
implementations show two useful bridges: one between Ruby and Python, and
another between two different Python runtimes.

# Introduction

A **language bridge** allows a program written in one language to use functions
or libraries from a different language. For example, a web server written in
Ruby might use a bridge to call backend functions written in Java.

The use of languages bridges often introduces significant overhead, either in
performance or in code complexity. However, if the destination language is
faster or has better libraries, the benefits often outweigh those costs.

## Existing techniques

The concept of a language bridge is not a new one. Current techniques vary in
overhead and complexity.

### Common Intermediate Language

Languages within the same runtime are typically capable of bridging with little
to no overhead.

This is the approach used by Microsoft's .NET framework to provide language
interoperability between C#, C++, Visual Basic, and others. All code is
compiled to a common intermediate language, a shared bytecode executed by a
virtual machine.

The Java ecosystem can also be used similarly -- for example, between Jython
and Java programs.

### Foreign Function Interface

For languages with a common link to C, the Foreign Function Interface is a
powerful tool for passing types across a bridge.

RubyPython uses FFI to dynamically generate a bridge between the C
implementation of Ruby and CPython.

This is an excellent approach but quickly becomes difficult when bridging
across runtimes. For example, Java uses JNI (Java Native Interface) for foreign
function calls to C. Prior art exists trying to connect Jython and CPython, but
the restriction on types makes things difficult. Many standard libraries are
not functional, and NumPy also does not work using this technique.

### Remote Procedure Call

The most robust technique for language bridging is remote procedure call --
send requests through a channel to a server running the other language.

Excellent frameworks exist such as Apache Thrift (used extensively by
Facebook). However, these require extensive configuration for each function
that needs bridging:

```
struct UserProfile {
    1: i32 uid,
    2: string name,
    3: string blurb
}
service UserStorage {
    void store(1: UserProfile user),
    UserProfile retrieve(1: i32 uid)
}
```

## Desired features

In order to achieve the best of both worlds, we need:

-   **Cross-runtime**: we may want to use libraries that are compiled in
    different bytecodes -- for example, mix Java and C libraries

-   **Dynamic**: generate the bindings for *any* library without tedious "glue
    code" like Thrift

We aim to do this by using:

-   **Remote procedure call** over UNIX pipes to cross runtimes
-   **Language introspection** to eschew configuration files

The following chapter presents a proof of concept called `Grisbr`, a Ruby
library that performs matrix multiplication by sending requests to a Python
server. This demonstrates the first part of the scheme -- the remote procedure
call using JSON over UNIX pipes. We evaluate the performance of this technique
for multiplication of large and small matrices.

The chapter after that presents `Bifrost`, a generalized version of `Grisbr`
that can load any module or package, generally handle most functions, and even
use foreign language objects in the client space.

# `Grisbr` -- matrix multiplication proof of concept

In Ruby, matrix multiplication is done via the
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
be very slow. For example, multiplying 512x512 matrices takes over 45 seconds
on a consumer laptop:

    $ cd ruby
    $ time ruby mm-native.rb ../inputs/512.txt
    ruby mm-native.rb ../inputs/512.txt  45.45s user 0.04s system 99% cpu 45.502 total

To speed this up, we could write the function in C, which Ruby supports, but
this is both tedious and error-prone.

Ruby has a fast *and* high-level matrix implementation in the works -- it's
called [NMatrix](http://sciruby.com/nmatrix/) from
[SciRuby](http://sciruby.com/). Unfortunately, this is still alpha software.

On the other hand, Python has [NumPy](http://www.numpy.org/), which is both
stable and feature-rich.

This proof-of-concept shows that the use of a Ruby-to-Python bridge is a
feasible method for matrix multiplication. The bridge uses
[JSON](http://www.json.org/)-encoded calls over POSIX pipes for communication
between the Ruby process and a forked Python process.

Our example demonstrates a 30x speedup from native Ruby, reducing the runtime
on the 512 by 512 down to just a second and a half:

    $ time ruby mm-grisbr.rb ../inputs/512.txt
    ruby mm-grisbr.rb ../inputs/512.txt  1.32s user 0.18s system 101% cpu 1.480 total

## Implementation Overview

The Ruby side of the bridge encodes a request as a JSON string with named
arguments. Ruby then forks a Python process with the Python receiver code and
sents the JSON request to the Python process's standard input pipe. After
blocking on computation, Ruby receives the JSON-encoded result back from the
Python process's standard output pipe. This result is then decoded and returned
to the main program.

```ruby
require 'open3'

module Grisbr
  def self.multiply a, b
    # Convert arguments to JSON.
    args = JSON.generate({a: a, b: b})

    # Fork a process running the Python receiver server, sending the function
    # parameters via stdin.
    result, status = Open3.capture2("python grisbr-receiver.py",
                                    stdin_data: args)
    # Parse the response JSON and return.
    JSON.parse(result)["c"]
  end
end
```

On the Python side, the JSON-encoded is read on standard input. The arguments
are transformed into NumPy arrays, multiplied, then printed back to standard
output in JSON form.

```python
from json import loads, dumps
from numpy import array, dot
from sys import stdin, stdout

# Load request JSON from stdin.
args = loads(stdin.read())

# Convert to NumPy types and multiply matrices.
a, b = map(array, (args['a'], args['b']))
c = dot(a, b)

# Encode the result in JSON and write to stdout.
result = dumps({'c': c.tolist()})
stdout.write(result)
stdout.flush()
```

The bridge is transparent to the library user, who simply uses the `Grisbr`
module:

```ruby
# From Ruby
require 'grisbr'
a = [[1, 2], [3, 4]]
b = [[4, 3], [2, 1]]
p Grisbr.multiply(a, b)
# => [[8, 5], [20, 13]]
```

## Results for `Grisbr`

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

On the 512 by 512 matrix, we saw a 30x speedup using the bridge!

However, on that same matrix, the bridge is still 6 times slower than just
using straight numpy.

Also, the time spent marshalling JSON data between processes is wasted. This
means on smaller matricies, native Ruby beats the bridge.

The next chapter presents `Bifrost`, which builds on the success of this one,
generalizing the bridging technique in two ways:

-   Use **dynamic introspection** to automatically support (almost) any
    library, rather than hard-coding the request server.

-   Use **object proxies** to allow for intermediate results.

# `Bifrost` -- General Protocol

Our full protocol will take the result from `Grisbr` and combine it with
introspection and object proxies for a general solution.

## JSON over IPC

The basic idea remains the same. The client will create a subprocess running a
server in the destination language which will receive requests over JSON.

The protocol is inspired by [JSON-RPC](http://json-rpc.org/), but it is not
compatible. (Changes were made to fit the requirements of this project.)

For example, to request `numpy.mean`, a NumPy function to calculate the mean of
an array of numbers from 1 to 4, the request might look like:

```javascript
{
    "method": "mean",
    "oid": 42,
    "params": [ [1, 2, 3, 4] ]
}
```

The request has three fields:

-   **`method`** is the name of the method to call.

-   **`oid`** is the identifier for the destination-language object that
    contains the method. In this case, `42` is the object ID (`oid`) for NumPy.
    (More on object IDs in the upcoming section on object proxies.)

-   **`params`** is an array of parameters for the method. In this case, there
    is only one parameter, the array of numbers that we want the mean for
    (`[1, 2, 3, 4]`).

The subprocess server receives this request on standard input, processes it,
and responds on standard output:

```javascript
{ "result": 2.5 }
```

The protocol also handles basic error reporting. For example, if a non-existing
method is requested:

```javascript
{
    "method": "asdfasdfasdf",
    "oid": 42,
    "params": []
}
```

The server should notice the error and respond with a message:

```javascript
{ "error": "method \"asdfasdfasdf\" does not exist" }
```

## Dynamic introspection

The `Grisbr` implementation hard-coded the server's reaction to a response --
instead, we will dynamically call a function based on the request.

Scripting languages typically have some form of introspection or
metaprogramming. In Python, the relevant function is the builtin `getattr`.
From the Python 3.3.4 documentation:

> **getattr**(*object*, *name*[, *default*])
>
> > Return the value of the named attribute of object. name must be a string.
> > If the string is the name of one of the object’s attributes, the result is
> > the value of that attribute. For example, getattr(x, 'foobar') is
> > equivalent to x.foobar. If the named attribute does not exist, default is
> > returned if provided, otherwise AttributeError is raised.

A Python server can use this to dynamically call a requested function. The
implementation might look something like this (ignoring any error processing
and other details):

```python
def handle(request_json):
    # Extract information from request.
    request = json.loads(request_json)
    parameter_list = request['params']
    method_name = request['method']
    object_id = request['oid']

    # Look up object via ID from internal table.
    obj = object_table[object_id]

    # Use python's introspective `getattr` to look up the method name.
    method = getattr(obj, method_name)

    # Call the method with the supplied parameters and return the result.
    result = method(*parameter_list)
    return result
```

## Object Proxies

Most libraries for modern scripting languages are more than just a flat module
of functions -- they make heavy use of objects, both to store state and to
group methods. This causes an issue for typical remote procedure call schemes
that assume the flat module paradigm.

In order to handle this, we assign each object stored on the server a unique
identifer called its `oid` (object ID).

For example, to create a NumPy n-dimensional array object, one could send the
following request:

```javascript
{
    "method": "ndarray",
    "oid": 42,
    "params": [ [[1, 2], [3, 4]] ]
}
```

This returns an **object proxy** -- essentially just an empty container with
the object ID it represents.

```javascript
{ "result": {"__oid__": 64} }
```

Then the client can use this new object ID to call methods on that object. For
example, to request the transpose of this new matrix, simply use the proxy as a
parameter for `numpy.transpose`:

```javascript
{
    "method": "transpose",
    "oid": 42,
    "params": [ {"__oid__": 64} ]
}
```

The response will be yet *another* object proxy representing the transposed
matrix.

```javascript
{ "result": {"__oid__": 144} }
```

To actually view the transposed matrix, we can call the `ndarray.tolist()`
function on the object proxy itself (#144, the transposed matrix):

```javascript
{
    "method": "tolist",
    "oid": 144,
    "params": []
}
```

The response will be our transposed matrix in native list types:

```javascript
{ "result": [[1, 3], [2, 4]] }
```

### Implementation

In order to implement transparent object proxies, metaprogramming comes to the
rescue once again. This time, it's Ruby's method\_missing:

> **method\_missing(symbol[, \*args]) -> result**
>
> Invoked by Ruby when obj is sent a message it cannot handle. symbol is the
> symbol for the method called, and args are any arguments that were passed to
> it. By default, the interpreter raises an error when this method is called.
> However, it is possible to override the method to provide more dynamic
> behavior.

We can create a proxy object by overriding method\_missing to pass the request
down to the server for the other language. A simplified example is shown below:

```ruby
class RuBifrost::Proxy
  attr_reader :oid
  def initialize oid
    # Create a proxy for an object with specific ID
    @oid = oid
  end
  def method_missing(method, *params)
    # Forward all method calls to the server
    send_request @oid, method, params
  end
end
```

The proxy approach even generalizes nicely to modules themselves. We define a
new message used to request a module:

```javascript
{ "module": "numpy" }
```

The response is then simply an object proxy representing the module.

```javascript
{ "result": {"__oid__": 42} }
```

This elegantly solves any namespacing issues between modules without any
additional code.

## Full protocol example -- matrix multiplication

The following exchange shows importing the `numpy` module, multiplying two
matrices, and retrieving the result from the returned object proxy.

1.  **Import the module**

    Request: ask for the module to be loaded

    ```javascript
    { "module": "numpy" }
    ```

    Response: an object proxy representing the module

    ```javascript
    { "result": {"__oid__": 42} }
    ```

2.  **Multiply the matrices.**

    Request: supply to matrices for multiplication by NumPy

    ```javascript
    {
        "method": "dot",
        "oid": 42,
        "params": [
            [[1, 2], [3, 4]],
            [[4, 3], [2, 1]]
        ]
    }
    ```

    Response: an object proxy representing the result of the matrix
    multiplication

    ```javascript
    { "result": {"__oid__": 101} }
    ```

3.  **Extract native types from object proxy**

    Since `numpy.dot` returns a `numpy.ndarray` instead of a native array of
    arrays, we need to use `numpy.ndarray.tolist()` to get those native types.

    Request: the list-of-lists versions of the final matrix

    ```javascript
    {
        "method": "tolist",
        "oid": 101,
        "params": [
            {"__oid__": 101}
        ]
    }
    ```

    Response: the final multiplied matrix in native Ruby lists

    ```javascript
    { "result": [[8, 5], [20, 13]] }
    ```

# Conclusion

The presented `Bifrost` protocol can dynamically and effectively generate
cross-language bindings for arbitrary libraries, even ones that make heavy use
of objects.

This is applicable in many situations:

1.  **Performance**

    Our original example is performance-based -- Ruby lacks sufficiently
    performant libraries for linear algebra. Bifrost allows the use of NumPy in
    Ruby, and the speed gains from the faster library far outweigh the data
    marshalling costs for all but the smallest matrices.

2.  **Implementations**

    Another powerful Python library is `NetworkX`, which has implementations
    for hundreds of common graph algorithms. Its performance is not
    particularly better than Ruby, but using it would save the time of
    reimplementing hundreds of algorithms.

    Object proxies allow very simple use of NetworkX in Ruby:

    ```ruby
    require 'rubifrost'

    # Establish connection to a Python bifrost server.
    python = RuBifrost.python

    # Methods that return objects (such as networkx.Graph) instead return object
    # proxies, which in turn have all the methods of the object.
    NetworkX = python.import 'networkx'
    graph = NetworkX.Graph()
    graph.add_edges_from([
      [1, 2], [1, 3], [2, 3],
      [5, 6], [5, 8], [6, 7]
    ])

    # Object proxies can also be used as arguments for other methods. In addition,
    # complicated nested objects can be returned as results.
    p NetworkX.connected_components(graph)
    # => [[8, 5, 6, 7], [1, 2, 3]]o``
    ```

3.  **Cross-runtime**

    Jython is a port of Python to the Java runtime, which allows it to natively
    interface with Java libraries. For an enterprise codebase written primarily
    in Java, this is extremely convenient.

    However, Jython cannot use any C or FORTRAN based libraries -- including
    NumPy! If we want to perform complicated linear algebra in Jython, we would
    formerly be stuck using JNI (Java Native Interface), which requires lots of
    glue code.

    Instead, Bifrost allows a quick bridge between the Java and C runtimes,
    dynamically connecting Jython to CPython.

    An extract from the source of a Jython Bifrost client is shown below:

    ```python
    from java.io import BufferedReader, InputStreamReader, BufferedWriter, OutputStreamWriter
    from java.lang import ProcessBuilder
    bifrost_cmd = ['python3', '-mpybifrost.server']
    process = ProcessBuilder(bifrost_cmd).start()
    input_stream = process.getInputStream()
    self.stdin = BufferedWriter(OutputStreamWriter(process.getOutputStream()))
    self.stdout = BufferedReader(InputStreamReader(process.getInputStream()))
    ```

    (The remainder is available in the appendix, but it is very similar to the
    Ruby client.)

    This allows us to easily use NumPy from Jython:

    ```python
    bridge = cpython()

    numpy = bridge.import_module('numpy')
    print numpy.mean([1,2,3,4,5])
    # => 3.0
    ```

## Future improvements

1.  **Cross-platform**.

    The current implementation is very UNIX-specific, using forked processes
    and pipes. Instead, once could use flat files, databases, HTTP, or a
    message bus of some sort as the transport layer for the JSON requests.

2.  **Method caching**.

    The current implementation sends a request to the server for every method
    call. It might be possible to cache some results (such as `length` for an
    array) to avoid overloading the server. This may require some glue code.

3.  **Optional glue code**.

    The protocol could be extended with user-defined flags to allow custom glue
    code where performance or implemenation quirks require them.
