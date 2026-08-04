`wolfProvider/libcryptsetup/libcryptsetup-wolfprov-routing.patch` routes
libcryptsetup's crypto through wolfProvider. It points the provider load in
`lib/crypto_backend/crypto_openssl.c` at `libwolfprov`, aliases the legacy
provider handle to it so stock OpenSSL crypto is not pulled back into the
context, and reports `[libwolfprov]` in place of `[default]` in the backend
version string (`cryptsetup --debug benchmark 2>&1 | grep 'Crypto backend'`).
libcryptsetup builds a private `OSSL_LIB_CTX`, so this cannot be done with
`OPENSSL_CONF` or any environment variable — the source change is the only
mechanism. The patch contains no test or build-system changes.

It is intentionally version-independent: both hunks carry only context that is
identical across releases. Verified to apply with no fuzz against every upstream
tag from 2.4.1 through 2.8.7, and against the Debian source packages for
bookworm, trixie and sid. It does not apply to 2.4.0 or earlier, which loaded
providers into the OpenSSL default library context.

Build patched packages with `DEB_BUILD_OPTIONS=nocheck`. Two gaps to expect
afterwards: wolfProvider serves no Argon2, so a `luksFormat` using LUKS2's
default Argon2id KDF fails outright rather than writing a non-approved header
(pass `--pbkdf pbkdf2`); and on cryptsetup before 2.7.0 the compiled-in plain
mode hash is still `ripemd160`, so plain mode needs `--hash sha256`.

`libcryptsetup-v2.6.1-wolfprov.patch` adds FIPS and non FIPS wolfProvider
support for libcryptsetup `v2.6.1`. It disables various tests that use out
of bounds or not supported crypto. examples include: `ripemd160`, `whirlpool`,
`blake2b-512`, `blake2s-256`, `stribog512`, `kuznyechik`, `argon2i`, `argon2id`,
and `pbkdf2`. It is pinned to `v2.6.1` and will not apply to 2.7.x or later, so
use the routing patch above for deployments.
