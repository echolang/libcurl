---
description: What a Response holds, how to ask about its status, and where the time went.
---

# Responses

A [](api:curl::Response) is the record of a transfer that completed. You don't build one and you don't edit one: every field is `const`, and `send` fills them all in from libcurl's own answers.

```echo
curl::Response $res = guard curl::send(.get('https://echoc.dev')) else ($e) {
    die("{$e}");
}

echo $res->status;      // 200
echo $res->body->size(); // the bytes, all of them
echo $res->url;         // https://echoc.dev/
```

## What comes back

- `status`: the HTTP status, or `0` when the protocol has none.
- `body`: the bytes, exactly as they arrived. A zero byte is a byte like any other.
- `bytes`: how many body bytes arrived. The same as `body->size()` by default, and the only count there is when the body went to a [file or a closure](./download.md).
- `url`: where the transfer ended up, after any redirects.
- `headers` and `rawHeaders`: the final hop's [headers](./headers.md), and every header block, raw.
- `seconds`: wall time for the whole thing.
- `redirects`, `version`, `ip`, `port` and `timings`: where it went and how long it took. More on those below.

## Asking about the status

Four helpers, one per class of status:

```echo
if ($res->success()) {          // 2xx
    echo $res->body;
}
else if ($res->clientError()) { // 4xx
    echo "the request was wrong: {$res->status}";
}
else if ($res->serverError()) { // 5xx
    echo "the server was wrong: {$res->status}";
}
```

`redirect()` answers true for a 3xx, which you only ever see with [`follow(false)`](./options.md#redirects).

## A 404 is a Response

Here's the split that matters most. "Did the transfer work?" and "Was the server happy?" are different questions, and the types keep them apart:

- The DNS lookup failed, the connection was refused, the timeout ran out: that's an [](api:curl::Error). There's no `Response` at all.
- The server answered 404, or 500: that's a `Response`. The transfer worked perfectly. The server said no.

```echo
curl::Response $res = guard curl::send(.get('https://httpbin.org/status/404')) else ($e) {
    die("never got an answer: {$e}");
}

echo "{$res->clientError()}"; // true
```

A wrapper that fumbles this split is not a wrapper you can trust. If you'd rather have a 4xx be an `Error`, libcurl has an option for that, and [Options by name](./set.md) shows it.

## Where it went and how long it took

```echo
echo "{$res->status} over {$res->version} from {$res->ip}:{$res->port}"; // 200 over HTTP/2 from 104.21.48.1:443
echo "{$res->redirects} redirects";
```

`version` is an [](api:curl::HttpVersion), the protocol the final hop actually spoke. That isn't always the one you asked for.

[](api:curl::Timings) is where the time went, as libcurl measured it. **Every mark counts from the start of the transfer**, not from the mark before:

```echo
echo "dns {$res->timings->dns:.3f}s";
echo "connect {$res->timings->connect:.3f}s";
echo "tls {$res->timings->tls:.3f}s";
echo "first byte {$res->timings->firstByte:.3f}s";
```

That's the useful shape. `firstByte - sent` is the server's think time, and `connect - dns` is the TCP handshake. A phase that didn't happen reads zero: `tls` over plain HTTP, `dns` on a reused connection.

## file:// responses

libcurl speaks more than HTTP, and `file://` is the handy one, since it needs no network at all. A file transfer completes like any other, with a catch:

```echo
curl::Response $res = guard curl::send(.get(std::env::file_url('/etc/hosts'))) else ($e) {
    die("{$e}");
}

echo $res->status;          // 0
echo "{$res->success()}";   // false
```

No HTTP, so no status, so `success()` is false even though the transfer worked. `version` is `.none` and `ip` is empty. That's the same split as the 404, from the other side.

## Printing one

`"{$res}"` is the short form, status and size. `"{$res->version}"` is the status-line spelling, and `"{$e}"` on an error is libcurl's sentence for it:

```echo
echo "{$res}"; // 200 (1256 bytes)
```

`examples/fileUrl.eco` runs the `file://` path, both sides of it:

```bash
echoc run -m . examples/fileUrl.eco
```
