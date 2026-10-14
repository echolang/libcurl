---
description: What this library doesn't do yet, how its tests are split, and the example programs.
---

# What isn't here

This library covers one transfer at a time, done well. A few things that sit next to that are deliberately not built yet. Here's the list, so you don't go looking.

## Not here yet

- **Concurrent transfers.** There's no `curl_multi`. One handle does one transfer at a time, and `send` blocks the thread it runs on. Threads work fine: give each thread its own `Client` (or use `curl::send`), and see [Progress and cancelling](./progress.md#cancelling-from-another-thread) for stopping one from outside. The `Easy` layer is separable enough that a `Multi` can arrive next to `Client` without either one changing.
- **A cookie jar.** `set(.cookie, 'a=1; b=2')` sends cookies, and `$res->headers->all('Set-Cookie')` reads them back. Nothing stores them between requests for you.
- **List, blob and callback options.** `raw` takes integers and strings. [Options by number](./raw.md#what-raw-can-t-do) says what's missing and how to get past it with your own `extern`.

## Running the tests

```bash
echoc test                       # everything
echoc test --filter group:unit   # no network
echoc test --filter group:net    # the public internet
echoc test --filter group:wire   # a loopback server
```

The tests come in three groups:

- **`unit`** needs no network at all. Header parsing, percent-encoding, the URL join and the option stack are pure Echo, and real transfers go over `file://`, which curl speaks.
- **`net`** talks to `example.com` and `httpbin.org`, and passes quietly when the machine is offline rather than failing.
- **`wire`** starts a small HTTP server on loopback and checks the exact bytes that went out: `Content-Length` on an empty POST, header merging, the redirect hops. It's POSIX only, so it doesn't run on Windows.

## The examples

Each one runs from the repository root:

```bash
echoc run -m . examples/get.eco       # the shortest thing that works
echoc run -m . examples/client.eco    # two requests on one connection
echoc run -m . examples/fileUrl.eco   # file://, both sides of it, no network
echoc run -m . examples/download.eco  # saveTo, progress and a cap, no network
echoc run -m . examples/upload.eco    # a multipart form, then the timings
```

`upload.eco` posts to `httpbin.org` by default. Pass any server that echoes a request to use that instead:

```bash
echoc run -m . examples/upload.eco -- http://127.0.0.1:8080/
```
