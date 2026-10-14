---
description: Where the C library comes from on macOS, Linux and Windows, and how to check which one you got.
---

# Finding libcurl

`epm add` gives you the Echo half. The other half is libcurl itself, and where that lives is a property of the machine, not of this repository. This page is about getting the linker and the JIT to find it.

## Linking is the whole install

The manifest of this module already carries the link line, so you never write it yourself. On macOS and Linux it is plain `-lcurl`:

```echo
#[link: lib "curl"]
```

That rides along with the module. Whatever program depends on `echolang/libcurl` links against the system's libcurl, dynamically, with no flags from you. The rest of this page is the cases where "the system's libcurl" needs help.

## macOS

macOS ships libcurl, so `echoc run`, `echoc test` and `echoc build` all work out of the box.

If you'd rather use Homebrew's newer curl than the system one, point the linker at it:

```bash
echoc build --link search:/opt/homebrew/opt/curl/lib
```

## Linux

You need the runtime, which every distro already has (`libcurl4` on Debian and Ubuntu). For `echoc build`, that's enough.

Here is the catch: `echoc run` and `echoc test` also need the `-dev` package. They go through the JIT, and the JIT `dlopen`s the unversioned `libcurl.so`, which only the dev package installs.

```bash
sudo apt install libcurl4-openssl-dev
```

## Windows

Windows has no system libcurl to link against, so this module links a **static** one there, with Schannel as the TLS stack. The module already names the system libraries that archive needs (`ws2_32`, `crypt32`, `bcrypt` and friends). What you provide is the archive and zlib. vcpkg builds both:

```bash
vcpkg install curl:x64-windows-static
```

Then point the build at them. The zlib archive is `zs.lib` on current vcpkg ports, `zlib.lib` on older ones:

```bash
echoc test --link "search:C:\vcpkg\installed\x64-windows-static\lib" --link "lib:zs"
```

That's exactly what this repository's CI does on `windows-2022`.

## A static binary

On macOS and Linux you can also bake libcurl into the executable, OpenSSL included:

```bash
echoc build -o app --define static_curl main.eco
```

`static_curl` swaps the link line for static `curl`, `ssl` and `crypto`. It only works for `echoc build`. `run` and `test` go through the JIT, which can't open an archive, so leave the define off for those.

## Which libcurl did I get?

[](api:curl::version) answers `curl_version()`, straight from the library that actually loaded:

```echo
echo curl::version(); // libcurl/8.7.1 SecureTransport (LibreSSL/3.3.6) zlib/1.2.12 nghttp2/1.64.0
```

A few features lean on a minimum version. Forms need 7.56 (`curl_mime`), progress needs 7.32 (`XFERINFOFUNCTION`), and turning a 303 into a GET for PUT, PATCH and DELETE needs 8.13. An older libcurl still follows that 303, it just keeps the method.
