# libcurl

An HTTP client for Echo, on libcurl's easy interface. Build a `curl::Request` and `send` it.

```bash
epm add --git https://github.com/echolang/libcurl.git --range ^1.0.0
epm install
```

macOS and Linux link `-lcurl`. macOS has it in the SDK. On Ubuntu, `libcurl4-openssl-dev`. `echoc run` and `echoc test` need the shared library. The JIT cannot open an archive.

`echoc build --define static_curl` links static `libcurl`, `ssl`, and `crypto`.

Windows links static `libcurl` (Schannel). The archive has to be on the link path. vcpkg also needs zlib, which the manifest does not name:

```bash
vcpkg install curl:x64-windows-static
echoc test --link "search:<prefix>\lib" --link "lib:zs"
```

`zs` is zlib 1.3.2 and newer. Older ports called it `zlib`.

## One request

```echo
curl::Request $req = .get('https://echoc.dev');
$req->userAgent('my-app/1.0');

curl::Response $res = guard curl::send($req) else ($e) {
    die("{$e}");
}

echo $res->body;
```

`send` returns `result<Response, Error>`. A finished transfer is a `Response`, 404 included. DNS, timeout, and a bad URL are an `Error`. `"{$e}"` is libcurl's sentence.

`curl::send` uses a handle once and drops it. A `Client` keeps the connection and your defaults. `send` resets the handle, then replays options: library defaults, client, request. Last write wins.

```echo
curl::Client $api = curl::Client();
$api->baseUrl = 'https://echoc.dev';
$api->header('Accept', 'application/json');
$api->timeout(10000);

curl::Response $res = guard $api->send(.get('/libcurl.json')) else ($e) {
    die("{$e}");
}
```

`/path` replaces the path and keeps the origin. Anything else is appended to the base path. An absolute URL ignores the base.

```echo
curl::Query $q = curl::Query();
$q->add('q', 'echo');

curl::Request $again = .get('/libcurl.json');
$again->query($q);
$again->header('X-Trace', 'once');
```

A request header replaces the client's header of the same name. `$req->headers->add` keeps both.

## Status and errors

`success()` is 2xx. `clientError()` is 4xx. `serverError()` is 5xx. `redirect()` is 3xx, only when you are not following.

`file://` can succeed with `status` 0. `success()` is false. The transfer still worked.

`.failOnError` is off. Turn it on and a 4xx or 5xx comes back as `.httpReturnedError`, with no `Response`.

## Bodies

The body lands in `Response::body`. For a file, or for chunks you do not want to keep:

```echo
$req->saveTo($path);    // failed transfer deletes the file. count is $res->bytes
$req->onChunk(function(const string& $chunk) : bool {
    return true;        // false cancels: .abortedByCallback
});
$req->uploadFile($path);
$req->maxBytes(1024);   // .filesizeExceeded. 0 clears a client cap
```

`uploadStream` pulls until the closure returns `''`. Size `-1` is chunked HTTP/1.1.

One body. A `Form` next to `$body`, or a HEAD with a body, fails with `.badFunctionArgument`. POST, PUT, and PATCH send `Content-Length: 0` when the body is empty.

```echo
curl::Request $json = .postJson($url, $json);

curl::Query $fields = curl::Query();
$fields->add('user', 'ada');
curl::Request $form = .postForm($url, $fields);   // urlencoded

curl::Form $parts = curl::Form();
$parts->file('document', $path, 'text/plain');
curl::Request $up = .postForm($url, $parts);      // multipart. curl writes the boundary
```

Do not set `Content-Type` on a multipart body. You would wipe the boundary.

## Defaults

Follow redirects, 10 hops. Ask for `gzip` and `deflate`, and decode them. TLS verification stays at libcurl's default, on. No timeout. No user agent.

```echo
$req->timeout(5000);          // ms. same helpers exist on Client
$req->follow(false);
$req->verifyTls(false);       // only for a host you already know is bad
$req->acceptEncoding(false);  // clears the default, including a client's
$req->set(.proxy, 'http://127.0.0.1:8080');
$req->set(.customRequest, 'PROPFIND');
```

`del` is DELETE. A `curl_off_t` option needs the label: `set(off: .postFieldSize, $n)`. Anything the enums skip goes through `raw` with the number from `curl.h`.

`curl::Easy` is the handle when you do not want `send` in between. `curl::version()` is `curl_version()`.

## Examples

```bash
echoc run -m . examples/get.eco
echoc run -m . examples/client.eco
echoc run -m . examples/download.eco
echoc run -m . examples/upload.eco
echoc test
```

`download.eco` uses `file://` and needs no network. `upload.eco` posts to `https://httpbin.org/post` unless you pass a URL after `--`.

MIT. Copyright (c) 2026 Mario Döring.
