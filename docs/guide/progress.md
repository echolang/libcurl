---
description: Watching a transfer move, and stopping it from inside or from another thread.
---

# Progress and cancelling

A long transfer wants two things: a way to show how far along it is, and a way to stop it. In this library both are one closure, set with `onProgress`, which libcurl calls as the transfer moves. It hands you a [](api:curl::Progress) and asks you a yes or no question: keep going?

## Watching a transfer

```echo
curl::Request $req = .get('https://example.com/big.iso');
$req->saveTo('/tmp/big.iso');
$req->onProgress(function(const curl::Progress& $p) : bool {
    echo "{$p->downloaded} of {$p->downloadTotal} bytes";

    return true;
});
```

The closure runs roughly once a second, and whenever bytes arrive. `Progress` has four counts:

- `downloaded` and `downloadTotal` for the response body.
- `uploaded` and `uploadTotal` for the request body.

A total is **zero until libcurl knows it**, and it stays zero when the server never says (no `Content-Length`, a chunked response). So guard any percentage against dividing by zero.

## Cancelling from inside

Echo has no exceptions, so stopping a transfer is a return value. Answer `false` and `send` fails with `.abortedByCallback`:

```echo
$req->onProgress(function(const curl::Progress& $p) : bool {
    return $p->downloaded < 1000000; // a megabyte is plenty
});

result<curl::Response, curl::Error> $out = curl::send($req);

if ($out->failed() && $out->failure() == curl::Error::abortedByCallback) {
    echo 'stopped';
}
```

That's the whole cancel story. `onChunk` on a [download](./download.md#in-chunks) works the same way, with the same error.

And yes, a `saveTo` file is removed when you cancel. A cancelled download is a failed one.

## Cancelling from another thread

`send` blocks the thread it runs on. A cancel button lives somewhere else. So the closure reads a flag, and the other thread sets it.

The flag has to be safe to share. That's an `atomic<bool>` in a class marked `#[atomic]`, because the class handle is reference-counted from two threads:

```echo
use std::thread;

#[atomic]
class Cancel
{
    public atomic<bool> $stop;

    constructor()
    {
        $this->stop = atomic<bool>(false);
    }
}

Cancel $cancel = Cancel();

curl::Request $req = .get('https://example.com/big.iso');
$req->onProgress(function(const curl::Progress& $p) : bool {
    return !$cancel->stop->load();
});

thread::task<bool> $worker = guard thread::spawn(function() : bool {
    return !curl::send($req)->failed();
}) else ($e) {
    die($e->message());
}

// later, from the button
$cancel->stop->store(true);
```

The next time libcurl calls the closure, it reads `true`, answers `false`, and the transfer on the worker thread fails with `.abortedByCallback`. There's no second mechanism and no handle to poke from outside. It's the same return value, decided somewhere else.

`examples/download.eco` reports progress for a `file://` download:

```bash
echoc run -m . examples/download.eco
```
