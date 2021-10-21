# Python

This document looks at Python's network stack from a .NET developer's perspective.

> **Note:** This document is incomplete

# Python Sockets

Python has `async`/`await` support, “awaitables” rather than a single `Task` object, and a "coroutine" which is sort of like an API for the async state machine that .NET creates behind the scenes.

A `TaskScheduler`-like object called an `Executor` is used to run blocking calls asynchronously. There are both thread pool and process pool executors. There is no global thread pool, but rather each executor instantiated will get its own pool.

Python runs async via an `EventLoop`. An event loop is essentially a single-threaded I/O Completion Port, but is only capable of a single execution thread. There is a global event loop, but you can also create them manually. To use multiple threads, you need to create multiple event loops. Being single-threaded, blocking is not allowed and has similarly bad consequences as .NET. Blocking operations are instead farmed off to an `Executor`.

An event loop is in charge of creating a `Transport` – similar to a `System.Net.Connections.Connection`, this acts as something between an `IDuplexPipe` and `Stream`, and uses a property bag to expose implementation-specific things.

Writes to the stream are always buffered; A call to `drain()` is used to flush.

## Echo Server Example

(Note, this uses the global event loop)

```py
import asyncio

# Run for each accepted socket
async def handle_echo(reader, writer):
    data = await reader.read(100)

    message = data.decode()
    addr = writer.get_extra_info('peername')
    print(f"Received {message!r} from {addr!r}")

    print(f"Send: {message!r}")
    writer.write(data)
    await writer.drain()

    print("Close the connection")
    writer.close()

async def main():
    # Start a server and accept sockets via callback.
    server = await asyncio.start_server(
        handle_echo, '127.0.0.1', 8888)

    addr = server.sockets[0].getsockname()
    print(f'Serving on {addr}')

    async with server:
        await server.serve_forever()

# Runs the coroutine in an event loop.
asyncio.run(main())
```

# Python HTTP

Python has two built-in HTTP client APIs: `http.client` and `urllib`. Python's docs recommend a very popular third-party package, `Requests`. These are all synchronous HTTP/1.1 clients.

A popular third-party package `aiohttp` provides an async API.

## `http.client`

`http.client` has you create `HttpConnection` objects which can run one or more requests, but only sequentially. To run requests in parallel, one must create multiple connections.

> Python docs recommend considering [`Requests`](#requests) instead.

```py
import http.client

conn = http.client.HTTPSConnection("www.python.org")

conn.putrequest("GET", "/")
conn.putheader("X-My-Header: foo")
conn.endheaders()

r1 = conn.getresponse()
print(f"{r1.status}: {r1.reason}")

print("headers:")
for h in r1.headers:
    print(f"{h.header}: {h.value}")

while chunk := r1.read(200):
    print(repr(chunk))

conn.close()
```

## `urllib.request`

`urllib.request` is a layer ontop of `http.client` which provides high-level functionality similar to .NET's `HttpClient`.

It also supports FTP and is abstracted similar to `RequestMessage`.

It does not perform connection pooling, and adds `Connection: close` to all requests.

There is a concept of a `HttpMessageHandler` called "openers" and "handlers" which uses layering to add features like authentication and proxies.

> Python docs recommend considering [`Requests`](#requests) instead.

```py
import urllib.request

headers = {"X-My-Header": "foo"}

with urllib.request.urlopen("https://www.python.org", headers) as r1:
    print(f"{r1.status}: {r1.reason}")

    print("headers:")
    for h in r1.headers:
        print(f"{h.header}: {h.value}")

    while chunk := r1.read(200):
        print(repr(chunk))
```

## `Requests`

`Requests` has various high-level features similar to .NET's `HttpClient`.

`Requests` uses cookies by default.

It has simple APIs like `requests.get` which do not use pooling:

```py
import requests

headers = {"X-My-Header": "foo"}

with requests.get("http://www.python.org", headers=headers, stream=True) as r1:
    print(f"{r1.status_code}: {r1.reason}")

    print("headers:")
    for name in r1.headers:
        print(f"{name}: {r1.headers[name]}")

    for chunk in r1.iter_content(chunk_size = 4096)
        print(repr(chunk))
```

It also has a session API that looks more in line with `HttpClient` doing pooling, cookies, proxy config, default headers, etc.:

```py
import requests

with requests.Session() as s:
    s.auth = ('user', 'pass')
    s.headers.update({"X-My-Default-Header": "bar"})

    headers = {"X-My-Header": "foo"}

    with s.get("http://www.python.org", headers=headers, stream=True) as r1:
        print(f"{r1.status_code}: {r1.reason}")

        print("headers:")
        for name in r1.headers:
            print(f"{name}: {r1.headers[name]}")

        for chunk in r1.iter_content(chunk_size = 4096)
            print(repr(chunk))
```

## aiohttp

`aiohttp` has more or less the same high-level features of [`Requests`](#requests), but now async.

`aiohttp` uses cookies by default.

```py
import aiohttp
import asyncio

async def main():

    async with aiohttp.ClientSession() as s:
        async with s.get('http://www.python.org') as r1:
            print(f"{r1.status}: {r1.reason}")

            print("headers:")
            for name in r1.headers:
                print(f"{name}: {r1.headers[name]}")

            while True:
                chunk = await resp.content.read(chunk_size)
                if not chunk:
                    break
                print(repr(chunk))

loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

It supports WebSockets:

```py
async with session.ws_connect('http://example.org/ws') as ws:
    async for msg in ws:
        if msg.type == aiohttp.WSMsgType.TEXT:
            if msg.data == 'close cmd':
                await ws.close()
                break
            else:
                await ws.send_str(msg.data + '/answer')
        elif msg.type == aiohttp.WSMsgType.ERROR:
            break
```

It allows getting redirect history of a request. This is interesting because .NET's `HttpClient` has no way to get this history without turning auto-redirects off and performing them manually.

```py
async with s.get('http://www.python.org') as r1:
    assert r1.history[0].status == 301
    assert r1.history[0].url = URL(
        'http://example.com/some/redirect/')
    assert r1.history[1].status == 301
    assert r1.history[1].url = URL(
        'http://example.com/some/other/redirect/')
```

It has a concept that combines `System.Net.Connections` with connection pooling, called a `Connector`.

The `TCPConnector` class has a built-in DNS cache, connects TLS if needed, and controls connection pool sizes both in terms of total connections and total connections per host. There is also a UDS and named pipe connector.

```py
resolver = AsyncResolver(nameservers=["8.8.8.8", "8.8.4.4"])
c = aiohttp.TCPConnector(limit=30, limit_per_host=30, resolver=resolver, ttl_dns_cache=300)

async with aiohttp.ClientSession(connector=c) as c:
    # ...
```

# Conclusion

