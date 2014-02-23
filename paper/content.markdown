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

We will first perform a proof-of-concept for matrix multiplication in Ruby
(`GrIsBr`), showing the performance of a manual bridge to Python very similar
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

-   cross-runtime (mix Java and C libraries)
-   dynamic (no tedious "glue code" like Thrift)

We aim to do this by using:

-   remote procedure call to link langauges
-   dynamic language introspection to eschew configuration files

# Proof of concept -- matrix multiplication

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
    args = JSON.generate({a: a, b: b})
    result, status = Open3.capture2("python grisbr-receiver.py",
                                    stdin_data: args)
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

args = loads(stdin.read())
a, b = map(array, (args['a'], args['b']))
c = dot(a, b)
result = dumps({'c': c.tolist()})
stdout.write(result)
stdout.flush()
```

The bridge is transparent to the library user, who simply uses the `Grisbr`
module:

```ruby
a = [[1, 2], [3, 4]]
b = [[4, 3], [2, 1]]
p Grisbr.multiply(a, b)
# => [[8, 5], [20, 13]]
```

## Results

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

# Improved Protocol -- Bifrost

Our full protocol will take the result from `Grisbr` and combine it with
introspection and object proxies for a full solution.

## JSON over IPC

The basic idea remains the same. Client:

-   Fork server process in other language

-   Send requests as JSON

## Dynamic introspection

The `Grisbr` implementation hard-coded the server's reaction to a response --
instead, we will dynamically call a function based on the request.

The basic idea (with error handling and other details simplified/removed):

```python
def handle(self, request):
    params = request['params']
    method_name = request['method']
    # Use python's introspective `getattr` to look up the method name
    method = getattr(obj, method_name)
    result = method(*params)
    return result
```

## Object Proxies

Object proxies pass method calls over to the other language.

```ruby
class RuBifrost::Proxy
  attr_reader :oid
  def initialize oid, bifrost
    @oid = oid
    @bifrost = bifrost
  end
  def method_missing(method, *params)
    @bifrost.callmethod @oid, method, params
  end
end
```

On the Python end, the server maintains a dictionary of objects:

```python
def getref(self, obj):
    oid = id(obj)
    if oid not in self.objs:
        self.objs[oid] = obj
    return {'__oid__': oid}

def deref(self, obj):
    oid = obj['__oid__']
    return self.objs[oid]
```

This allows us to use libraries that store state as objects, such as NetworkX.

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
# => [[8, 5, 6, 7], [1, 2, 3]]
```

## Implementation

A Bifrost client can be implemented in less than a hundred lines of code.

Here is a client for Jython, along with test cases:

```python
import simplejson as json
from java.io import BufferedReader, InputStreamReader, BufferedWriter, OutputStreamWriter
from java.lang import ProcessBuilder

def main():
    python = cpython()

    numpy = python.import_module('numpy')
    print numpy.mean([1,2,3,4,5])

    networkx = python.import_module('networkx')
    graph = networkx.Graph()
    graph.add_edges_from([
        [1, 2], [1, 3], [2, 3],
        [5, 6], [5, 8], [6, 7]
    ])

    print(networkx.connected_components(graph))

def cpython():
    return PyBifrost(['python3', '-mpybifrost.server'])

class Proxy(object):

    def __init__(self, oid, bifrost):
        self.oid = oid
        self.bifrost = bifrost

    def __getattr__(self, name):
        def method_proxy(*args):
            return self.bifrost.callmethod(self.oid, name, args)
        return method_proxy

class PyBifrost:

    def __init__(self, bifrost_cmd):
        process = ProcessBuilder(bifrost_cmd).start()
        input_stream = process.getInputStream()
        self.stdin = BufferedWriter(OutputStreamWriter(process.getOutputStream()))
        self.stdout = BufferedReader(InputStreamReader(process.getInputStream()))

    def import_module(self, module_name):
        return self.get_result({'module': module_name})

    def callmethod(self, oid, method, params):
        return self.get_result({
            'oid': oid,
            'method': method,
            'params': [self.deproxify(param) for param in params]
        })

    def get_result(self, request):
        message = json.dumps(request) + '\n'
        self.stdin.write(message)
        self.stdin.flush()
        line = self.stdout.readLine()
        result = json.loads(line)
        if 'error' in result:
            raise RuntimeError(result['error'])
        if 'result' not in result:
            raise RuntimeError('no result')
        return self.proxify(result['result'])

    def deproxify(self, obj):
        if isinstance(obj, (int, float, str, bool)) or obj is None:
            return obj
        if isinstance(obj, Proxy):
            return {'__oid__': obj.oid}
        if isinstance(obj, dict):
            return dict([(self.deproxify(key), self.deproxify(val))
                    for key, val in obj.items()])
        if isinstance(obj, list):
            return [self.deproxify(item) for item in obj]

        raise RuntimeError('unexpected obj for deproxify')

    def proxify(self, obj):
        if isinstance(obj, (int, float, str, bool)) or obj is None:
            return obj
        if isinstance(obj, dict):
            if '__oid__' in obj:
                return Proxy(obj['__oid__'], self)
            return dict([(self.proxify(key), self.proxify(val))
                    for key, val in obj.items()])
        if isinstance(obj, list):
            return [self.proxify(item) for item in obj]

        raise RuntimeError('unexpected obj for proxify')

if __name__ == '__main__':
    main()
```

The server-side is more complicated, see <http://github.com/Pringley/pybifrost>.

# Conclusion

Future improvements:

-   Cross-platform: use flat files or database or message bus of some sort
    instead of pipes.

-   Cache method results: save time on obj.len
