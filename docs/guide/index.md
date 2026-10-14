---
description: Install libcurl, send a GET, and read what came back.
---

# Getting started

You want to talk HTTP from Echo, and you don't want to write a TLS stack to do it. libcurl is already on basically every machine, it speaks every protocol you've heard of, and it has been debugged by the entire internet for 25 years. This library is that C library, wrapped so it reads like Echo.

The namespace is `curl`. The whole thing is written in Echo: the callbacks libcurl calls back into are ordinary Echo functions handed to C, so there's no C shim to compile.

## Install it

From your project directory:

```bash
epm add echolang/libcurl --git https://github.com/echolang/libcurl --range ^1.0
```

That's the Echo side. The C side is the libcurl on your machine, and on macOS that's already done. Linux and Windows need one more step, and [Finding libcurl](./install.md) has it.

## A complete pass

Let's fetch a page. There is exactly one way to make a request here: build a [](api:curl::Request), then hand it to [](api:curl::send).

```echo
curl::Request $req = .get('https://echoc.dev');

curl::Response $res = guard curl::send($req) else ($e) {
    die("request failed: {$e}");
}
```

`.get` is a static constructor on `Request`, reached through the leading dot, so there's no `curl::get()` function to keep in sync with anything. `send` answers `result<Response, Error>`. The `guard` either binds the [](api:curl::Response), or hands you the [](api:curl::Error) and has to leave.

So, what's in the response?

```echo
echo $res->status;                                // 200
echo $res->contentType() ?? '(no content type)';  // text/html; charset=utf-8
echo "{$res->seconds:.3f}s";                      // 0.183s
echo $res->body->size();
```

Now the same request with two things added. A request is a value, so you fill it in before you send it:

```echo
curl::Request $req = .get('https://echoc.dev');
$req->userAgent('libcurl-example/0.1');
$req->timeout(10000);
```

`userAgent` and `timeout` each push one option onto the request. Nothing happens on the network until `send`.

Here is the catch everyone hits once: a 404 is not an `Error`. The transfer worked, the server just said no, and that's a perfectly good `Response` with `clientError()` answering true. `Error` is for transfers that didn't happen at all: DNS, connect, TLS, a timeout. [Errors](./errors.md) has the whole split.

The same program is `examples/get.eco`:

```bash
echoc run -m . examples/get.eco
```

## Where to go next

- **Requests** builds one up: verbs, bodies, [headers](./headers.md), [query strings](./query.md), [forms](./forms.md), and the [everyday options](./options.md).
- **Responses** reads one back, and covers [what a failure looks like](./errors.md).
- **Streaming** is for bodies that shouldn't sit in memory: [downloads](./download.md), [uploads](./upload.md), and [progress](./progress.md).
- **Clients** keeps a connection alive across requests, with [defaults](./client.md) that [stack](./stacking.md) and a base URL that [joins](./urls.md).
- **Below the API** is every libcurl option this library didn't name, and the [raw handle](./easy.md) underneath.
