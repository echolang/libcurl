---
description: How a request URL is joined onto a Client's base URL, case by case.
---

# How URLs join

A [client](./client.md) with a `baseUrl` lets each request say only the part that differs. The join follows the rules a browser uses for a link, with one deliberate exception for a plain relative segment.

```echo
curl::Client $api = curl::Client();
$api->baseUrl = 'https://api.example.com/v1';

$api->send(.get('users')); // https://api.example.com/v1/users
```

## The rules

With `https://api.example.com/v1?key=1#top` as the base:

| Request URL | Joined | Why |
|:--|:--|:--|
| `https://other.example/x` | `https://other.example/x` | An absolute URL wins. |
| `//cdn.example.com/x` | `https://cdn.example.com/x` | Protocol-relative: the base's scheme, nothing else. |
| `/health` | `https://api.example.com/health` | A leading `/` replaces everything after the host. |
| `users` | `https://api.example.com/v1/users` | A segment goes onto the base's path with exactly one slash. |
| `?page=2` | `https://api.example.com/v1?page=2` | A query replaces the base's query. |
| `#bottom` | `https://api.example.com/v1?key=1#bottom` | A fragment replaces the base's fragment. |
| `''` | `https://api.example.com/v1?key=1#top` | Empty means the base itself. |

The base's own query never ends up in front of a joined segment. `users` gets the path, not `?key=1`.

"Absolute" means a scheme at the very start. A `://` later in the string doesn't count, so `redirect?url=https://evil.com` is still a segment, and it joins onto the base like one.

With no `baseUrl` at all, the request URL is used as written.

## The catch with a leading slash

This is the one that bites:

```echo
$api->baseUrl = 'https://api.example.com/v1';

$api->send(.get('/health')); // https://api.example.com/health
$api->send(.get('health'));  // https://api.example.com/v1/health
```

`/health` is an **absolute path**. It replaces the base's whole path, `/v1` included, exactly like an `href="/health"` would on a page at `/v1`. If your base URL carries a version prefix, write request paths without the leading slash.

And the deliberate exception: a browser resolving `users` against `/v1` would replace the last segment and give you `/users`. Here it's appended, giving `/v1/users`, slash or no slash at the end of the base. A base URL is a prefix you build on, not a page you're linking from, so I went with what an API client means.
