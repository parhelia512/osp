## How to setup wolfSSL support for standard Zephyr TLS Sockets and RNG (Zephyr 4.3)

wolfSSL can also be used as the underlying implementation for the default Zephyr TLS socket interface.
With this enabled, all existing applications using the Zephyr TLS sockets will now use wolfSSL inside
for all TLS operations. This will also enable wolfSSL as the default RNG implementation. To enable this
feature, first ensure wolfSSL has been added to the west manifest using the instructions from the
README.md here: https://github.com/wolfSSL/wolfssl/tree/master/zephyr

This integration depends on Kconfig options added to the wolfSSL Zephyr module for Zephyr default
TLS support (`WOLFSSL_SESSION_EXPORT`, `WOLFSSL_OPENSSL_EXTRA_X509_SMALL`,
`WOLFSSL_ALWAYS_VERIFY_CB`, and the `native_sim` timer gate extension). They are all in
**wolfSSL >= 5.9.2**, so a released wolfSSL is enough.

One later improvement needs **wolfSSL > 5.9.2** (master, or any newer release) - pull request
[wolfSSL/wolfssl#10983](https://github.com/wolfSSL/wolfssl/pull/10983), merged after
`v5.9.2-stable`: `wc_port.c`'s `z_time()` now reads the `native_sim`/`native_posix` simulator RTC
regardless of the selected C library, instead of only under picolibc/newlib. Without it, a
`native_sim` build on the host libc validates ASN.1 dates against uptime-since-boot. Real targets
are unaffected.

Once the west manifest has been updated, run west update, then patch the Zephyr tree (run inside the
`zephyr` directory). The integration ships as three patches:

- **`zephyr-tls-4.3.1.patch`** - the core wolfSSL backend (the BSD-sockets TLS layer and the
  RNG/CSPRNG). This is all that is required to use wolfSSL for TLS sockets.
- **`zephyr-tls-4.3.1-tests.patch`** - the wolfSSL twister test scenarios and the echo_server
  sample overlay. Apply it only if you want to run the test suite yourself; it is not needed to
  use the integration. It depends on the core patch, so apply the core patch first.
- **`zephyr-tls-4.3.1-mbedtls-decoupling.patch`** - optional. Removes the remaining hard mbedTLS
  dependencies from subsystems whose crypto already runs through the PSA API or the TLS socket
  layer (mcumgr/UpdateHub DTLS, LwM2M, and uuid v5), so an image using wolfSSL plus the wolfPSA
  PSA provider can drop `CONFIG_MBEDTLS` entirely. Apply on top of the core patch. The provider
  side is not part of this patch set: it ships in wolfPSA itself, which is a Zephyr module of its
  own (`zephyr/` subdirectory, `CONFIG_WOLFPSA=y`) - see
  https://github.com/wolfSSL/wolfPSA/tree/master/zephyr.

```
# required:
patch -p1 < /path/to/your/osp/zephyr/4.3/zephyr-tls-4.3.1.patch
# optional, only to run the tests:
patch -p1 < /path/to/your/osp/zephyr/4.3/zephyr-tls-4.3.1-tests.patch
# optional, to drop mbedTLS entirely (needs wolfSSL + the wolfPSA PSA provider):
patch -p1 < /path/to/your/osp/zephyr/4.3/zephyr-tls-4.3.1-mbedtls-decoupling.patch
```

### Minimum prj.conf

Use `tests/net/socket/tls/overlay-wolfssl.conf` as a template. At minimum the application needs
`CONFIG_MBEDTLS=n`, `CONFIG_WOLFSSL=y`, and Zephyr POSIX support (`CONFIG_POSIX_API=y`,
`CONFIG_POSIX_TIMERS=y`, `CONFIG_POSIX_THREADS=y`). Size `CONFIG_COMMON_LIBC_MALLOC_ARENA_SIZE`
to the application footprint.

### Configuration options

Kconfig help text is authoritative:
- wolfSSL module: https://github.com/wolfSSL/wolfssl/blob/master/zephyr/Kconfig
- Zephyr TLS socket layer: `subsys/net/lib/sockets/Kconfig` (after applying the patch)

Options added by this integration:

| Kconfig | Purpose |
|---|---|
| `WOLFSSL_VERIFY_CALLBACK` | Enable wolfSSL-native per-cert verify callback via the `TLS_CERT_VERIFY_CALLBACK_WOLFSSL` socket option |

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
- OCSP and CRL handling is library-internal on both backends; there is no Zephyr socket-option API for it.

### Run Zephyr TLS samples

```
west build -b <your_board> samples/net/sockets/echo_server -DEXTRA_CONF_FILE=overlay-wolfssl.conf
```

### Run Zephyr TLS tests

```
west build -b <your_board> tests/net/socket/tls_ext/ -DEXTRA_CONF_FILE=overlay-wolfssl.conf
```

```
west build -b <your_board> tests/net/socket/tls/ -DEXTRA_CONF_FILE=overlay-wolfssl.conf
```

### References

- Zephyr TLS sockets: https://docs.zephyrproject.org/latest/connectivity/networking/api/sockets.html
- wolfSSL Zephyr module: https://github.com/wolfSSL/wolfssl/tree/master/zephyr
