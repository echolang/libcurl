---
description: Bodies that never sit in memory, straight to a file or one chunk at a time, and a cap on how big they get.
---

# Downloads

By default the response body is a `string` in `Response::body`. That's the right shape for an API call. It's the wrong shape for a 4 GB ISO. A request's [](api:curl::Sink) says where the body goes instead, and two helpers on `Request` set it.

## To a file

`saveTo` writes the body to a file as it arrives:

```echo
curl::Request $req = .get('https://example.com/big.iso');
$req->saveTo('/tmp/big.iso');

curl::Response $res = guard curl::send($req) else ($e) {
    die("{$e}");
}

echo $res->bytes;        // 4371513344
echo $res->body->size(); // 0
```

The body went to the file, so `body` is empty and `bytes` is the count. The file is created, or truncated if it exists.

Here is the part I like: **a transfer that fails removes the file again.** The path holds a complete body or nothing at all, so a half-downloaded file can never pass for a whole one. A path that can't be created or written is `.writeError`.

## In chunks

When you want to look at the body as it streams (hash it, parse it, forward it), `onChunk` hands it over one chunk at a time and keeps none of it:

```echo
curl::Request $req = .get('https://example.com/big.log');
$req->onChunk(function(const string& $chunk) : bool {
    echo "{$chunk->size()} bytes";

    return true;
});
```

Chunks arrive in order, and their size is whatever libcurl had ready. Return `false` and the transfer stops with `.abortedByCallback`.

So, how does the closure keep a running total? A closure captures by value, so an `int64` declared outside gets a copy of its own, and after `send` the outer one still reads `0`. Capture a **class** instead: the closure and the code around it then share one object.

```echo
class Tally
{
    public int64 $bytes;

    constructor()
    {
        $this->bytes = 0;
    }
}

Tally $seen = Tally();

$req->onChunk(function(const string& $chunk) : bool {
    $seen->bytes = $seen->bytes + $chunk->size();

    return true;
});
```

After `send`, `$seen->bytes` is the total.

## Capping the size

A server can send you more than you asked for. `maxBytes` puts a ceiling on the body:

```echo
curl::Request $req = .get('https://example.com/avatar.png');
$req->maxBytes(5 * 1024 * 1024);
```

Past the cap, the transfer fails with `.filesizeExceeded`, and a `saveTo` file is removed like any other failure.

This one is enforced by this library and not by libcurl, on purpose. libcurl's own check trusts `Content-Length`, and a server is free to leave that out. Here the bytes are counted as they arrive, **after** decompression, whatever the sink. So a small gzip body that inflates into a large one still trips it, and that's the case that actually costs you memory.

The rest of the rules:

- `0` means no cap. That's also how a request lifts a cap its [client](./client.md) set.
- A negative cap fails `send` with `.badFunctionArgument`.
- `set(off: .maxFileSize, $n)` is the same option, spelled the long way.

`examples/download.eco` streams a file to another file over `file://`, so it needs no network. It reports progress on the way, then shows the cap refusing:

```bash
echoc run -m . examples/download.eco
```
