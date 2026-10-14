---
description: A Client keeps one connection alive between requests, and carries the defaults they share.
---

# Clients

`curl::send` opens a handle, does one transfer and throws the handle away. For a script, that's exactly right. For a program that talks to the same API fifty times, it means fifty TCP handshakes, fifty TLS handshakes and fifty DNS lookups. [](api:curl::Client) is the fix: one handle, kept alive, with your defaults on it.

```echo
curl::Client $api = curl::Client();
$api->baseUrl = 'https://api.example.com/v1';
$api->header('Accept', 'application/json');
$api->timeout(10000);

curl::Response $users = guard $api->send(.get('users')) else ($e) {
    die("{$e}");
}
```

## A client is a connection

**A `Client` is a connection, not a namespace.** It holds one libcurl handle, and that handle is what keeps the TCP connection, the DNS answer and the TLS session alive between requests. That's the whole reason to use libcurl instead of a socket.

It's a class, not a struct, so two names for one client are two names for one connection. Hand it to a function and the function talks over the same socket. As a struct it would quietly copy the handle, and that's two connections.

`send` is the only verb. It takes the exact same `Request` the free `curl::send` takes, so moving a call from one to the other changes the receiver and nothing else:

```echo
curl::send(.get('https://api.example.com/v1/users'));
$api->send(.get('users'));
```

## Nothing leaks between requests

Here's the bug every keep-alive wrapper has had at some point: request one sets a header or an option on the shared handle, and request two inherits it. This one doesn't. Before every send, the handle is reset. That keeps the connection and drops every option. Then the client's defaults go back on, then the request's own.

So the method, the body, the headers, the options, a form and a sink from one request are all gone by the next. The client's own defaults are there every time, because they're replayed after each reset.

## Defaults

Everything a client holds is a default for the requests sent through it:

- `baseUrl`, which a relative request URL is joined onto. [How URLs join](./urls.md) has the rules, and one of them will surprise you.
- `headers`, through `header()`. A request header of the same name replaces the client's.
- Options, through the same helpers a `Request` has: `timeout`, `follow`, `verifyTls`, `userAgent`, `auth`, `proxy`, `maxBytes`, `set` and `raw`.

```echo
curl::Request $slow = .get('reports/yearly');
$slow->timeout(60000); // this one request gets a minute
```

The request always wins. [How options stack](./stacking.md) is the exact order.

## Sending through it

`$api->send($req)` answers `result<Response, Error>`, the same as `curl::send`. The [Responses](./response.md) and [Errors](./errors.md) pages apply unchanged.

`open()` is false when libcurl wouldn't hand out a handle at all. That's rare, and you don't have to check it: every `send` on such a client fails with `.failedInit`.

`examples/client.eco` sends two requests over one connection, the second with its own query, timeout and header:

```bash
echoc run -m . examples/client.eco
```
