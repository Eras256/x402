---
'@x402/extensions': patch
---

Fix `extractDiscoveryInfo` building a broken canonical URL (`"null/..."`) for resource URLs on schemes without a WHATWG-defined origin, such as `mcp://tool/{toolName}`. Canonicalization is now skipped for any opaque-origin scheme, not just `mcp://`, and the raw resource URL is used as canonical instead.
