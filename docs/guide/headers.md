---
description: Setting, adding and reading headers, and why they're a list and not a map.
---

# Headers

Headers go out on a [](api:curl::Request) and come back on a [](api:curl::Response), and both sides use the same type: [](api:curl::Headers), an ordered list of name and value pairs that's looked up case-insensitively.

## Setting vs adding

`header()` on a request replaces any header of the same name. That's what you want almost every time:

```echo
curl::Request $req = .get('https://httpbin.org/headers');
$req->header('Accept', 'text/html');
$req->header('Accept', 'application/json'); // replaces the first one
```

When a header legitimately repeats, go through the list and `add`. It keeps whatever is already there:

```echo
$req->headers->add('X-Tag', 'one');
$req->headers->add('X-Tag', 'two'); // both go out
```

An empty value is a real header. `$req->header('Accept', '')` sends `Accept:` with nothing after it, and replaces the `Accept: */*` libcurl would have written on its own.

## What send refuses

Nothing is checked when you add a header. It's checked when the request is sent, because `items` is a public field and `add` isn't the only way in.

A CR, LF or zero byte in either half, a `:` in the name, or an empty name fails `send` with `.badFunctionArgument`:

```echo
$req->header('X-Evil', "a\r\nInjected: yes");

result<curl::Response, curl::Error> $out = curl::send($req);
echo "{$out->failed()}"; // true
```

That's header injection, caught before a single byte leaves. A value with `\r\n` in it would otherwise become a second header you never wrote.

## Reading them back

A response hands you the same type. `header()` answers the first match or nothing, and `all()` answers every match, in arrival order:

```echo
echo $res->header('content-type') ?? '';
echo $res->contentType() ?? '';            // the same thing, spelled out

array<string> $cookies = $res->headers->all('Set-Cookie');
```

To walk all of them, loop over `items` by `const` reference. The fields on [](api:curl::Header) are public, which is what makes this compile:

```echo
foreach ($res->headers->items as const &$h) {
    echo "{$h->name}: {$h->value}";
}
```

`statusLine` holds the line the block started with, `HTTP/1.1 200 OK`, for the reason phrase. The number is `$res->status`.

## Why it's a list and not a map

A `map<string, string>` would compile, so this was a choice. Every consideration agreed:

- `Set-Cookie` repeats. A map keeps one of them.
- HTTP names are case-insensitive. A map hashes exact bytes, so every insert and every read would allocate a lowercase key.
- Map order is unspecified, and header order occasionally matters.

And a response carries ten to thirty headers. A scan over a contiguous array beats hashing at that size anyway.

## After a redirect

With redirects followed, libcurl sees one header block per hop. `$res->headers` is the **final** hop only, so `contentType()` is the destination's and not the redirect's. That's the classic wrapper bug, and it doesn't happen here.

Every block is still there, raw, in `$res->rawHeaders`. [](api:curl::parseHeaders) turns one back into `Headers`, and like the response it keeps only the last block:

```echo
curl::Headers $final = curl::parseHeaders($res->rawHeaders);
```
