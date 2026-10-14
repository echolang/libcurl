---
description: One request type, the verbs that build it, and what happens to the body.
---

# Requests

Every HTTP call in this library goes through one type: [](api:curl::Request). Method, URL, body and headers are fields. Everything else (timeouts, redirects, TLS, the proxy) is an option pushed onto the same value. Then you `send` it.

```echo
curl::Request $req = .post('https://httpbin.org/post', 'hello');
$req->header('Content-Type', 'text/plain');

curl::Response $res = guard curl::send($req) else ($e) {
    die("{$e}");
}
```

## One way to make a request

There's no `curl::get($url)` next to `curl::post($url, $body)` next to `curl::send($req)`. I left that family out on purpose. A second, thinner entry point per verb is surface that has to stay consistent with the first one forever, and it's always the half that drifts.

The convenience lives in the static constructors instead, and the leading dot reaches them for free:

```echo
curl::send(.get('https://echoc.dev'));
```

`send` wants a `Request`, so `.get(...)` resolves against `Request::get`. Same request as the long form, no extra API.

## Choosing a verb

The common verbs each have a constructor:

```echo
curl::Request $a = .get($url);
curl::Request $b = .head($url);
curl::Request $c = .post($url, $body);
curl::Request $d = .put($url, $body);
curl::Request $e = .patch($url, $body);
curl::Request $f = .del($url);                      // DELETE
curl::Request $g = .postJson($url, '{"a":1}');      // plus Content-Type: application/json
```

`del` and not `delete`, because `delete` reads as a keyword in half the languages nearby. [Forms](./forms.md) covers `postForm`.

The rest of [](api:curl::Method) goes through the constructor. A verb `Method` doesn't name at all is an option, and options go on after the method, so it wins:

```echo
curl::Request $opts = curl::Request(.options, $url);

curl::Request $dav = curl::Request(.get, $url);
$dav->set(.customRequest, 'PROPFIND');
```

## Bodies

`body` is a plain `string`, sent verbatim with its length. A zero byte in it is just a byte, and nothing calls `strlen` on your payload.

A few rules ride along with the method:

- **POST, PUT and PATCH always send a body**, an empty one included, as `Content-Length: 0`. Without it, nginx and plenty of APIs answer 411 Length Required.
- **A GET with a body** goes out as a custom `GET`, because libcurl's own GET drops the body. Odd, but some search APIs want exactly that.
- **A DELETE with a body** is sent as is.
- **A HEAD with a body** fails `send` with `.badFunctionArgument`. libcurl would drop it silently, and I'd rather you hear about it.

There is one body per request. `body`, a [form](./forms.md) and a [streamed upload](./upload.md) are three spellings of it, and setting two at once fails `send` with `.badFunctionArgument`.

## Copying a request

`Request` is a struct with no destructor, so `$b = $a;` is an ordinary deep copy. One template, three variants, no factory:

```echo
curl::Request $template = .get('https://api.example.com/items');
$template->header('Accept', 'application/json');

curl::Request $slow = $template;
$slow->timeout(30000);

curl::Request $html = $template;
$html->header('Accept', 'text/html');
```

`$template` still has its JSON `Accept` and no timeout. The copies don't share headers, options or anything else.

If the thing you're really copying is "these defaults for every request", that's a [Client](./client.md), not a template.
