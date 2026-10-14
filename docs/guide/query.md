---
description: Building ?a=1&b=2 without gluing strings, and percent-encoding on its own.
---

# Query strings

Gluing `"?q={$term}"` onto a URL works right up until `$term` holds a space, an `&` or a `#`. [](api:curl::Query) is the list of name and value pairs that encodes itself, so you never have to think about which characters are safe.

## Building one

Add pairs, then hand the whole thing to the request:

```echo
curl::Query $params = curl::Query();
$params->add('q', 'echo lang');
$params->add('tag', 'a&b');
$params->addInt('page', 2);

curl::Request $req = .get('https://example.com/search');
$req->query($params);

echo $req->url; // https://example.com/search?q=echo%20lang&tag=a%26b&page=2
```

`addInt` is there so you don't have to reach for `str::from` first. Pairs keep their order, and a name can repeat (`tag=a&tag=b`).

`query()` is careful about the URL it's appending to. It uses `&` when there's already a query, and it slots in before a `#fragment` instead of after it:

```echo
curl::Request $req = .get('https://example.com/docs?lang=en#install');
$req->query($params);

echo $req->url; // https://example.com/docs?lang=en&q=echo%20lang&tag=a%26b&page=2#install
```

The same `Query` is also the body of a URL-encoded form. [Forms](./forms.md) has that one.

## Encoding on its own

When you need one encoded piece and not a whole query (a path segment, say), [](api:curl::escape) and [](api:curl::unescape) are right there:

```echo
echo curl::escape('a b/c');       // a%20b%2Fc
echo curl::unescape('a%20b%2Fc'); // a b/c
```

`escape` leaves RFC 3986's unreserved set alone (`A-Z a-z 0-9 - . _ ~`) and turns every other byte into `%XX` with upper-case hex. It's binary-safe, so a zero byte becomes `%00`.

`unescape` has two rules worth knowing:

- `+` stays a `+`. Only `%20` is a space. That's form-encoding's convention, not the URL's.
- A `%` that isn't followed by two hex digits is kept as written. That's the forgiving reading every browser takes.

```echo
echo curl::unescape('1+1=2');  // 1+1=2
echo curl::unescape('100%');   // 100%
```

All of this is plain Echo. No handle, no libcurl call, which is why it works in a unit test with no network.
