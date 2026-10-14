---
description: URL-encoded forms for a login, multipart forms for a file, and the header you shouldn't set.
---

# Forms

HTML has two kinds of form, and so does this library. `application/x-www-form-urlencoded` is `a=1&b=2` in the body: fine for a login, useless for a file. `multipart/form-data` sends parts, and a part can be a file. Both go through the same verb, [](api:curl::Request::postForm), and the argument you pass picks the encoding.

## URL-encoded

Hand `postForm` a [](api:curl::Query). The body is the encoded pairs, and the `Content-Type` is set for you:

```echo
curl::Query $login = curl::Query();
$login->add('user', 'mario');
$login->add('password', 'hunter 2');

curl::Request $req = .postForm('https://httpbin.org/post', $login);

echo $req->body; // user=mario&password=hunter%202
```

## Multipart

Hand it a [](api:curl::Form) instead, and you get parts:

```echo
curl::Form $form = curl::Form();
$form->field('title', 'a field next to a file');
$form->file('document', '/tmp/notes.txt', 'text/plain');
$form->upload('generated', 'report.csv', "a,b\n1,2\n", 'text/csv');

curl::Request $req = .postForm('https://httpbin.org/post', $form);
```

Three kinds of part, three helpers:

- `field` is a plain text field, the multipart spelling of `name=value`.
- `file` is a file on disk. It's read while the request goes out and never held in memory. The server sees the path's last segment as the filename.
- `upload` is a file you already have in memory, sent as if it were `$filename` on disk.

The type argument is optional on both file helpers. Leave it out and libcurl guesses from the extension. It knows a handful (`.txt`, `.png`, `.pdf`, ...), and past those a file goes as `application/octet-stream`. Pass the type for a `.csv` or a `.json`.

A `Form` is a value, like the `Request` it rides on. `$form->parts` is a public array of [](api:curl::Part), so anything the helpers don't spell is a field write away. `postForm` builds a POST, and a form works on other methods too: `$put->form = $form;`.

## The Content-Type catch

Here is the catch: **don't set `Content-Type` yourself on a multipart request.** libcurl writes it, and it writes the boundary into it:

```text
Content-Type: multipart/form-data; boundary=------------------------a1b2c3d4
```

Set your own and the boundary is gone. The server gets a body it has no way to split into parts. `postForm` with a `Form` sets no header for exactly this reason.

## What goes wrong

A form is the whole body, so pairing it with something else fails `send` with `.badFunctionArgument`. That covers a form next to a non-empty `body`, a form next to a [streamed upload](./upload.md), and a form on a HEAD.

A file part that can't be read is `.readError`, and it fails **before** anything goes out. libcurl checks the file while the form is built, so the server never sees half a request.

A part whose name, filename or type holds a CR, LF or zero byte is refused too, for the same reason as a [header](./headers.md#what-send-refuses). An empty name is refused because the server has no way to look the field up.

`examples/upload.eco` sends a field, a file and an in-memory upload, then prints where the time went:

```bash
echoc run -m . examples/upload.eco
```
