For passing the most `make test` tests, build wolfSSL with the configure

```
./configure --enable-all --enable-oldtls --enable-tlsv10 --enable-ipv6 'CPPFLAGS=-DWOLFSSL_NO_DTLS_SIZE_CHECK -DOPENSSL_COMPATIBLE_DEFAULTS'
```

Download socat-1.8.0.3.tar.gz and apply patch:

```
curl -O http://www.dest-unreach.org/socat/download/socat-1.8.0.3.tar.gz
tar xvf socat-1.8.0.3.tar.gz
cd socat-1.8.0.3
patch -p1 < socat-1.8.0.3.patch
autreconf -fvi
./configure --with-wolfssl=/usr/local
make
```


Current fail cases seen with `make test` are:

```
FAILED:  146 386 402 410 418 459 460 475 491 492 495
```

- Test 146 is with a DSA certificate and gets a -501 (bad cipher suite). wolfSSL
does not support DSA cipher suites.
- Test 402 "i2v function not yet implemented for Subject Alternative Name"
- Test 475 wolfSSL handles DTLS timeouts internally. Setting so-rcvtimeo on the socket does not affect the timeout.
- Test 491 UDP packet boundary timing race — intermittently fails on any build
- Test 495 Output ordering timing race in POSIX MQ tests


The configure.ac was updated to have `[ ]` instead of `[]` because of autoconf's expected format with AC_DEFINE.
