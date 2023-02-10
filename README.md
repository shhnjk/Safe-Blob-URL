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
2. It has a unique non-opaque origin (e.g. `https://[b9b68b26-a98b-4ad6-b089-33d2afa96944]`).
3. It does *NOT* inherit CSP from the creator.
4. It is treated as cross-site to other URLs (except itself) when rendered as a document (e.g. in [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation/)).

## Why do we need this?

It aims to solve 2 problems.

### XSS through Blob URLs

Blob URL is useful for loading locally available resources. However it also leads to XSS bugs.

1. [XSS on WhatsApp Web](https://blog.checkpoint.com/2017/03/15/check-point-discloses-vulnerability-whatsapp-telegram/).
2. [XSS on Shopify](https://hackerone.com/reports/1276742).
3. [XSS on chat.mozilla.org](https://gccybermonks.com/posts/xss-mozilla/).

The cross-origin Blob URL is designed in a way that these XSS won't happen (because a script will execute in a unique origin).

### A native alternative to sandbox domains

Many Web apps require a place to host user contents (e.g. `usercontent.goog`, `dropboxusercontent.com`, etc) to safely render them. In order to do so securely (e.g. to avoid XSS, [cookie bomb](https://speakerdeck.com/filedescriptor/the-cookie-monster-in-your-browsers?slide=26), and [Spectre](https://security.googleblog.com/2021/03/a-spectre-proof-of-concept-for-spectre.html) attacks), a site needs to register a [sandbox domain](https://security.googleblog.com/2012/08/content-hosting-for-modern-web.html), add it to the [public suffix list](https://publicsuffix.org/), and then host user contents in randomly generated subdomains. However this is not something that any site can afford due to engineering and maintenance cost.

The cross-origin Blob URL provides a way to render user contents in a cross-site context without such setup.

## FAQ

### Why not opaque Blob URLs?

Good question! There has been [discussion](https://github.com/w3c/FileAPI/issues/74) around adding a way to create opaque Blob URLs.

However, there are few issues in opaque Blob URLs.

1. Access to cetain APIs are prohibited (e.g. [cookie](https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie), [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), etc). Therefore code that touches any prohibited API causes breakage. Which isn't ideal for this proposal, because one of the goals is to provide a native alternative to sandbox domains.
2. Opaque origins are serialized to "null". This makes it difficult for a Blob URL creator to send [postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)s to the Blob URL iframe and ensure that it is received to the intended page (because any other opaque origin will have "null" origin).

Therefore, I think cross-origin Blob URLs are better!

### The format of cross-origin Blob URLs look weird. Why did you choose that?

While I'm okay with changing it to `blob:scheme://UUID/UUID` or similar other format, here are some reasons.

1. It is a [clever idea](https://github.com/whatwg/html/issues/3585).
2. All requests that require an origin header will have `Origin: https://[UUID]` for cross-origin Blob URLs. This form of origin is very similar to [IPv6 URLs](https://www.rfc-editor.org/rfc/rfc2732.html). But as far as I know, the `https://UUID`  origin format does not exist today. Therefore, I thought it will cause less breakages.

### Who is the creator of a cross-origin Blob URL?

A context (e.g. Window, Worker, etc) which called [`URL.createObjectURL`](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL) method to create a Blob URL is the creator (i.e. not the context which created a [Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob) object).

### Is there a way for a cross-origin Blob URL to get its creator's origin?

Yes!!!
We'd like to expose `URL.getCreatorOrigin` method to the [URL interface](https://developer.mozilla.org/en-US/docs/Web/API/URL).

```
// script on blob:https://[UUID]/UUID
window.addEventListener('message', event => {
  if (event.origin === URL.getCreatorOrigin(location.href)) {
    console.log('message from a trusted creator!');
  }
});
```

We could avoid exposing this method by including the creator origin in a cross-origin Blob URL (e.g. `blob:https://[UUID]/example.com`), but I have a slight concern of phishing if the Blob URL is rendered on a top-level page.

### Can we apply CSP or sandbox to a cross-origin Blob URL?

Yes! You can use iframe sandbox and/or [CSP Embedded Enforcement](https://w3c.github.io/webappsec-cspee/) to apply CSP and/or sandbox.

```
<iframe sandbox="allow-scripts" csp="default-src 'self';" src="blob:https://[958c8e12-9f61-43a0-950a-56ecb19d3028]/958c8e12-9f61-43a0-950a-56ecb19d3028"></iframe>
```

You can also add a `<meta>` tag with CSP in the Blob URL content to enforce CSP (but not sandbox).

### If CSP is not inherited to cross-origin Blob URLs, isn't it a CSP bypass?

No, because cross-origin Blob URLs are treated as a cross-origin URL, it's similar to embedding any other cross-origin pages (which have their own CSP settings).

### Is there a way to block cross-origin Blob URLs in iframes?

If there are sites which deploys CSP such as `frame-src 'self' blob:;` and wish to block cross-origin Blob URLs, we could add a keyword like `'deny-unique-blob'` for `frame-src` to specifically block cross-origin Blob URLs.

### Who is allowed to fetch or navigate to a cross-origin Blob URL?

Only pages which are same-origin with the cross-origin Blob URL creator can fetch or navigate to the cross-origin Blob URL.
