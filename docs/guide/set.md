---
description: Every libcurl option this library names, set through one enum per C type.
---

# Options by name

The [everyday helpers](./options.md) cover a dozen options. libcurl has a few hundred more. A good chunk of the useful ones are named here anyway, in three enums: [](api:curl::LongOption), [](api:curl::StringOption) and [](api:curl::OffOption). `set` on a `Request` or a `Client` takes any of them.

```echo
$req->set(.verbose, 1);
$req->set(.cookie, 'session=abc123');
$req->set(off: .postFieldSize, 5);
```

## One enum per C type

Why three enums and not one? Because `curl_easy_setopt` is variadic, and C reads the value as whatever type the option says it is. Hand a four-byte `int32` to an option that reads a `long` and you get garbage on darwin and Linux, where `long` is eight bytes. That's the oldest bug in every curl binding.

So the option's enum carries its C type, and each `set` overload widens the value to match before it ever reaches C. Passing the wrong width isn't something you can get wrong, because you can't spell it.

The leading dot does the rest. `.verbose` resolves against `LongOption`, `.cookie` against `StringOption`, and you never qualify an option.

## Why off: is labelled

`curl_off_t` options take an `int64`, and so do `long` options. With two overloads both taking `(enum, int64)`, a bare `.postFieldSize` can't say which enum it means. So the `curl_off_t` overload is labelled:

```echo
$req->set(off: .maxFileSize, 1048576); // works
$req->set(.maxFileSize, 1048576);      // will not compile: .maxFileSize isn't a LongOption
```

The label is part of the signature, so it isn't optional. It's also the only one there is.

## A few worth knowing

```echo
$req->set(.failOnError, 1);                          // a 4xx or 5xx becomes .httpReturnedError
$req->set(.unixSocketPath, '/var/run/docker.sock');  // talk to Docker
$req->set(.cookie, 'a=1; b=2');                      // a Cookie header, no jar
$client->set(.tcpKeepAlive, 1);                      // probe idle connections
$req->set(.verbose, 1);                              // libcurl's own trace, on stderr
```

`failOnError` is the one people ask for. It's off by default because a 404 is a [perfectly good response](./response.md#a-404-is-a-response), but if your code only ever wants 2xx, it turns everything else into an [](api:curl::Error).

`.verbose` is the first thing to reach for when a request misbehaves. It prints every header, both ways, and the TLS handshake.

## The Opt list underneath

Every helper and every `set` does the same thing: it pushes one [](api:curl::Opt) onto the request's `opts` array. `Opt` is the option language. A request is method, URL, body, headers and a list of these.

The list is a public field, so you can push onto it yourself:

```echo
$req->opts[] = curl::Opt::long(.verbose, 1);
$req->opts[] = curl::Opt::clear(.acceptEncoding);
```

`clear` is the one you can't get through `set`. It sets a string option back to NULL, which is how you undo something an earlier layer wrote. `acceptEncoding(false)` is exactly that.

For an option that isn't in any of the three enums, [Options by number](./raw.md) is the next step down.
