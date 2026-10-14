---
description: Library defaults, then the client, then the request. Last write wins, for options and for headers.
---

# How options stack

A request can get an option from three places: this library's defaults, the [client](./client.md) it's sent through, and the request itself. The rule that decides between them is one sentence long.

## Three coats of paint

Options go on in a fixed order: **library defaults, then the client's list, then the request's list.** libcurl is last-write-wins, so a later option is an override, not a merge.

```echo
curl::Client $api = curl::Client();
$api->timeout(10000);           // coat two

curl::Request $req = .get('https://example.com/');
$req->timeout(500);             // coat three

$api->send($req);               // 500 ms wins
```

Every helper (`timeout`, `follow`, `set`, `raw`, ...) pushes an [](api:curl::Opt) onto a list, and `send` writes the three lists onto the handle in that order. The free `curl::send` is the same with an empty client list.

Within one list, order matters too. `$req->timeout(500); $req->timeout(900);` sends 900.

## The defaults

The first coat is short, and every entry is something libcurl leaves off on its own that you almost always want:

- **Follow redirects**, as `CURLFOLLOW_OBEYCODE`, so a 303 becomes the GET it asks for.
- **At most 10 redirects**, so a redirect loop fails fast with `.tooManyRedirects`.
- **Ask for compression**, with every encoding libcurl can decode.

Every handle also starts with `noSignal` on, because a library has no business taking `SIGALRM` from its host. That's still an option like any other, and you can turn it off.

TLS isn't on the list. Verification is libcurl's default, and this library doesn't touch it unless you do.

Turning a default off is the same as any override: `$req->follow(false)`, `$req->acceptEncoding(false)`. The last one is a real clear, not an empty value. No `Accept-Encoding` goes out.

## Headers merge the same way

Headers aren't options, but they follow the same idea. Client headers go first, and a request header with the same name (case-insensitively) **replaces** the client's:

```echo
$api->header('Accept', 'application/json');
$api->header('X-Client', 'kept');

curl::Request $req = .get('page');
$req->header('accept', 'text/html');
```

That request sends one `Accept: text/html` and `X-Client: kept`. Duplicates you add on the request itself with `headers->add` all go out, because those are yours.

## maxBytes across layers

[`maxBytes`](./download.md#capping-the-size) is an option too, so it stacks like one:

```echo
$api->maxBytes(1048576);  // a megabyte for everything

curl::Request $big = .get('export.csv');
$big->maxBytes(0);        // except this one: no cap
```

`0` is a real value, so it's an override like any other. That's how one request lifts a client-wide cap.
