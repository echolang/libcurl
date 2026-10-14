---
description: Timeouts, redirects, TLS, auth, proxies and compression, and what zero means for each.
---

# Everyday options

libcurl has a few hundred options. A dozen of them come up in every program, and those have a named helper on [](api:curl::Request). Each helper pushes one option (two for TLS) onto the request, and the same helpers exist on [](api:curl::Client) as defaults.

```echo
curl::Request $req = .get('https://echoc.dev');
$req->timeout(10000);
$req->userAgent('my-app/1.0');
$req->follow(false);
```

Nothing is applied until `send`. Then the options go on in a fixed order, and the last write wins. [How options stack](./stacking.md) has the order.

## Timeouts, and what zero means

Both timeouts are in milliseconds, and both have a zero that **does not** mean zero:

```echo
$req->timeout(5000);        // the whole transfer gets 5 seconds
$req->connectTimeout(2000); // DNS, TCP and TLS get 2 of them

$req->timeout(0);           // no limit at all
$req->connectTimeout(0);    // libcurl's built-in 300 seconds
```

That's libcurl's meaning, passed straight through. There's no way to ask for a zero-millisecond budget, and you don't want one. Running out of either is `.timeout`.

## Redirects

Redirects are followed by default, up to 10 hops. `follow(false)` hands the 3xx back as the response, and `maxRedirects` moves the cap. Going over it fails with `.tooManyRedirects`.

```echo
$req->follow(false);
$req->maxRedirects(3);
```

A 303 turns into a GET without the body, whatever the method was, which is what the spec asks for. That needs libcurl 8.13. An older one keeps a PUT a PUT.

## TLS

Certificates are verified, which is libcurl's default and this library never touches it unless you do. For a server with a private CA, trust its bundle:

```echo
$req->caFile('/etc/ssl/internal-ca.pem');
```

`verifyTls(false)` turns off both the chain check and the hostname check. **Use it for a known-bad host and nothing else.** It's the right answer for a dev box with a self-signed cert. It's the wrong answer for "the production API stopped working", where `caFile` is the fix.

## Auth, proxies and compression

```echo
$req->auth('user', 'secret');            // HTTP Basic
$req->proxy('http://127.0.0.1:8080');
$req->proxy('');                         // no proxy, even when http_proxy is set
$req->acceptEncoding(false);             // ask for no compression
```

Compression is on by default: libcurl sends `Accept-Encoding` with everything it can decode, and decodes the answer before you see it. `acceptEncoding(false)` clears that, so no `Accept-Encoding` goes out at all.

An empty `proxy('')` is the explicit "no proxy", and it beats the `http_proxy` environment variable. That's how a test talks to `127.0.0.1` on a machine with a corporate proxy configured.

## The list

Every helper on `Request`, and the libcurl option it writes. `Client` has the same set.

#### `timeout(int32 $ms)`
`CURLOPT_TIMEOUT_MS`. The whole transfer. `0` is no limit.

#### `connectTimeout(int32 $ms)`
`CURLOPT_CONNECTTIMEOUT_MS`. The connect phase. `0` is libcurl's 300 seconds.

#### `follow(bool $on)`
`CURLOPT_FOLLOWLOCATION`, as 2 for on (`CURLFOLLOW_OBEYCODE`). On by default.

#### `maxRedirects(int32 $n)`
`CURLOPT_MAXREDIRS`. 10 by default.

#### `verifyTls(bool $on)`
`CURLOPT_SSL_VERIFYPEER` and `CURLOPT_SSL_VERIFYHOST`, together.

#### `caFile(const string& $path)`
`CURLOPT_CAINFO`. A PEM bundle.

#### `userAgent(const string& $agent)`
`CURLOPT_USERAGENT`.

#### `auth(const string& $username, const string& $password)`
`CURLOPT_USERNAME` and `CURLOPT_PASSWORD`. Basic, unless `set(.httpAuth, ...)` says otherwise.

#### `proxy(const string& $url)`
`CURLOPT_PROXY`. `''` is no proxy.

#### `acceptEncoding(bool $on)`
`CURLOPT_ACCEPT_ENCODING`, as `''` for on and cleared for off. On by default.

#### `maxBytes(int64 $bytes)`
A byte cap on the response body. Enforced by this library, not libcurl. [Downloads](./download.md#capping-the-size) has the details.

Anything not on this list is still one line away. [Options by name](./set.md) is the next layer down.
