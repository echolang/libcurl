---
layout: home
hero:
  name: libcurl
  text: HTTP for Echo
  tagline: The C library everyone already has, wrapped so it reads like Echo. One request type, failures as values, and nothing hidden underneath.
  actions:
    - theme: brand
      text: Get started
      link: /guide/
    - text: API reference
      link: /api/
features:
  - title: One request type
    details: Build a Request, send it. The verbs are static constructors, so curl::send(.get($url)) is the whole short form.
    link: /guide/request
  - title: Failures are values
    details: send answers result<Response, Error>. A dead DNS lookup is an Error. A 404 is a Response. The types keep the two apart.
    link: /guide/errors
  - title: Streams both ways
    details: Save a download to disk, upload a file as it's read, watch the progress, and cancel with a return value.
    link: /guide/download
  - title: Nothing hidden
    details: Every libcurl option is one line away, by name or by number, and the easy handle underneath is yours to drive.
    link: /guide/set
---

You want to talk HTTP from Echo. libcurl is already on the machine, it speaks every protocol you've heard of, and it has been debugged by the entire internet. This library is that, without the C.

```echo
curl::Response $res = guard curl::send(.get('https://echoc.dev')) else ($e) {
    die("request failed: {$e}");
}

echo "{$res->status} in {$res->seconds:.3f}s"; // 200 in 0.183s
```

That `guard` is the error handling. There's no exception to catch: `send` hands back the response or the reason it couldn't. [Getting started](/guide/) builds from there.

```bash
epm add echolang/libcurl --git https://github.com/echolang/libcurl --range ^1.0
```

It's all Echo, with no C shim to compile. The callbacks libcurl calls back into are ordinary Echo functions handed to C. On Linux and Windows the C library needs one extra step, and [Finding libcurl](/guide/install) has it.
