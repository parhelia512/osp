## How to setup wolfSSL support for standard Zephyr TLS Sockets and RNG (Zephyr 4.4)

wolfSSL can also be used as the underlying implementation for the default Zephyr TLS socket interface.
With this enabled, all existing applications using the Zephyr TLS sockets will now use wolfSSL inside
for all TLS operations. This will also enable wolfSSL as an alternative RNG implementation. To enable
this feature, first ensure wolfSSL has been added to the west manifest using the instructions from the
README.md here: https://github.com/wolfSSL/wolfssl/tree/master/zephyr

This integration requires **wolfSSL > 5.9.2** (master, or any newer release). 5.9.2 itself is the
baseline for the socket backend: earlier revisions gate `wolfSSL_clear()` - used by the
per-session reset path in `sockets_tls.c` - behind `OPENSSL_EXTRA`/`WOLFSSL_WPAS_SMALL`, and lack
the upstream `OPENSSL_EXTRA_X509_SMALL` gate arm that lets the backend validate credentials
without forcing `KEEP_PEER_CERT`. Zephyr 4.4 support then needs a few changes that landed just
after that tag, detailed below.

The Kconfig options the backend builds on - `WOLFSSL_SESSION_EXPORT`,
`WOLFSSL_OPENSSL_EXTRA_X509_SMALL` and `WOLFSSL_ALWAYS_VERIFY_CB` in the wolfSSL Zephyr module -
are part of that 5.9.2 baseline.

Those post-5.9.2 changes are: a `wolfio.h` include guard for the retyped 4.4 zsock prototypes, an
`arpa/inet.h` include in `test.h`, a malloc-arena bump in the `wolfssl_tls_sock` sample, and a
`native_sim` simulation fix - `wc_port.c`'s `z_time()` used to
consult the simulator RTC only when `CONFIG_RTC` *and* picolibc/newlib were selected, so under the
host libc ASN.1 date validation fell back to uptime-since-boot; it now reads the simulator RTC on
native boards regardless of the selected libc (native simulation only, real targets unchanged).

Those are merged in pull request
[wolfSSL/wolfssl#10983](https://github.com/wolfSSL/wolfssl/pull/10983), which landed after the
`v5.9.2-stable` tag. Any wolfSSL newer than 5.9.2 therefore works: a released version once the
next one ships, or a master revision containing that pull request in the meantime.

Then patch the Zephyr tree (run inside the `zephyr` directory). The integration ships as three
patches, all generated against the Zephyr **4.4.1** release:

- **`zephyr-tls-4.4.1.patch`** - the core wolfSSL backend (the BSD-sockets TLS layer and the
  RNG/CSPRNG). This is all that is required to use wolfSSL for TLS sockets.
- **`zephyr-tls-4.4.1-tests.patch`** - the wolfSSL twister test scenarios and the echo_server
  sample overlay. Apply it only if you want to run the test suite yourself; it is not needed to
  use the integration. It depends on the core patch, so apply the core patch first.
- **`zephyr-tls-4.4.1-mbedtls-decoupling.patch`** - optional. Removes the remaining hard mbedTLS
  dependencies from subsystems whose crypto already runs through the PSA API or the TLS socket
  layer (mcumgr/UpdateHub DTLS, LwM2M, WireGuard, and uuid v5), so an image using wolfSSL plus the
  wolfPSA PSA provider can drop `CONFIG_MBEDTLS` entirely. Apply on top of the core patch. The
  provider side is not part of this patch set: it ships in wolfPSA itself, which is a Zephyr
  module of its own (`zephyr/` subdirectory, `CONFIG_WOLFPSA=y`) - see
  https://github.com/wolfSSL/wolfPSA/tree/master/zephyr.

```
# required:
patch -p1 < /path/to/your/osp/zephyr/4.4/zephyr-tls-4.4.1.patch
# optional, only to run the tests:
patch -p1 < /path/to/your/osp/zephyr/4.4/zephyr-tls-4.4.1-tests.patch
# optional, to drop mbedTLS entirely (needs wolfSSL + the wolfPSA PSA provider):
patch -p1 < /path/to/your/osp/zephyr/4.4/zephyr-tls-4.4.1-mbedtls-decoupling.patch
```

The tests patch also includes one small, wolfSSL-independent test fix -
`sizeof(sec_tag_list_verify_none)` in `tests/net/lib/http_server/tls/src/main.c`, which the
`net.http.server.tls` scenario needs on 64-bit builds. That fix is being submitted upstream
separately and is expected to land in Zephyr 4.4.2; if you apply the tests patch to a tree that
already contains it, drop the corresponding one-line hunk.

The 4.4 mbedTLS module also requires the `tf-psa-crypto` west project - make sure it is in
your manifest's allowlist before `west update`.

### Minimum prj.conf

Use `tests/net/socket/tls/prj_wolfssl.conf` as a template. At minimum the application needs
`CONFIG_MBEDTLS=n`, `CONFIG_WOLFSSL=y`, and Zephyr POSIX support (`CONFIG_POSIX_API=y`,
`CONFIG_POSIX_TIMERS=y`, `CONFIG_POSIX_THREADS=y`; without `CONFIG_POSIX_API` also set
`CONFIG_POSIX_SYSTEM_INTERFACES=y` - Zephyr 4.4 gates the POSIX option groups on it).
Size `CONFIG_COMMON_LIBC_MALLOC_ARENA_SIZE` to the application footprint.

### What changed compared to the Zephyr 4.3 integration

Zephyr 4.4 restructured the TLS socket layer and the random subsystem; the integration follows:

- **Per-session TLS contexts / DTLS multi-client servers.** Upstream 4.4 moved the TLS session
  state into per-session contexts (`struct tls_session_context`), allowing a DTLS server socket
  to serve multiple clients concurrently (bounded by
  `CONFIG_NET_SOCKETS_TLS_MAX_SESSION_CONTEXTS`). The wolfSSL backend implements full parity:
  one `WOLFSSL` object per session sharing the per-socket `WOLFSSL_CTX`, with incoming datagrams
  matched to sessions by peer address. Session matching by DTLS Connection ID is **not**
  supported on the wolfSSL backend (the `TLS_DTLS_CID*` socket options return `-ENOPROTOOPT`,
  as in 4.3); the upstream CID-based address-migration test reports as skipped under the
  wolfssl scenarios.
- **Socket option naming.** Upstream renamed the TLS socket options to `ZSOCK_TLS_*` (with
  legacy `TLS_*` aliases in `<zephyr/net/net_compat.h>`). The integration adds
  `ZSOCK_TLS_CERT_VERIFY_CALLBACK_WOLFSSL` (21) plus a `TLS_CERT_VERIFY_CALLBACK_WOLFSSL`
  compat alias, and the option struct is `struct zsock_tls_cert_verify_cb_wolfssl`
  (compat alias `tls_cert_verify_cb_wolfssl`).
- **TLS handshake timeout.** Handshakes during `connect()`/`accept()` are now bounded by the
  upstream `CONFIG_NET_SOCKETS_TLS_CONNECT_TIMEOUT` (default 10 s) on both backends.
- **RNG.** Zephyr 4.4 removed `random_ctr_drbg.c` (the CSPRNG is PSA-based now). The wolfSSL
  RNG integration is therefore a new CSPRNG choice option `CONFIG_WOLFSSL_CSPRNG_GENERATOR`
  (file `subsys/random/random_wolfssl.c`, wolfSSL Hash-DRBG seeded from the entropy driver).
  Select it inside `choice CSPRNG_GENERATOR_CHOICE` instead of the deprecated
  `CTR_DRBG_CSPRNG_GENERATOR`.
- **Minimum TLS version.** Upstream 4.4 enforces a minimum TLS version derived from the socket
  protocol. The wolfSSL backend keeps the 4.3 exact-version method selection
  (`wolfTLSv1_2/1_3_*_method`), which is stricter: an `IPPROTO_TLS_1_2` socket negotiates
  exactly TLS 1.2 and will not upgrade to 1.3.
- **Kconfig rename (action required when migrating from 4.3).** The option gating the
  wolfSSL-style verify callback was renamed from `CONFIG_WOLFSSL_VERIFY_CALLBACK` to
  `CONFIG_NET_SOCKETS_TLS_WOLFSSL_VERIFY_CALLBACK` (the old name risked colliding with the
  wolfSSL module's own Kconfig namespace). A 4.3-era `prj.conf` still setting the old name
  fails the build with `error: Aborting due to Kconfig warnings` /
  `attempt to assign the value 'y' to the undefined symbol WOLFSSL_VERIFY_CALLBACK` -
  rename the option in your application config.

### Configuration options

Kconfig help text is authoritative:
- wolfSSL module: https://github.com/wolfSSL/wolfssl/blob/master/zephyr/Kconfig
- Zephyr TLS socket layer: `subsys/net/lib/sockets/Kconfig` (after applying the patch)

Options added by this integration:

| Kconfig | Purpose |
|---|---|
| `NET_SOCKETS_TLS_WOLFSSL_VERIFY_CALLBACK` | Enable wolfSSL-native per-cert verify callback via the `TLS_CERT_VERIFY_CALLBACK_WOLFSSL` socket option (named `WOLFSSL_VERIFY_CALLBACK` in the 4.3 integration) |
| `WOLFSSL_CSPRNG_GENERATOR` | Use the wolfSSL DRBG as the system CSPRNG (`sys_csrand_get`) |

`CONFIG_NET_SOCKETS_SOCKOPT_TLS` force-selects the wolfSSL module options the backend cannot
work without, so an application config does not have to set them:

| Kconfig | Purpose |
|---|---|
| `WOLFSSL_OPENSSL_EXTRA_X509_SMALL` | Exposes `WOLFSSL_X509_STORE_CTX::current_cert`, which the verify callback reads for the leaf CN/SAN match |
| `WOLFSSL_ALWAYS_VERIFY_CB` | Invoke verify callback on success in addition to failure - without it the CN/SAN match is silently bypassed |
| `WOLFSSL_SESSION_CACHE` | wolfSSL compiles the session API out under `NO_SESSION_CACHE`, which the TLS layer needs to link |

`CONFIG_WOLFSSL_SNI` is optional: with it disabled the Server Name Indication extension is not
sent, but a configured `TLS_HOSTNAME` still drives the certificate CN/SAN match that
`TLS_PEER_VERIFY` relies on. `CONFIG_WOLFSSL_CRYPTO_ONLY` is incompatible with TLS sockets and
is rejected with a build error.

Session resumption is a deliberate opt-in: set `CONFIG_WOLFSSL_SESSION_EXPORT=y` (external session
cache) if you want it - the integration's session store/restore are no-ops without it. Existing
wolfSSL module options (`WOLFSSL_DTLS`, `WOLFSSL_ALPN`, `WOLFSSL_PSK`, `WOLFSSL_TLS_VERSION_1_3`,
`WOLFSSL_MAX_FRAGMENT_LEN`) are opt-in as usual.

### Limitations

- A client that never sets `TLS_HOSTNAME` is rejected under `TLS_PEER_VERIFY_REQUIRED`, because
  there is no name to check the peer certificate against. This matches the mbedTLS backend, which
  sets an empty hostname for that case so no certificate can match. Set `TLS_HOSTNAME`, or use
  `TLS_PEER_VERIFY_OPTIONAL` and read `TLS_CERT_VERIFY_RESULT`, or `TLS_PEER_VERIFY_NONE`.
- TLS 1.0 and 1.1 disabled (`NO_OLD_TLS`).
- The mbedTLS-style `TLS_CERT_VERIFY_CALLBACK` socket option is not supported on the wolfSSL backend.
- `TLS_CERT_NOCOPY` has no effect - certificates are always copied.
- TLS 1.3 0-RTT not wired on the wolfSSL path.
- DTLS Connection ID (`TLS_DTLS_CID*`) is not supported; DTLS server sessions are matched by
  peer address only.
- OCSP and CRL handling is library-internal on both backends; there is no Zephyr socket-option API for it.
- The test suites use the wolfSSL-internal `SendAlert()` API (via `<wolfssl/internal.h>`) to
  inject fatal alerts; that is a test-only dependency that may need attention on wolfSSL uprevs.

### Run Zephyr TLS samples

```
west build -b <your_board> samples/net/sockets/echo_server -DEXTRA_CONF_FILE=overlay-wolfssl.conf
```

### Run Zephyr TLS tests

```
west twister -p native_sim -s tests/net/socket/tls/net.socket.tls.wolfssl
west twister -p native_sim -s tests/net/socket/tls_ext/net.socket.tls.ext.wolfssl
west twister -p native_sim -s tests/net/socket/tls_ext/net.socket.tls.ext.wolfssl.verify_cb
west twister -p native_sim -s tests/subsys/random/rng/crypto.rng.random_wolfssl
```

(Additional wolfssl-tagged scenarios exist for `tests/net/socket/register`,
`tests/net/lib/tls_credentials`, `tests/net/lib/http_server/tls` and
`tests/net/lib/coap_server`: `west twister -p native_sim --tag wolfssl`.)

### References

- Zephyr TLS sockets: https://docs.zephyrproject.org/latest/connectivity/networking/api/sockets.html
- wolfSSL Zephyr module: https://github.com/wolfSSL/wolfssl/tree/master/zephyr
