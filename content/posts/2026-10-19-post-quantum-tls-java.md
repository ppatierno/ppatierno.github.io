---
title: "Post-Quantum TLS made easy in Java: Hybrid Key Exchange in action"
date: 2026-10-19
description: "Protecting TLS traffic from 'harvest now, decrypt later' attacks usually means new libraries and code changes. In Java 27 it is a JDK upgrade. This post shows how to verify the hybrid key exchange with a plain HTTP client and server, and how to break it on purpose."
image: "images/posts/post_quantum_tls_java.png"
---

Post-Quantum Cryptography is moving from research papers into the platforms we use every day.
With JEP-527, Java 27 brings Post-Quantum Hybrid Key Exchange (PQHKE) to TLS 1.3.
The surprising part is how little you have to do to benefit from it.
This article shows what changes in the JDK and how to verify it with nothing but a plain HTTP client and server.


## The recording in the drawer: Post-Quantum TLS in plain Java

Imagine someone sitting on a network tap, quietly recording your encrypted traffic.
They can't read any of it.
The ciphers are fine, the certificates are valid, and nothing is wrong.
So they just keep the recordings and wait.

That's the "Harvest Now, Decrypt Later" (HNDL) scenario, and it is why Post-Quantum Cryptography (PQC) is urgent even though no quantum computer can break today's cryptography yet.
The worry is that an adversary could store encrypted data today and decrypt it once such computers exist.
If your data still matters in ten or fifteen years, the clock has already started.

In this post I want to follow that story into the Java JDK.
We'll look at what is actually vulnerable, what the fix looks like, and how [**JEP-527**](https://openjdk.org/jeps/527) in Java 27 brings the fix to `javax.net.ssl`.
Then we'll build a tiny HTTPS server and client with nothing but the JDK, watch the handshake, and break it on purpose to prove it works.

## What is actually at risk

TLS does two jobs.
First, the two sides agree on a shared secret.
Then they use that secret to encrypt the traffic with a fast symmetric cipher like AES.

The symmetric part is in decent shape against quantum computers.
The first part, the key exchange, is the weak link.
Classical key exchange (ECDHE, which includes X25519) relies on math problems that [Shor's algorithm](https://en.wikipedia.org/wiki/Shor%27s_algorithm) would solve efficiently on a large enough quantum computer.
Whoever recorded the handshake could then recompute the shared secret and decrypt the whole session.

So the part to replace is the handshake, not the cipher.

## The replacement: a KEM, and why we don't trust it alone

The new building block is the **KEM** (Key Encapsulation Mechanism).
One side publishes a public key.
The other side uses it to "encapsulate" a fresh random secret and sends back a ciphertext.
Only the holder of the private key can recover that secret.
The standardized post-quantum KEM is [**ML-KEM**](https://csrc.nist.gov/pubs/fips/203/final), which is based on lattice problems believed to resist quantum attacks.

ML-KEM is much younger than elliptic curves, which have decades of scrutiny behind them.
A flaw could still be found.
So the IETF approach for TLS 1.3 is **hybrid** key exchange.
Run a classical exchange and a post-quantum one side by side, and mix both results into the session keys.
The result stays secure as long as one of the two algorithms remains unbroken.
You lose nothing against classical attackers, and you gain protection against quantum ones.

## How Java got here

Java didn't do this in one jump. It assembled the parts over several releases:

- Java 21 added the KEM API via [JEP-452](https://openjdk.org/jeps/452).
- Java 24 added ML-KEM via [JEP-496](https://openjdk.org/jeps/496).
- Java 27 adds hybrid key exchange for TLS 1.3 via [JEP-527](https://openjdk.org/jeps/527).

TLS calls its key exchange schemes **named groups**.
JEP-527 adds three hybrid ones to SunJSSE:

- `X25519MLKEM768`
- `SecP256r1MLKEM768`
- `SecP384r1MLKEM1024`

The name tells you the recipe.
`X25519MLKEM768` runs X25519 and ML-KEM-768 together.
The Java team also explains the design and the new named groups in the [Post-Quantum Hybrid Key Exchange for TLS 1.3](https://inside.java/2026/02/17/tls-post-quantum-hybrid-key-exchange/) article on Inside.java.

The part I like best is what the JEP does by default.
`X25519MLKEM768` is placed first in the client's preference list, ahead of x25519, secp256r1 and the rest, so code that doesn't pin named groups benefits automatically.
The goal is that you upgrade the JDK and your TLS is quantum-resistant, without touching your code.
Let's check that claim.

## Setting the stage

All we need is JDK 27.
Both programs are ordinary single-class `.java` files that the JDK can run directly with `java FileName.java`.

First, a key pair for the server and a truststore for the client:

```shell
keytool -genkeypair -alias server -keyalg EC -groupname secp256r1 \
  -dname "CN=localhost" -ext san=dns:localhost -validity 365 \
  -keystore server.p12 -storetype PKCS12 -storepass changeit

keytool -exportcert -alias server -keystore server.p12 \
  -storepass changeit -file server.crt

keytool -importcert -alias server -file server.crt -noprompt \
  -keystore client-truststore.p12 -storetype PKCS12 -storepass changeit
```

Now the server, using the HTTP server built into the JDK.
Notice that it doesn't mention post-quantum anything.

```java
import com.sun.net.httpserver.HttpsConfigurator;
import com.sun.net.httpserver.HttpsExchange;
import com.sun.net.httpserver.HttpsServer;

import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSession;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyStore;

public class PqcServer {
    public static void main(String[] args) throws Exception {
        char[] pwd = "changeit".toCharArray();

        KeyStore ks = KeyStore.getInstance("PKCS12");
        try (InputStream in = Files.newInputStream(Path.of("server.p12"))) {
            ks.load(in, pwd);
        }
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        kmf.init(ks, pwd);

        SSLContext ctx = SSLContext.getInstance("TLS");
        ctx.init(kmf.getKeyManagers(), null, null);

        HttpsServer server = HttpsServer.create(new InetSocketAddress(8443), 0);
        server.setHttpsConfigurator(new HttpsConfigurator(ctx));

        server.createContext("/hello", exchange -> {
            SSLSession session = ((HttpsExchange) exchange).getSSLSession();
            byte[] body = ("Hello over " + session.getProtocol()
                    + " / " + session.getCipherSuite() + "\n")
                    .getBytes(StandardCharsets.UTF_8);
            exchange.sendResponseHeaders(200, body.length);
            try (OutputStream out = exchange.getResponseBody()) {
                out.write(body);
            }
        });

        server.start();
        System.out.println("Listening on https://localhost:8443/hello");
    }
}
```

And the client, using `java.net.http.HttpClient`:

```java
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManagerFactory;
import java.io.InputStream;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyStore;

public class PqcClient {
    public static void main(String[] args) throws Exception {
        char[] pwd = "changeit".toCharArray();

        KeyStore ts = KeyStore.getInstance("PKCS12");
        try (InputStream in = Files.newInputStream(Path.of("client-truststore.p12"))) {
            ts.load(in, pwd);
        }
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(ts);

        SSLContext ctx = SSLContext.getInstance("TLS");
        ctx.init(null, tmf.getTrustManagers(), null);

        HttpClient client = HttpClient.newBuilder()
                .sslContext(ctx)
                .version(HttpClient.Version.HTTP_1_1)
                .build();

        HttpRequest request = HttpRequest.newBuilder(URI.create("https://localhost:8443/hello")).build();
        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        System.out.println(response.statusCode() + " " + response.body().strip());
        response.sslSession().ifPresent(s ->
                System.out.println("Client side: " + s.getProtocol() + " / " + s.getCipherSuite()));
    }
}
```

Neither program has any code about named groups and that is deliberate.

## Chapter one: just run it

```shell
# terminal 1
java PqcServer.java
# terminal 2
java PqcClient.java
```

On the client side, you get a `200`, TLS 1.3, and an AES-GCM cipher suite.

```shell
200 Hello over TLSv1.3 / TLS_AES_256_GCM_SHA384
Client side: TLSv1.3 / TLS_AES_256_GCM_SHA384
```

That output is the first thing to be skeptical about—it looks identical whether the key exchange was classical or post-quantum.
The cipher suite name only describes the symmetric part.
The key exchange group lives in the handshake, and the `SSLSession` API does not expose which named group was chosen.

To see it, we have to look at the handshake itself.

## Chapter two: watching the handshake

The JDK can log the TLS handshake.
Because `javax.net.debug` writes to stderr, redirect both streams:

```shell
java -Djavax.net.debug=ssl:handshake PqcServer.java > server.log 2>&1
java -Djavax.net.debug=ssl:handshake PqcClient.java > client.log 2>&1
```

The output is verbose.
The two places to look in `client.log` are the **ClientHello** and the **ServerHello**.

In the **ClientHello**, the `supported_groups` extension lists every named group the client is willing to use. `X25519MLKEM768` appears first—ahead of every classical group—which is exactly the JEP-527 default preference order:

```
"supported_groups (10)": {
  "named groups": [X25519MLKEM768, x25519, secp256r1, secp384r1, secp521r1, ...]
}
```

The `key_share` extension contains the actual public key material the client sent speculatively.
There are **two** shares: one for `X25519MLKEM768` and a second, smaller one for `x25519`.
The client sends both as a bet—it prefers the hybrid group, but includes the classical share so the handshake can complete in one round trip even if the server does not know the hybrid group:

```
"key_share (51)": {
  "client_shares": [
    {
      "named group": X25519MLKEM768
      "key_exchange": {
        0000: 91 97 62 D6 56 86 21 5B  ...   ← ~1.2 KB ML-KEM + X25519 public key
        ...
      }
    },
    {
      "named group": x25519
      "key_exchange": {
        0000: E1 73 68 DF B0 07 5A D2  ...   ← 32 bytes X25519 public key
      }
    }
  ]
}
```

Notice the size difference: the `X25519MLKEM768` share occupies over 1 KB (the ML-KEM-768 public key alone is 1,184 bytes), while the `x25519` share is just 32 bytes.
This is why the ClientHello grows noticeably compared to a classical-only handshake.

In the **ServerHello**, the `key_share` extension shows which group the server actually chose.
Here it confirms `X25519MLKEM768`:

```
"ServerHello": {
  "cipher suite": "TLS_AES_256_GCM_SHA384(0x1302)",
  ...
  "key_share (51)": {
    "server_share": {
      "named group": X25519MLKEM768
      "key_exchange": { ... }   ← ML-KEM ciphertext encapsulating the shared secret
    }
  }
}
```

The server discarded the `x25519` fallback share because it understood the hybrid group.
The session key material was derived from both the X25519 and the ML-KEM exchanges combined.

As a control experiment, run the same two programs on JDK 24 or earlier.
The code is identical, but `supported_groups` will not include `X25519MLKEM768` and the ServerHello will select plain `x25519`.
The JDK upgrade alone changes the outcome.

## Chapter three: breaking it on purpose

Logs show what was offered.
We want stronger evidence that the hybrid exchange really carried the session.
We can get it by removing every alternative.

We use the `jdk.tls.namedGroups` system property to control which groups each side will use.
First, a server that only knows classical groups, and a client that will only speak hybrid:

```shell
# server: classical only
java -Djdk.tls.namedGroups=x25519,secp256r1 PqcServer.java > server.log 2>&1
# client: hybrid only
java -Djdk.tls.namedGroups=X25519MLKEM768 PqcClient.java > client.log 2>&1
```

This should fail.
The two sides have no group in common, so TLS 1.3 cannot complete the handshake.
And it does:

```shell
Exception in thread "main" javax.net.ssl.SSLException: (unexpected_message) Received fatal alert: unexpected_message
	at java.net.http/jdk.internal.net.http.HttpClientImpl.send(HttpClientImpl.java:932)
Caused by: javax.net.ssl.SSLProtocolException: (unexpected_message) Received fatal alert: unexpected_message
	at java.base/sun.security.ssl.Alert.createSSLException(Alert.java:128)
	...
```

The server sent a fatal `unexpected_message` alert which is how this handshake failure surfaces.
That failure is the proof: the client genuinely refused to fall back to a classical exchange.

Now let the server learn the hybrid group:

```shell
java -Djdk.tls.namedGroups=X25519MLKEM768,x25519 PqcServer.java > server.log 2>&1
java -Djdk.tls.namedGroups=X25519MLKEM768 PqcClient.java > client.log 2>&1
```

The same client now succeeds.
It offered only `X25519MLKEM768`, so that must be the group that produced the session keys.
No log reading is needed for this proof.

If you want per-connection control in code, `SSLParameters::setNamedGroups` is the right hook.
With `HttpClient` you pass it through the builder:

```java
SSLParameters params = new SSLParameters();
// two hybrid schemes first, then two traditional fallbacks
params.setNamedGroups(new String[] {
    "X25519MLKEM768", "SecP256r1MLKEM768", "x25519", "secp256r1"
});

HttpClient client = HttpClient.newBuilder()
        .sslContext(ctx)
        .sslParameters(params)
        .version(HttpClient.Version.HTTP_1_1)
        .build();
```

`HttpsServer` does not expose an equivalent builder method; use the `jdk.tls.namedGroups` system property when you need to constrain the server side in tests, or configure the named groups via a custom `HttpsConfigurator` subclass that overrides `configure(HttpsParameters)`:

```java
server.setHttpsConfigurator(new HttpsConfigurator(ctx) {
    @Override
    public void configure(HttpsParameters params) {
        SSLParameters sslParams = getSSLContext().getDefaultSSLParameters();
        // two hybrid schemes first, then two traditional fallbacks
        sslParams.setNamedGroups(new String[] {
            "X25519MLKEM768", "SecP256r1MLKEM768", "x25519", "secp256r1"
        });
        params.setSSLParameters(sslParams);
    }
});
```

`configure` is called by `HttpsServer` for every new connection, just before the TLS handshake begins.
`getSSLContext().getDefaultSSLParameters()` fetches the base parameters from the same `SSLContext`, so all other settings (cipher suites, protocols) are preserved and only the named groups are overridden.

I used the system property in the demo because it makes the "no code changes" point most clearly.

## What this does and does not buy you

Before anyone declares victory, here is what the demo does not do.

- **It protects confidentiality, not authentication.** Our certificate is still an elliptic curve key. JEP-527 covers the key exchange only and recorded traffic is protected against a future quantum attacker, but post-quantum signatures and certificates are separate work.
- **A hybrid handshake needs both peers.** The hybrid group is used only if both peers support it. If the server does not understand `X25519MLKEM768`, TLS falls back to the next group in the preference list—which is why the default list still includes classical groups. A server running an old JDK anywhere in your path quietly keeps you classical.
- **Mind the network path.** An `X25519MLKEM768` key share is roughly 1.2 KB, making the ClientHello noticeably larger than before. Some legacy proxies, firewalls, and TLS-inspecting middleboxes impose size limits on the ClientHello or the first flight. Test with your real infrastructure before rolling this out broadly.

## JDK availability: Java 27, 25, and beyond

This article uses Java 27, where JEP-527 first landed (September 2026).
The backport to Java 25 LTS is coming via the October 2026 Critical Patch Update (CPU), which has not shipped yet at the time of writing.
Java 24 added ML-KEM as a standalone cryptographic primitive (JEP-496), but did not wire it into TLS.
`X25519MLKEM768` as a TLS named group only exists from Java 25 (post-October 2026 CPU) or Java 27 onwards.

Oracle's [JRE and JDK Cryptographic Roadmap](https://www.java.com/en/jre-jdk-cryptoroadmap.html) plans further backports to the Java 21, 17, and 11 LTS lines, expected in the first half of 2027.
If your production baseline is Java 21 today, the same code shown here will work unchanged once the backport reaches your LTS of choice.
Until then, any peer still running a JDK without JEP-527 will silently fall back to classical key exchange, which is exactly the hybrid design working as intended.

## The takeaway

What I like about JEP-527 is how simple the migration is.
The story started with a scary threat, and the fix for most Java applications is a JDK upgrade.
Anything built on `javax.net.ssl`, including the plain `HttpClient` and `HttpsServer` above, picks up hybrid key exchange by default.
That is the Java platform at its best: a hard cryptographic migration arrives as a runtime upgrade, and years of existing code are protected without a single line changing.
Java's long-standing commitment to backward compatibility pays off here, and it is a big reason why moving to post-quantum TLS can be this easy.
Your job is to know how to verify it, how to pin it when you need to, and what it still leaves unprotected.
