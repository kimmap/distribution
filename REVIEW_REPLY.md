Thanks for pointing this out. Fixed in 89fc7049: `200 OK` and `206 Partial Content` responses now use the already-resolved blob digest when `Docker-Content-Digest` is missing, while preserving any digest provided upstream.

I extended the existing redirect tests to check the digest for partial responses and the full `200 OK` fallback caused by a stale `If-Range`. These assertions fail without the fix, and the client/proxy short tests pass locally with it.
