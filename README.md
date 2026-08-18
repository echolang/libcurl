# libcurl

An HTTP client for [Echo](https://github.com/echolang/echo), sitting on the C library of the same name.

## Introduction

Sometimes your Echo program needs to talk HTTP, and the standard library does not. libcurl already does, on every machine you actually ship to, so this module is that library with an Echo API on top.

It is all Echo. There is no C shim and no extra build step of its own. The write callback is an ordinary Echo function handed to libcurl as a C function pointer, and `curl_easy_setopt` is reached through `variadic_args`. The only thing outside the process is `-lcurl`.

Let's look at a complete program:

```echo
curl::Response $res = guard curl::send(.get('https://api.github.com/zen')) else ($e) {
    die("request failed: {$e}");
}

echo "{$res->status} in {$res->seconds:.3f}s";
echo $res->body;
```

That is the whole shape of the library: one `Request`, one `send`, and a `result` that tells you whether the transfer completed. Don't worry if the types are new. We'll walk through each piece below.

## Installing

There is no package manager, so a dependency is a path on disk. Place this repository beside your project and name it in your module file:

```echo
// your module.eco
#[module: "myapp"]
#[depends: "../libcurl"]
#[sources: "src/*.eco"]
```

`#[link: lib "curl"]` travels with this module, so you never write it yourself.

### Finding libcurl

Where libcurl is installed is a property of the machine, not of this repository. On a Mac with Homebrew, you may point the linker at it:

```bash
echoc build --link search:/opt/homebrew/opt/curl/lib
```

You still need the runtime. macOS ships it. Linux calls it `libcurl4`.

Here is the catch: `echoc run` additionally needs the `-dev` package, because the JIT `dlopen`s the unversioned `libcurl.so` and only that package installs it. `echoc build` has no such requirement.

If you would like to see which libcurl your program actually picked up:

```echo
echo curl::version();
```

### A note on Windows

This library does not target Windows. C `long` is eight bytes on the platforms it does target, and four under Windows' LLP64. The declarations would compile. They would also be wrong.

## Sending a Request

You need to make an HTTP request. This library has one path for that: build a `Request` and `send` it. There is no parallel `curl::get($url)` family beside it. A second entry point per verb is surface that has to stay consistent with the first one forever, and it is the half that drifts.

The convenience lives on the request itself. The leading-dot shorthand reaches the static constructors, so the short spelling costs no extra API:

```echo
curl::send(.get('https://example.com/'))
curl::send(.postJson('https://example.com/items', '{"name":"echo"}'))
curl::send(.del('https://example.com/items/1'))
```

### Choosing a verb

`get`, `post`, `postJson`, `postForm`, `put`, `patch`, `del`, and `head` are the verbs this library names.

`del` rather than `delete`, which reads as a keyword in most of the languages nearby. `head` tells libcurl not to expect a body at all. `postJson` sets the JSON body and the `Content-Type` that has to go with it. `postForm` encodes a `Query` as `application/x-www-form-urlencoded`.

### Methods this library does not name

A verb this library does not name is still a request. The constructor is public:

```echo
curl::Request $req = curl::Request('OPTIONS', 'https://example.com/items');
```

### Filling a request in

Method, URL, body, and headers are fields. They are the HTTP bits that are not a single setopt. Everything else is an option. Echo has no named arguments, so the methods that push an `Opt` are the optional arguments:

```echo
curl::Request $req = .get('https://example.com/feed');

$req->timeout(2000);
$req->follow(true);
$req->maxRedirects(3);
$req->body = '{"ok":true}';
$req->header('Accept', 'application/json');
$req->auth('user', 'secret');

curl::Response $res = guard curl::send($req) else ($e) {
    die("failed: {$e}");
}
```

`header` and `query` edit the request. `timeout`, `auth`, `set`, `proxy`, and `raw` append to `$req->opts`. If you need an option this type never grew a field for, you may still send it:

```echo
$req->set(.proxy, 'http://127.0.0.1:8080');
```

`$req->header($name, $value)` replaces any header of that name. `$req->headers->add($name, $value)` keeps both. Same type, two spellings, because those are two different things to want.

A few knobs have a sharper meaning than they first look. `$req->maxRedirects(0)` refuses every redirect. `$req->timeout(0)` is a zero-millisecond budget, not "no timeout." Leaving a knob unsaid leaves the layer above it alone.

### Copying a request

A `Request` has no destructor, so `$b = $a;` is an ordinary deep copy. That is what makes "one template, three variants" plain value code rather than a factory you have to remember:

```echo
curl::Query $page = curl::Query();
$page->addInt('page', 1);

curl::Request $base = .get('https://api.example.com/items');
$base->header('Accept', 'application/json');
$base->timeout(2000);

curl::Request $first = $base;
$first->query($page);

curl::Request $second = $base;
$second->url = 'https://api.example.com/items/2';
```

## Handling Failures

Sometimes the transfer will not happen at all: the host is unknown, the connection times out, the URL is malformed. `send` answers `result<Response, Error>`, which is what the whole of `std::io` does. `guard` is the short way through:

```echo
curl::Response $res = guard curl::send(.get($url)) else ($e) {
    die("request failed: {$e}");
}
```

So, what happens if you want to treat one failure differently from another? `Error` is an enum, so the `else` arm of `guard` is something you can `match`:

```echo
curl::Response $res = guard curl::send(.get($url)) else ($e) {
    match ($e) {
        .timeout($m) => {
            die("timed out");
        },
        .couldNotResolveHost($m) => {
            die("no such host");
        },
        .unknownOption($m) => {
            die("this libcurl is too old");
        },
        else => {
            die("request failed: {$e}");
        },
    }
}
```

You do not have to name every case. Match the ones you care about and let `else` take the rest.

`"{$e}"` renders libcurl's sentence. `$e->value()` is the ABI number, and `$e->message()` is the same sentence without interpolating. Echo has no "enum from a raw value," so a CURLcode this library has not named becomes `.other` rather than a lie.

### Transfers versus HTTP status

**A `Response` exists only when the transfer completed.** A DNS failure is an `Error`. A 404 is a perfectly good `Response` whose `clientError()` answers true.

"Did the transfer work" and "was the server happy" are different questions, and the types keep them apart.

`file://` makes the split obvious. Curl speaks it, the transfer can succeed, and there is still no HTTP status:

```echo
curl::Response $res = guard curl::send(.get('file:///tmp/fixture.txt')) else ($e) {
    die("{$e}");
}

echo $res->body;
echo $res->status;      // 0
echo $res->success();   // false
```

The body came back. The server was not happy, because there was no server.

## Working With Responses

When `send` succeeds, you hold a `Response`:

```echo
$res->status           // int64, 0 when the protocol has none (file://)
$res->body             // string, binary safe
$res->url              // where it ended up, after redirects
$res->seconds          // float64, wall time for the whole transfer
$res->headers          // curl::Headers, the final hop
$res->rawHeaders       // every block libcurl saw, status lines and all
```

The body is exactly the bytes that arrived. A zero byte is a byte like any other. `"{$res}"` is the short form, `200 (142 bytes)`, which is useful in a log and useless as a parser.

### Asking about the status

You may ask the status in words instead of comparing numbers yourself:

```echo
$res->success()        // 2xx
$res->redirect()       // 3xx, only reachable when redirects were not followed
$res->clientError()    // 4xx
$res->serverError()    // 5xx

$res->header('content-type')    // string?, case-insensitive
$res->contentType()             // string?
```

`redirect()` is only reachable when this request did not follow redirects. If the library (or you) followed them, you land on the final hop instead.

### Reading headers

Header lookup is case-insensitive, repeated headers are kept, and order is preserved. That is not a `map<string, string>`. A map would compile. It would also hash exact bytes (so case-insensitive lookup would allocate a lowercase key on every insert *and* every read), drop every `Set-Cookie` after the first, and iterate in an order HTTP does not promise.

```echo
echo $res->header('Content-Type') ?? '(none)';

array<string> $cookies = $res->headers->all('set-cookie');

foreach ($res->headers->items as const &$header) {
    echo "{$header->name}: {$header->value}";
}
```

`$res->header($name)` is the first match, or nothing. `$res->headers->all($name)` is every match, in arrival order. `Set-Cookie` is why `all` exists.

### Headers after a redirect

If the request followed redirects, `$res->headers` is the **final** response. Every hop is still in `rawHeaders`. Missing that split is the classic wrapper bug where you read the redirect's `Content-Type` instead of the destination's.

## Reusing a Connection

If you send several requests to the same host, you would rather not open a new TCP connection each time. A `Client` is that connection, not a namespace. It holds one libcurl handle, so a run of requests reuses the TCP connection, the DNS answer, and the TLS session. That is the entire performance argument for using libcurl rather than a socket.

Everything else it holds is defaults, so "every request to this API carries this header" is said once.

It is a class because it has identity rather than contents. Two names for one client are two names for one connection. As a struct it would silently copy the handle.

```echo
curl::Client $api = curl::Client();

$api->baseUrl = 'https://api.example.com';
$api->header('Authorization', 'Bearer abc123');
$api->userAgent('myapp/1.0');
$api->timeout(5000);
$api->connectTimeout(1000);
$api->auth('user', 'secret');

curl::Response $one = guard $api->send(.get('/health')) else ($e) { die("{$e}"); }
curl::Response $two = guard $api->send(.postJson('/items', $json)) else ($e) { die("{$e}"); }
```

The free `send` and `Client::send` take the same `Request`. Moving between them changes the receiver and nothing else. The free function is the spelling for a script, or for a request that has nothing to reuse. It differs in exactly one way: no connection is kept afterwards.

### How options stack

A `Client` holds the same `Opt` list a `Request` does. `send` writes three layers, in order: the library defaults, then the client's list, then the request's. libcurl is last-write-wins, so a later option is an override, not a merge.

Think of it as three coats of paint. The last coat is what you see.

The library writes three things unless you overwrite them:

- follow redirects
- ten hops
- ask for compression (`gzip` / `deflate`)

TLS stays on by not touching it. That is libcurl's own default.

```echo
$api->verifyTls(false);          // this API is a known-bad host. that is the only reason
$req->verifyTls(true);           // this one request still checks
```

Turn `verifyTls` off only for a known-bad host. The default is on. If you need your own CA bundle instead, you may point at it with `caFile`.

A request header of the same name replaces the client's default rather than joining it. Request-level duplicates (`$req->headers->add`) stay as two items.

`send` resets the handle between requests. That is why defaults live on a list, not on a live handle. Writes on a handle would be discarded the moment the next `send` started.

### How URLs join

An absolute URL always wins over the base. A protocol-relative one (`//cdn.example.com/x`) takes the base's scheme and nothing else of it. A path that starts with `/` replaces everything after the host, the way a browser does:

```echo
// https://api.example.com/v1  +  /health
// becomes https://api.example.com/health
$api->baseUrl = 'https://api.example.com/v1';
$api->send(.get('/health'));
```

That is what gluing the slash onto the path would silently fail to do. A relative segment (`health`) is joined with exactly one slash.

A URL that only *contains* `://` is not absolute. `redirect?url=https://evil.com` still joins onto the base, which is what you asked for.

## Building Query Strings and Forms

Sometimes you need a query string on a URL. Sometimes you need an `application/x-www-form-urlencoded` body. They are the same encoding, so they are the same type.

`Query` is a list of name/value pairs that render as `a=1&b=2`:

```echo
curl::Query $params = curl::Query();
$params->add('q', 'echo lang');
$params->addInt('page', 2);

curl::Request $search = .get('https://example.com/search');
$search->query($params);            // ?q=echo%20lang&page=2

curl::Request $form = .postForm('https://example.com/login', $params);
```

`addInt` is there so you do not have to reach for `str::from` first.

A fragment stays a fragment. `.get('https://example.com/page#section')` plus a query becomes `.../page?a=1#section`, not `...#section?a=1`. `$url->contains('?')` plus an append is how the fragment used to eat the query.

### Encoding on its own

`curl::escape` and `curl::unescape` are the same percent-encoding, on their own. Both are pure Echo. They need no handle and no connection, which is why a query can be tested without touching the network.

Unreserved is RFC 3986's set (`A-Z a-z 0-9 - . _ ~`). Everything else becomes `%XX` with upper-case hex. A `%` that is not followed by two hex digits is kept as written, which is the forgiving reading every browser takes. A zero byte encodes to `%00` like any other.

## Reaching Past the Named API

Sometimes you need an option this library has not grown a method for, or has not named at all. Nothing in libcurl is out of reach, and you never have to wait for a release.

Options this library names are enums, so they read as `.followRedirects` rather than as a transliterated C `#define`. There is one enum per C type, because a variadic tail element keeps its own type and there is no destination to widen a literal against. `cSetopt($h, 52, [1])` would place a four-byte `int32` where C reads an eight-byte `long`. That is the oldest bug in every curl binding. Going through `Easy::setLong` makes the mistake unreachable rather than merely discouraged.

### Named options through `set`

Options this library has named but not grown a method for go through `set`. They live on the same list, on the request or the client, and they survive `send`:

```echo
$api->set(.tcpKeepAlive, 1);     // CURLOPT_TCP_KEEPALIVE, every request on this client
$req->set(.proxy, 'http://127.0.0.1:8080');
$req->set(.cookie, 'session=abc');
```

An option the installed libcurl has never heard of is `.unknownOption`.

### Unnamed options through `raw`

Options this library has not named go by number through `raw`. They are applied last among that request's (or client's) opts, so they win:

```echo
$req->raw(215, 60);              // CURLOPT_TCP_KEEPINTVL, an unnamed long
```

### Driving an Easy handle

Drive an `Easy` yourself if you want the `CURL*` with no `send` in between. This is the floor of the library. Most programs never name it. Nothing is hidden behind `Client` and `Request`.

```echo
curl::Easy $handle = curl::Easy();
$handle->setString(.url, 'https://example.com/');
$handle->setLong(.followRedirects, 1);

int32 $code = $handle->perform();

if ($code != curl::CODE_OK) {
    die($handle->error($code)->message);
}

echo $handle->body();
echo $handle->infoLong(.responseCode);
```

`Easy` is `#[unique]`. Every `perform` clears both buffers, so if a handle were shared a second holder calling `body()` after the first performed again would read somebody else's response. The copy is a located error instead of a puzzle at runtime.

`Easy::raw()` hands back the `CURL*` itself for anything even `setRaw` cannot express.

After a transfer you may also read infos back off the handle: `infoLong(.redirectCount)`, `infoString(.primaryIp)`, `infoDouble(.connectTime)`, and the rest of the named families.

### Named options and infos

These are the options and infos this library currently names. Anything else is still reachable through `raw` / `setRaw`.

**Long options:** `verbose`, `noBody`, `failOnError`, `upload`, `post`, `followRedirects`, `verifyPeer`, `maxRedirects`, `httpGet`, `verifyHost`, `noSignal`, `httpAuth`, `timeoutMs`, `connectTimeoutMs`, `tcpKeepAlive`

**String options:** `url`, `proxy`, `referer`, `userAgent`, `cookie`, `customRequest`, `caInfo`, `caPath`, `acceptEncoding`, `username`, `password`, `unixSocketPath`

**Offset options:** `maxFileSize`, `postFieldSize`

**Infos:** `responseCode`, `redirectCount`, `httpVersion`, `effectiveUrl`, `contentType`, `redirectUrl`, `primaryIp`, `scheme`, `totalTime`, `nameLookupTime`, `connectTime`

## Testing

```bash
echoc test                        # everything
echoc test --filter group:unit    # no network
echoc test --filter group:net     # only the network ones
```

The unit tests need no network at all. Header parsing, percent-encoding, URL join, and the option stack are pure Echo. Transfers are exercised over `file://`, which curl speaks. The network tests give up quietly when the machine is offline rather than failing.

## Examples

```bash
echoc run -m . examples/fileUrl.eco     # no network
echoc run -m . examples/get.eco
echoc run -m . examples/client.eco
echoc run -m . examples/rawOption.eco
```

## What Is Not Here Yet

- **Streaming.** Bodies are assembled in memory. Download-to-file and upload-from-file are the obvious next feature, and `std::io::file` is what they will be built on.
- **Concurrency.** No `curl_multi`, because Echo has no threads. The `Easy` layer is separable enough that a `Multi` can arrive beside `Client` without either changing.
- **A cookie jar and a MIME builder.** `CURLOPT_COOKIE` and `CURLOPT_PROXY` are named options; `$req->set(.cookie, ...)` and `$req->set(.proxy, ...)` are enough to use them. A jar and a multipart builder are not wrapped.
- **A byte cap on the response.** `MAXFILESIZE` is checked against `Content-Length`, so a server that declines to send one can still grow the buffer.

## Source Layout

| File | What it holds |
|---|---|
| `src/libcurl.eco` | the one `extern` block. Every C symbol, declared once |
| `src/opt.eco` `info.eco` | the options and infos this library names, and `Opt` |
| `src/easy.eco` | `Easy`. The handle, the write callback, the escape hatch |
| `src/error.eco` `from.eco` | `Error`, and the `str::from` that makes `"{$e}"` work |
| `src/headers.eco` | `Header`, `Headers`, and the response-block parser |
| `src/query.eco` | percent-encoding, `Field`, and `Query` |
| `src/url.eco` | absolute / join / appendQuery |
| `src/request.eco` `response.eco` | the two values |
| `src/apply.eco` | join URL, merge headers, stack options, poke libcurl |
| `src/client.eco` | `Client`, and the free `send` |
| `src/text.eco` | the ASCII bytes this module compares against |
