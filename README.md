# purerpc

[![Build Status](https://img.shields.io/travis/standy66/purerpc.svg?style=flat)](https://travis-ci.org/standy66/purerpc)
[![PyPI version](https://img.shields.io/pypi/v/purerpc.svg?style=flat)](https://pypi.org/project/purerpc/)
[![Downloads](https://pepy.tech/badge/purerpc/month)](https://pepy.tech/project/purerpc)

Asynchronous pure Python gRPC client and server implementation supporting
[asyncio](https://docs.python.org/3/library/asyncio.html),
[uvloop](https://github.com/MagicStack/uvloop),
[curio](https://github.com/dabeaz/curio) and
[trio](https://github.com/python-trio/trio) (achieved with [anyio](https://github.com/agronholm/anyio) compatibility layer).

## Requirements

* CPython >= 3.5
* PyPy >= 3.5

## Installation

Latest PyPI version:

```bash
pip install purerpc
```

Latest development version:

```bash
pip install git+https://github.com/standy66/purerpc.git
```

By default purerpc uses asyncio event loop, if you want to use uvloop, curio or trio, please install them manually.

## protoc plugin

purerpc adds `protoc-gen-purerpc` plugin for `protoc` to your `PATH` enviroment variable
so you can use it to generate service definition and stubs: 

```bash
protoc --purerpc_out=. --python_out=. -I. greeter.proto
```

or, if you installed `grpcio_tools` Python package:

```bash
python -m grpc_tools.protoc --purerpc_out=. --python_out=. -I. greeter.proto
```

## Usage

NOTE: `greeter_grpc` module is generated by purerpc's `protoc-gen-purerpc` plugin.

Below are the examples for Python 3.6 and above which introduced asynchronous generators as a language concept.
For Python 3.5, where native asynchronous generators are not supported, you can use [async_generator](https://github.com/python-trio/async_generator) library for this purpose.
Just mark yielding coroutines with `@async_generator` decorator and use `await yield_(value)` and `await yield_from_(async_iterable)` instead of `yield`.

### Server

```python
from greeter_pb2 import HelloRequest, HelloReply
from greeter_grpc import GreeterServicer
from purerpc import Server


class Greeter(GreeterServicer):
    async def SayHello(self, message):
        return HelloReply(message="Hello, " + message.name)

    async def SayHelloToMany(self, input_messages):
        async for message in input_messages:
            yield HelloReply(message=f"Hello, {message.name}")


server = Server(50055)
server.add_service(Greeter().service)
server.serve(backend="asyncio")  # backend can also be one of: "uvloop", "curio", "trio"
```

### Client

```python
import anyio
import purerpc
from greeter_pb2 import HelloRequest, HelloReply
from greeter_grpc import GreeterStub


async def gen():
    for i in range(5):
        yield HelloRequest(name=str(i))


async def main():
    async with purerpc.insecure_channel("localhost", 50055) as channel:
        stub = GreeterStub(channel)
        reply = await stub.SayHello(HelloRequest(name="World"))
        print(reply.message)

        async for reply in stub.SayHelloToMany(gen()):
            print(reply.message)


if __name__ == "__main__":
    anyio.run(main, backend="asyncio")  # backend can also be one of: "uvloop", "curio", "trio"
```

You can mix server and client code, for example make a server that requests something using purerpc from another gRPC server, etc.

More examples in `misc/` folder
