# Safe Blob URL

A Web Platform API proposal for Blob URL

Jun Kokatsu, Feb 2023 (©2023, Google)

## TL;DR: Add a crossOrigin option to `Blob`

Blob URL is useful for loading locally available resources. However it also leads to XSS bugs.
- [XSS on WhatsApp Web](https://blog.checkpoint.com/2017/03/15/check-point-discloses-vulnerability-whatsapp-telegram/).
- [XSS on Shopify](https://hackerone.com/reports/1276742).
- [XSS on chat.mozilla.org](https://gccybermonks.com/posts/xss-mozilla/).





