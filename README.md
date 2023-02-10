# Safe Blob URL

A Web Platform API proposal for Blob URL

Jun Kokatsu, Feb 2023 (©2023, Google)

## TL;DR: Add a crossOrigin option to `Blob`

Expose a `crossOrigin` option to [BlobPropertyBag](https://w3c.github.io/FileAPI/#dfn-BlobPropertyBag) in the [Blob constructor](https://developer.mozilla.org/en-US/docs/Web/API/Blob/Blob) and a read-only `crossOrigin` property in [Blob instances](https://developer.mozilla.org/en-US/docs/Web/API/Blob#instance_properties).

```
// script on https://example.com
const untrustedHTML = '<script>alert(document.domain)</script>';
const blob = new Blob([untrustedHTML], {
                 type: 'text/html',
                 crossOrigin: true
             });
if ('crossOrigin' in Blob.prototype && blob.crossOrigin) {
  const url = URL.createObjectURL(blob);
  console.log(url); // blob:https://[958c8e12-9f61-43a0-950a-56ecb19d3028]/958c8e12-9f61-43a0-950a-56ecb19d3028
}
```

A cross-origin Blob URL is special in a few ways.
1. It has a format of `blob:scheme://[UUID]/UUID`.
2. It is has a unique non-opaque origin (e.g. `https://[b9b68b26-a98b-4ad6-b089-33d2afa96944]`).
3. It does *NOT* inherit CSP from the creator.
4. It is treated as cross-site to other URLs (except itself) when rendered as a document (e.g. in [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation/)).

## Why do we need this?

It aims to solve 2 problems.

### XSS

Blob URL is useful for loading locally available resources. However it also leads to XSS bugs.

1. [XSS on WhatsApp Web](https://blog.checkpoint.com/2017/03/15/check-point-discloses-vulnerability-whatsapp-telegram/).
2. [XSS on Shopify](https://hackerone.com/reports/1276742).
3. [XSS on chat.mozilla.org](https://gccybermonks.com/posts/xss-mozilla/).

The cross-origin Blob URL is designed in a way that these XSS won't happen (because a script will execute in a unique origin).

### A native alternative to sandbox domains

Many Web apps require a place to host user contents (e.g. `googleusercontent.com`, `dropboxusercontent.com`, etc) to safely render those. And doing so securely (e.g. to avoid [cookie bomb](https://speakerdeck.com/filedescriptor/the-cookie-monster-in-your-browsers?slide=26) and [Spectre](https://security.googleblog.com/2021/03/a-spectre-proof-of-concept-for-spectre.html)) a site would need to register a "sandbox domain", add it to [public suffix list](https://publicsuffix.org/), and then host user contents in randomly generated subdomains.

The cross-origin Blob URL allows rendering user contents in a cross-site context without such setup.
