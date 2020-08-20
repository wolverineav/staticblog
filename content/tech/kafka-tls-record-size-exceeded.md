---
title: "Kafka TLS Record Size Exceeded"
date: 2020-08-18T16:20:00-07:00
slug: ""
description: "Kafka client unable to connect, potentially due to TLS record size limit exceeded. How to resolve using Kafka's extensible SSL context/engine - KIP-519."
keywords: []
draft: false
tags: ["tech", "scale", "kafka", "ssl", "tls"]
math: false
toc: true
---

## Pre-requisites

This post assumes you know what is SSL, TLS, keystore, truststore and  how they
all mingle together in the context of a specific service.

The problem described is in the context of Kafka, but may happen with other web
services using TLS as well. The fix may be similar, but you would have to make
the trade-off of whether it is acceptable for your service clients.

## Problem

### Kafka clients are unable to connect to the broker

We observed a strange behaviour in our scale test - after a sync of all the
client _self-signed_ certificates to the broker truststore, none of the clients
were able to connect.

Kafka's `server.log` would show that is awaiting socket connections as the last
log line:

```bash
2020-07-02T11:55:03.180Z{UTC}  INFO main Acceptor - Awaiting socket connections on 0.0.0.0:9092.
```

If you were to take a thread dump of the Kafka process thread, you would see it
stuck at the `validate` method in `SslFactory`:

```bash
Jul 02 16:13:11 broker kafka[10663]: "main" #1 prio=5 os_prio=0 tid=0x000078f6e000f000 nid=0x2d24 runnable [0x000078f6e6e8e000]
Jul 02 16:13:11 broker kafka[10663]:    java.lang.Thread.State: RUNNABLE
Jul 02 16:13:11 broker kafka[10663]:         at org.apache.kafka.common.security.ssl.SslFactory$SslEngineValidator.validate(SslFactory.java:299)
Jul 02 16:13:11 broker kafka[10663]:         at org.apache.kafka.common.security.ssl.SslFactory$SslEngineValidator.validate(SslFactory.java:280)
Jul 02 16:13:11 broker kafka[10663]:         at org.apache.kafka.common.security.ssl.SslFactory.configure(SslFactory.java:98)
Jul 02 16:13:11 broker kafka[10663]:         at org.apache.kafka.common.network.SslChannelBuilder.configure(SslChannelBuilder.java:69)
Jul 02 16:13:11 broker kafka[10663]:         at org.apache.kafka.common.network.ChannelBuilders.create(ChannelBuilders.java:146)
...
Jul 02 16:13:11 broker kafka[10663]:         at kafka.network.SocketServer.createDataPlaneAcceptorsAndProcessors(SocketServer.scala:238)
...
Jul 02 16:13:11 broker kafka[10663]:         at kafka.server.KafkaServer.startup(KafkaServer.scala:263)
Jul 02 16:13:11 broker kafka[10663]:         at kafka.server.KafkaServerStartable.startup(KafkaServerStartable.scala:44)
Jul 02 16:13:11 broker kafka[10663]:         at kafka.Kafka$.main(Kafka.scala:84)
Jul 02 16:13:11 broker kafka[10663]:         at kafka.Kafka.main(Kafka.scala)
```

This led us to the next part - of understanding the handshake process of SSL/TLS.

### Kafka and SSL

Kafka has <cite>a couple of ways of managing truststore [^1]</cite> when SSL is
enabled.

1. The truststore contains one or many certificates: the broker or logical
client will trust _any certificate listed in the truststore_.
2. The truststore contains a Certificate Authority (CA): the broker or logical
client will trust _any certificate that was signed by the CA in the truststore_.
Described in a diagram <cite>here.[^2]</cite>

With the approach described in point 2, you are most likely to have a handful
of certificates in the truststore.

On the off chance that all your clients use self-signed certificate and you go
with approach 1; to import all the certificates into the broker truststore, you
would need to keep track of the total size of all the certificates combined.
Let me will elaborate *why* in the next section.

### SSL/TLS Handshake

The TCP layer equivalent of SSL is TLS - which handles the handshake.

![TLS handshake protocol](/img/kafka-tls-record-size-exceeded/tls-handshake.svg)*TLS handshake protocol. More details <cite>here: [^3]</cite>*

Of special note is the `Certificate Request` step described in detail in the
<cite>TLS RFC-5246 [^4]</cite> - it is an optional step, to
send the list of known CAs to the client. This is to help client identify which
certificate to send during the handshake; in case it has multiple certs signed
by different certificate authorities.

#### TLS Record Size - 16KB

This is where the TLS record size comes into picture. For the client to
correctly decode the message from server, it has to be received in a single
record. If it is split into multiple parts, client will not be able to decode
the remaining half.

In case of Kafka, it would not even reach that stage, the server (broker) side
itself would get stuck at the sending of `certificate_authorities` step.

#### Optional Protocol Step

Key thing to notice here is that the sending of `certificate_authorities` is an
optional step. Depending on your use case, you may be fine with skipping that
step.

In my case, we knew all the client were on the internal subnet, they all used
self-signed certificate and they would always have _one and only one_
certificate as its identity. There will not be ambiguity at the client side
about which certificate to send based on the known CA certificates list.

So we could unilaterally decide to skip the optional step; YMMV. But how do we
do that exactly?

## Solution

### KIP-519 Make SSL context/engine configuration extensible

Luckily, the developers of Kafka were aware of this requirement based on growing
use cases in matured enterprise environments. They already had <cite>a feature in
the works [^5]</cite>, which was not released yet, but was available if you
built a Kafka distribution from git.

```java
diff --git a/clients/src/main/java/org/apache/kafka/common/security/ssl/DefaultSslEngineFactory.java b/clients/src/main/java/org/apache/kafka/common/security/ssl/DefaultSslEn
gineFactory.java
index f71adaf62..82ce12811 100644
--- a/clients/src/main/java/org/apache/kafka/common/security/ssl/DefaultSslEngineFactory.java
+++ b/clients/src/main/java/org/apache/kafka/common/security/ssl/DefaultSslEngineFactory.java
@@ -32,6 +32,7 @@ import javax.net.ssl.KeyManagerFactory;
 import javax.net.ssl.SSLContext;
 import javax.net.ssl.SSLEngine;
 import javax.net.ssl.SSLParameters;
+import javax.net.ssl.TrustManager;
 import javax.net.ssl.TrustManagerFactory;
 import java.io.IOException;
 import java.io.InputStream;
@@ -233,11 +234,10 @@ public final class DefaultSslEngineFactory implements SslEngineFactory {
             }
             String tmfAlgorithm = this.tmfAlgorithm != null ? this.tmfAlgorithm : TrustManagerFactory.getDefaultAlgorithm();
-            TrustManagerFactory tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
             KeyStore ts = truststore == null ? null : truststore.get();
-            tmf.init(ts);
+            TrustManager[] myTMs = new MyX509TrustManager[] { new MyX509TrustManager(tmfAlgorithm, ts) };
-            sslContext.init(keyManagers, tmf.getTrustManagers(), this.secureRandomImplementation);
+            sslContext.init(keyManagers, myTMs, this.secureRandomImplementation);
             log.debug("Created SSL context with keystore {}, truststore {}, provider {}.",
                     keystore, truststore, sslContext.getProvider().getName());
             return sslContext;
```

The goal was to override the default TrustManager with our custom TrustManager.
This is well documented and explained in the Java Secure Socket Extension
(JSSE) <cite>reference guide. [^6]</cite>

### My509TrustManager

```java
public class MyX509TrustManager implements X509TrustManager {
    /*
     * The default X509TrustManager.  Decisions are delegated
     * to it, and a fall back to the logic in this class is performed
     * if the default X509TrustManager does not trust it.
     */
    X509TrustManager baseTrustManager;
    private static final X509Certificate[] emptyAcceptedIssuers = {};

    /**
     * Merely a pass-through
     */
    @Override
    public void checkClientTrusted(X509Certificate[] x509Certificates, String authType) throws CertificateException {
        baseTrustManager.checkClientTrusted(x509Certificates, authType);
    }

    /**
     * Merely a pass-through
     */
    @Override
    public void checkServerTrusted(X509Certificate[] x509Certificates, String authType) throws CertificateException {
        baseTrustManager.checkServerTrusted(x509Certificates, authType);
    }

    /**
     * Since the list of Accepted Issuers consists of all CAs - in our case, all
     * certs being self-signed, this is a list of all Transport Node and Bare
     * Metal self-signed certs.
     *
     * This list may go over 16KB - which is approx 300 certs as per scale setup
     * testing.
     *
     * Since this is optional, as per TLS RFC spec [1], we will return an empty
     * list always.
     *
     * Considerations:
     *  - if the client has multiple certificates - each signed by a different
     *    CA, this list helps identify which CA signed cert to use
     *  - but since we use self-signed certs, this is not an issue
     *  - in future, if clients install 'multiple' (not just one) CA signed
     *    cert, then this would be a concern
     *  - in that case, we should remove this customization and allow the default
     *    code to play
     *  - in that case, install _only_ the CA certs in kafka_broker_truststore
     *
     * [1] https://tools.ietf.org/html/rfc5246#page-53
     */
    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return emptyAcceptedIssuers;
    }
}
```

That's it!

Build Kafka, deploy it as the broker and watch clients connect without the
broker getting stuck during SSL/TLS handshake.

## Thanks

The whole search for root cause was a slow-moving train that gained speed with
help from some clever folks. [Ming Wen](https://bitmingw.com) helped identify
where Kafka was getting stuck and
[Mark Boon](http://linkedin.com/in/mark-boon-738abb8), our in house X509
certificate expert suggested how to approach the problem with a custom
TrustManager.

Being in the platform engineering team, I'm usually the one connecting the
dots/teams/modules and figuring out where is the ideal place to fix it.

**TIL**

[^1]: https://docs.confluent.io/current/kafka/authentication_ssl.html
[^2]: https://github.com/confluentinc/confluent-platform-security-tools/blob/master/single-trust-store-diagram.pdf
[^3]: https://hpbn.co/transport-layer-security-tls/
[^4]: https://tools.ietf.org/html/rfc5246#page-53
[^5]: https://github.com/apache/kafka/pull/8338
[^6]: https://docs.oracle.com/en/java/javase/11/security/java-secure-socket-extension-jsse-reference-guide.html#GUID-E1205974-3249-4E40-83C0-5F89C7375CF4
