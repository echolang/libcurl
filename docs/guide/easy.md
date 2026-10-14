---
description: The libcurl easy handle underneath everything, driven by hand.
---

# An Easy handle

`Request`, `Client` and `send` are all convenience on top of one type: [](api:curl::Easy), a single libcurl easy handle. Most programs never name it. It's here for the transfer the surface above doesn't spell, and nothing is hidden behind it: anything libcurl can do is reachable from here.

```echo
curl::Easy $handle = curl::Easy();

guard $handle->setString(.url, 'https://echoc.dev') else ($e) {
    die("{$e}");
}

guard $handle->perform() else ($e) {
    die("{$e}");
}

echo $handle->infoLong(.responseCode); // 200
echo $handle->body()->size();
```

## What you give up

Driving `Easy` yourself means you're below the layer that makes choices for you:

- **No library defaults.** Redirects aren't followed and compression isn't requested unless you set them. The one exception is `noSignal`, which every handle starts with.
- **No reset between performs.** Options stay on the handle until you call `reset()`. That's the bug a [Client](./client.md#nothing-leaks-between-requests) exists to avoid, so it's yours to handle now.
- **Every setter answers `status<Error>`.** A `Request` collects options and fails once, at `send`. Here each one can fail on the spot, `.unknownOption` included.

## Driving it

Options go on through the same three enums `set` uses, one setter each, plus `setRaw` for [numbers](./raw.md):

```echo
guard $handle->setLong(.followRedirects, 1) else ($e) { die("{$e}"); }
guard $handle->setString(.userAgent, 'my-app/1.0') else ($e) { die("{$e}"); }
guard $handle->setRaw(214, 60) else ($e) { die("{$e}"); }
```

The body and headers have their own setters, because they aren't single options in libcurl either. `setBody` is binary-safe, `addHeader` takes a whole `Name: value` line, and `setForm` takes a [Form](./forms.md). Set the form **after** the method: `CURLOPT_POST` flips a handle back to a plain POST, and the form would go out as an empty body. `sink`, `source` and `progress` are the [streaming](./download.md) hooks.

`reset()` drops every option, the headers, the form, the sink and the byte cap. It keeps the connection, which is the whole point of reusing a handle.

## Reading it back

After `perform`, the body and the raw header blocks are on the handle:

```echo
string $body = $handle->body();
curl::Headers $headers = curl::parseHeaders($handle->rawHeaders());
```

Everything else is an info, in three enums by C type, the same split as the options:

```echo
echo $handle->infoLong(.redirectCount);
echo $handle->infoString(.primaryIp) ?? '';
echo $handle->infoDouble(.connectTime);
```

[](api:curl::LongInfo), [](api:curl::StringInfo) and [](api:curl::DoubleInfo) list what's there. `infoString` answers a `string?`, since every string info can legitimately be missing.

`Easy` is `#[unique]`, so it can't be copied. Two holders of one handle would read each other's responses after the next `perform`. That's an error at the copy, not a mystery at runtime.

## The CURL* underneath

`raw()` hands you the `CURL*` itself. This library's own externs are internal, so the C call you want needs an `extern` of your own. Declare the opaque type and recast the pointer to it:

```echo
extern {
    struct CURL;
    function curl_easy_getinfo as getinfo(ptr<CURL> $handle, int32 $info, variadic_args $args) : int32;
}

$theirs = $handle->raw();
ptr<CURL> $c = ptr<CURL>($theirs:$);

int64 $code = 0;
getinfo($c, 2097154, [&$code]); // CURLINFO_RESPONSE_CODE
```

The handle still belongs to the `Easy`. Don't clean it up yourself, and don't keep the pointer past the `Easy` that owns it.
