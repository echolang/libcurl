---
description: Any libcurl option this library never named, by its number from curl.h.
---

# Options by number

The [named enums](./set.md) are closed. A libcurl release adds options every few months, and I'm not going to chase them all. So the enums being closed has to cost nothing, and that's what `raw` is for: any option, by its number from `curl.h`.

```echo
$req->raw(213, 1);   // CURLOPT_TCP_KEEPALIVE
$req->raw(214, 60);  // CURLOPT_TCP_KEEPIDLE, in seconds
```

## Finding the number

`curl.h` builds every option number from its type and an index: `CURLOPTTYPE_LONG` is 0, `CURLOPTTYPE_OBJECTPOINT` is 10000, `CURLOPTTYPE_OFF_T` is 30000. So `CURLOPT_TCP_KEEPIDLE` is `CURLOPT(..., CURLOPTTYPE_LONG, 214)`, which is 214. A string option like `CURLOPT_PROXY` (index 4, an object pointer) is 10004.

There are two overloads, one for integers and one for strings:

```echo
$req->raw(214, 60);                  // a long
$req->raw(10016, 'https://from.example/'); // a string, CURLOPT_REFERER
```

The integer overload also covers `curl_off_t`. A number in the 30000 to 39999 range goes out as a 64-bit `curl_off_t`, and everything below it as a `long`. So `raw` gets the C type right by reading it off the number, the same way `curl.h` encoded it.

An option the installed libcurl doesn't know fails `send` with `.unknownOption`. That's how an option newer than your libcurl shows up.

## On a client

`raw` on a [client](./client.md) is a default like any other, and here's the detail that matters: it's **replayed after every reset**. `send` resets the handle before every request so nothing leaks, and the client's raw options go back on right after, in the order you set them.

```echo
$api->raw(213, 1);
$api->send(.get('a')); // keepalive on
$api->send(.get('b')); // and still on
```

So don't reach past the client to poke the live handle. The next `send` resets it. Put the option on the client, or drive an [Easy](./easy.md) yourself.

## What raw can't do

`raw` takes an integer or a string. Some libcurl options take something else:

- **Lists** (`CURLOPT_RESOLVE`, `CURLOPT_HTTP200ALIASES`) want a `curl_slist`.
- **Blobs** (`CURLOPT_SSLCERT_BLOB`) want a `curl_blob`.
- **Callbacks** want a function pointer and its data pointer, and this library already owns the write, read, header and progress ones.

None of those fit through `raw`. A list or a blob is a job for your own `extern` on the raw `CURL*`, which [Easy](./easy.md#the-curl-underneath) hands you.
