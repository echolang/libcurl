---
description: Transfer failures as values, what each case means here, and the one catch with other.
---

# Errors

Echo has no exceptions, so a failed transfer comes back as a value. `send` answers `result<Response, Error>`, and [](api:curl::Error) is the reason nothing came back: the DNS lookup failed, the connection was refused, the timeout ran out.

A 404 is **not** an `Error`. The transfer worked, the server said no, and that's a [Response](./response.md#a-404-is-a-response). This page is only about transfers that didn't happen.

## guard and match

The usual shape is a `guard` that binds the response, or leaves:

```echo
curl::Response $res = guard curl::send(.get($url)) else ($e) {
    die("request failed: {$e}");
}
```

`"{$e}"` is libcurl's own sentence for the code, the same thing `$e->message()` answers. `$e->value()` is the raw CURLcode.

When you care which failure it was, `match` on it. `Error` is an enum, so the cases read like any other:

```echo
curl::Response $res = guard curl::send(.get($url)) else ($e) {
    match ($e) {
        .timeout => {
            die('too slow, try again later');
        },
        .couldNotResolveHost => {
            die('offline?');
        },
        else => {
            die("{$e}");
        },
    }
}
```

Without the `guard`, the `result` has the usual methods:

```echo
result<curl::Response, curl::Error> $out = curl::send($req);

if ($out->failed() && $out->failure() == curl::Error::timeout) {
    echo 'timed out';
}
```

## What each case means here

Most cases mean exactly what libcurl says they mean. A handful are also how this library reports **your** mistakes, and those are worth knowing by name:

| Case | In this library |
|:--|:--|
| `badFunctionArgument` | The request contradicts itself: two bodies at once, a HEAD with a body, a header or form part that can't go on the wire, a negative `maxBytes`. |
| `abortedByCallback` | An `onProgress` or `onChunk` closure answered `false`. That's what [cancelling](./progress.md) looks like. |
| `filesizeExceeded` | The body went past the [`maxBytes`](./download.md#capping-the-size) cap. |
| `writeError` | A `saveTo` path couldn't be created or written. |
| `fileCouldNotReadFile` | The file behind `uploadFile`, or a `file://` URL, couldn't be read. |
| `readError` | A file part of a [form](./forms.md) couldn't be read. |
| `unknownOption` | A `set` or `raw` option the installed libcurl doesn't know. |
| `httpReturnedError` | A 4xx or 5xx, but only after you asked for that with `set(.failOnError, 1)`. |
| `failedInit` | libcurl wouldn't hand out a handle. |

The [API page](api:curl::Error) lists every case, with its CURLcode.

## The other catch

`Error` is an open enum. A CURLcode it doesn't name, from a newer libcurl or a protocol this library doesn't wrap, is `other`, and `value()` still has the number. `Error::from` never fails.

Here is the catch, and it's a compiler one: under echoc 0.8.0, a leftover number goes to an `else` arm **even when an `.other` arm sits right next to it**, and `== curl::Error::other` doesn't compile.

```echo
match ($e) {
    .timeout => { ... },
    .other => { ... },  // never reached
    else => { ... },    // a leftover lands here
}
```

So if you really need `other` on its own, match every case and leave `else` out. For everything else, `else` plus `$e->value()` already gives you the number.

`examples/fileUrl.eco` shows the failing side with a real message from libcurl:

```bash
echoc run -m . examples/fileUrl.eco
```
