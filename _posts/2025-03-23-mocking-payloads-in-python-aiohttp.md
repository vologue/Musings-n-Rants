---
layout: post
title:  "Mocking payloads in python AIOHTTP"
date:   2025-03-23 17:07:40 +0530
tags: [ python, code, testing ]
categories: ["engineering"]
---

The Python AIOHTTP library provides an extensive testing library. However, writing unit tests for middlewares and 
handlers can still be challenging, particularly in cases where middlewares behave differently based on the request's 
body.

For these situations, the official recommendation is to write integration tests/end-to-end tests. However, I find the 
feedback loop from e2e tests to be too slow and integration tests require extensive setup. Searching online, I found a 
lot of places where people have implemented `FakeRequest` class, which would implement the required methods. This is a 
valid and sometimes even an elegant way to write tests. But I wanted to use the 
`make_mocked_request` util offered to us in `aiohttp.web_exception`. This is how I did it.

To provide clarity on my approach, I will first explain what `make_mocked_request` does and how it can be utilized 
effectively.

Following is the function definition from the [aiohttp repo][aio-http-repo]

```python
def make_mocked_request(
    method: str,
    path: str,
    headers: Optional[LooseHeaders] = None,
    *,
    match_info: Optional[Dict[str, str]] = None,
    version: HttpVersion = HttpVersion(1, 1),
    closing: bool = False,
    app: Optional[Application] = None,
    writer: Optional[AbstractStreamWriter] = None,
    protocol: Optional[RequestHandler[Request]] = None,
    transport: Optional[asyncio.Transport] = None,
    payload: StreamReader = EMPTY_PAYLOAD,
    sslcontext: Optional[SSLContext] = None,
    client_max_size: int = 1024**2,
    loop: Any = ...,
) -> Request:
        ...
```

From this we see the `make_mocked_request` function takes a parameter of type `aiohttp.streams.StreamReader` and by default is set to the value 
`_EMPTY_PAYLOAD` which intern is nothing but:

> In older versions, it takes a parameter of type `Any` and by default is set to the value`sentinel` which in turn is nothing but:
> ```python 
> sentinel = enum.Enum(value="_SENTINEL", names="sentinel").sentinel
> ```
We come to the same StreamReader conclusion in the older versions by seeing the `BaseRequest` implementation. 

This tells us that all we need is to pass a StreamReader object that will return the data to be mocked when the `read` 
method is called. So essentially this is what I did:

```python
from aiohttp import StreamReader
from aiohttp.test_utils import make_mocked_request
from unittest.mock import Mock

def get_mock_payload(body: bytes):
    stream_reader = StreamReader(protocol=Mock(), limit=2**16)
    stream_reader.feed_data(body)
    return stream_reader


mock_request = make_mocked_request(
    path='/',
    method='POST',
    payload=get_mock_payload(b'test payload data.')
)
```

Have fun mocking payloads for handlers and middlewares.



[aio-http-repo]: https://github.com/aio-libs/aiohttp/blob/b6f34d4b27ffc45c138bdba428f6e1a5cf9367e4/aiohttp/test_utils.py#L609C1-L625C14

