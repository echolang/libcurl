---
description: Sending a file or a stream as the request body, without reading it into memory first.
---

# Uploads

A request body is a `string` by default, and for most API calls that's exactly right. For a 2 GB backup it isn't. A request's [](api:curl::Source) says where the body comes from instead, and libcurl pulls the bytes as it sends them, so the body never has to exist in memory all at once.

## From a file

`uploadFile` sends a file as the body, read while it goes out:

```echo
curl::Request $req = curl::Request(.put, 'https://example.com/backups/today.tar');
$req->uploadFile('/var/backups/today.tar');

curl::Response $res = guard curl::send($req) else ($e) {
    die("{$e}");
}
```

The file's size goes out as `Content-Length`. A file that can't be read is `.fileCouldNotReadFile`, and that's checked before anything else happens. If the same request was also saving the response with `saveTo`, the target file is never even created.

An upload is a PUT by default, the way `curl -T` is. Any other method keeps its own verb, so this sends a POST, like `curl -T file -X POST` would:

```echo
curl::Request $req = curl::Request(.post, 'https://example.com/ingest');
$req->uploadFile('/tmp/events.ndjson');
```

## From a closure

When the body is being produced as you go (compressed on the fly, read from another socket, generated), `uploadStream` calls a closure for each chunk, until it answers `''`:

```echo
class Lines
{
    public int32 $left;

    constructor(int32 $count)
    {
        $this->left = $count;
    }
}

Lines $lines = Lines(3);

curl::Request $req = curl::Request(.put, 'https://example.com/log');
$req->uploadStream(function() : string {
    if ($lines->left == 0) {
        return ''; // the end
    }

    $lines->left = $lines->left - 1;

    return "line {$lines->left}\n";
});
```

Same trick as `onChunk` on a [download](./download.md#in-chunks): the closure captures a class so its position survives between calls.

Without a size, HTTP/1.1 sends the body chunked, with `Transfer-Encoding: chunked`. When you know the total up front, pass it, and it goes out as `Content-Length` instead:

```echo
$req->uploadStream($next, 1048576);
```

## One body at a time

`body`, a [form](./forms.md) and an upload are three spellings of the same thing, and a request has one. Setting two fails `send` with `.badFunctionArgument`.

`uploadFile` and `uploadStream` clear `body` for you, so a request built with `.post($url, 'x')` and then given a file doesn't trip over its old body.

Here is the catch: assigning `$req->source` directly does **not** clear `body`. The field is public, but the helpers are the spelling that keeps the request consistent. Use them.

Progress works on the upload side too. `Progress::uploaded` counts what went out. [Progress and cancelling](./progress.md) has that part.
