---
title: Issue with SSLHandshakeException in Nutch v1.20
date: 2025-1-10 19:06:00 +0800
categories: [Crawling, Nutch]
tags: [https,tls,ssl,handshake,exception,nutch,protocol-http,plugin]
description: How to resolve SSLHandshakeException in Nutch v1.20 with protocol-http plugin. 
---

## 0. Background
I recently encountered an SSLHandshakeException while crawling a website with `Nutch v1.20`. I will use `example.com` as an illustration. Below is the stack trace:

```java
2025-01-10 15:12:32,399 ERROR o.a.n.p.h.Http [main] Failed to get protocol output
org.apache.nutch.protocol.http.api.HttpException: SSL connect to https://example.com failed with: Remote host terminated the handshake
	at org.apache.nutch.protocol.http.HttpResponse.<init>(HttpResponse.java:156) ~[classes/:?]
	at org.apache.nutch.protocol.http.Http.getResponse(Http.java:65) ~[classes/:?]
	at org.apache.nutch.protocol.http.api.HttpBase.getProtocolOutput(HttpBase.java:354) [classes/:?]
	at org.apache.nutch.protocol.http.api.HttpBase.main(HttpBase.java:697) [classes/:?]
	at org.apache.nutch.protocol.http.Http.main(Http.java:59) [classes/:?]
Caused by: javax.net.ssl.SSLHandshakeException: Remote host terminated the handshake
	at java.base/sun.security.ssl.SSLSocketImpl.handleEOF(SSLSocketImpl.java:1715) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.decode(SSLSocketImpl.java:1514) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.readHandshakeRecord(SSLSocketImpl.java:1421) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:455) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:426) ~[?:?]
	at org.apache.nutch.protocol.http.HttpResponse.<init>(HttpResponse.java:136) ~[classes/:?]
	... 4 more
Caused by: java.io.EOFException: SSL peer shut down incorrectly
	at java.base/sun.security.ssl.SSLSocketInputRecord.read(SSLSocketInputRecord.java:489) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketInputRecord.readHeader(SSLSocketInputRecord.java:478) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketInputRecord.decode(SSLSocketInputRecord.java:160) ~[?:?]
	at java.base/sun.security.ssl.SSLTransport.decode(SSLTransport.java:111) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.decode(SSLSocketImpl.java:1506) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.readHandshakeRecord(SSLSocketImpl.java:1421) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:455) ~[?:?]
	at java.base/sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:426) ~[?:?]
	at org.apache.nutch.protocol.http.HttpResponse.<init>(HttpResponse.java:136) ~[classes/:?]
	... 4 more
Status: exception(16), lastModified=0: org.apache.nutch.protocol.http.api.HttpException: SSL connect to https://example.com failed with: Remote host terminated the handshake
```

## 1. What is SSLHandshakeException
`SSLHandshakeException` is an exception in Java, part of `the javax.net.ssl` package. It typically occurs during the `SSL/TLS handshake process` when the client and server fail to establish a secure connection[^1].

SSLHandshakeException is usually caused by the following:

### 1.1 Certificate Issues
  * Invalid Certificate:<br>The server provides an invalid certificate, such as one that is expired, untrusted, or has a hostname mismatch.
  * Self-Signed Certificate: <br>If the server uses a self-signed certificate and the client has not added it to its trust store, the handshake will fail.
  * Incomplete Certificate Chain: <br>The server does not provide the full certificate chain, preventing the client from verifying the certificate.

### 1.2 Protocol Version Mismatch
The client and server do not support a common TLS protocol version. For example, the client tries to use TLS 1.3, but the server only supports TLS 1.2 or earlier.
### 1.3 Cipher Suite Mismatch
There is no shared encryption algorithm or cipher suite between the client and server.

## 2. How to conduct an investigation
I can open this website in a browser, so it is most likely related to the `Protocol Version Mismatch` mentioned above in section `1.2`. You can use the following two methods to check the HTTPS protocol versions supported by the web server.

### 2.1 nmap command
```bash
nmap -sV ssl-enum-ciphers -p 443 example.com
```
`Note:` If the web server is hosted by a third party like `Cloudflare`, you may not get the expected results.
### 2.2 openssl command
```bash
openssl s_client -connect example.com:443 -tls1_1
```
In the above command, `tls1_1` refers to `TLSv1.1`, `tls1_2` refers to `TLSv1.2`, and so on.

In my case, the web server only supports `TLSv1.2` and `TLSv1.3` and doesn't support `TLSv1` and `TLSv1.1`.
```bash
openssl s_client -connect example.com:443 -tls1_1
CONNECTED(00000003)
140618722170176:error:141E70BF:SSL routines:tls_construct_client_hello:no protocols available:../ssl/statem/statem_clnt.c:1112:
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 0 bytes and written 7 bytes
Verification: OK
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

## 3. Solution
You can try the following methods to resolve this exception.

## 3.1 Specify https.protocols using system property
In this article[^2], you can specify `https.protocols` in your JAVA application.
```java
-Dhttps.protocols=TLSv1.2,TLSv1.3
``` 
`Note:` Pay attention to your Java JDK version, as different versions may use different default TLS versions. My versions are Oracle JDK `11.0.23` and `17.0.11`.

## 3.2 Specify https.protocols in your JAVA code
The same idea like 3.1, but in this article[^3], someone mentioned it didn't work for them. I also tried their suggested solution, which is as follows:
```java
try {
        SSLContext ctx = SSLContext.getInstance("TLSv1.3");
        ctx.init(null, null, null);
        SSLContext.setDefault(ctx);
} catch (Exception e) {
        System.out.println(e.getMessage());
}
```
However, it doesn't work for me.

## 3.3 Workaround solution
I replaced the default `protocol-http` plugin with the `protocol-okhttp` plugin, and it worked for me.

**Reference links:**
<br>
[chatgpt](https://chatgpt.com/)
<br>
[https://stackoverflow.com/questions/21245796/javax-net-ssl-sslhandshakeexception-remote-host-closed-connection-during-handsh/34891294](https://stackoverflow.com/questions/21245796/javax-net-ssl-sslhandshakeexception-remote-host-closed-connection-during-handsh/34891294)
<br>
[https://stackoverflow.com/questions/39157422/how-to-enable-tls-1-2-in-java-7](https://stackoverflow.com/questions/39157422/how-to-enable-tls-1-2-in-java-7)

[^1]:https://chatgpt.com/
[^2]:https://stackoverflow.com/questions/21245796/javax-net-ssl-sslhandshakeexception-remote-host-closed-connection-during-handsh/34891294
[^3]:https://stackoverflow.com/questions/39157422/how-to-enable-tls-1-2-in-java-7
