# react-native-ssl-pinning-proposal

## Background
[SSL pinning](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning) is a technology for client applications to ensure security for server transmission. Instead of original [SSL CA validation process](https://www.globalsign.com/en/ssl-information-center/what-are-certification-authorities-trust-hierarchies/), SSL pinned client only accepts server certificate which provide by the author.

i.e. Originally, a client device can redirect to fake https://www.facebook.com/ if that client installed a malicious Root Cerficate as well as its DNS record was polluted. By SSL pinning, client application can prevent this due to the Root Cerficate is different to pinned certificate.

For me, there are two advantages of SSL pinning:

1. Prevent [MITM attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) just as mentioned above.
2. Protect your API. Originally, it is quite easy to know how the app call its API server by the technology such as [mitmproxy](https://mitmproxy.org/). It will increase the difficulty by SSL pinning. Please note that is not 100% protection. Attackers can still inspect the traffic on rooted or jailbreaked devices.

## Prior art
For some iOS/Android network libraries, SSL pinning are already implemented.

- [okhttp](https://github.com/square/okhttp/wiki/HTTPS#certificate-pinning)
- [AFNetworking](https://github.com/AFNetworking/AFNetworking/blob/master/AFNetworking/AFSecurityPolicy.h#L135-L154)

For React Native, the Android version uses okhttp, so we can adopt it easily. For iOS, we should do it handy.

## What to be pinned

There are many variations about what information to be pinned, from the whole certificate, public key and SubjectPublicKeyInfo(SPKI).

There are also some variations to pinned format, from binary file, base64 encoded, or hashed string.

For me, I prefer to pin the SPKI and in sha1 hashed format, as aligned with okhttp and Google Chrome.

## Where to store the pinned information

Originally for native mobile app development, we used to setup pinned information in source code as the okhttp sample

```java
   client = new OkHttpClient.Builder()
        .certificatePinner(new CertificatePinner.Builder()
            .add("publicobject.com", "sha1/DmxUShsZuNiqPQsX2Oi9uv2sCnw=")
            .add("publicobject.com", "sha1/SXxoaOSEzPC6BgGmxAt/EAcsajw=")
            .add("publicobject.com", "sha1/blhOM3W9V/bVQhsWAcLYwPU6n24=")
            .add("publicobject.com", "sha1/T5x9IXmcrQ7YuQxXnxoCmeeQ84c=")
            .build())
        .build();
```

However, for React Native, I prefer to store the pinned information in JavaScript. The advantage is for developr can update the pinned information easier.
However, storing pinned information in JavaScript will add vulnerablity that attackers can still give you malicious RN bundle file. That is why I am thinking to introduce digital signature for bundle file.

## Action items

There are two parts of action items:

### Bundle file Digital Signature
1. Modify [signedsource](https://github.com/facebook/react-native/blob/master/local-cli/bundle/signedsource.js) in RN packager that accepted a given RSA private key file. (Currently the signedsource only do MD5 checksum to do integrity check).
2. Modify `react-native bundle` to accept given RSA pem file to do sign.
3. [iOS] Add new interface to `RCTBridge` that accept RSA public key, then `initWithBundleURL:...` in `RCTRootView` can accept a public key for bundle authorize verification.
4. [iOS] Port [signedsource.verify_signature()](https://github.com/facebook/react-native/blob/master/local-cli/bundle/signedsource.js#L79-L95) logic but allow verify by given public key to Objective C.
5. [Android] Add new interface to `ReactInstanceManager` that accept RSA public for bundle authorize verification.
6. [Android] Port [signedsource.verify_signature()](https://github.com/facebook/react-native/blob/master/local-cli/bundle/signedsource.js#L79-L95) logic but allow verify by given public key to Java.

### SSL Pinning

1. Add new interface to `fetch` and `XMLHttpRequest` that accepts `{domain: SPKI_sha1_hashed_string}` pair for pinning.
2. [iOS] Modify `RCTNetworking` and foundmental `NSURLSession` to accept SPKI sha1 hashed string and do verification when connecting to target domain.
3. [Android] Modify `RCTNetworking` and pass given pinning information to okhttp
