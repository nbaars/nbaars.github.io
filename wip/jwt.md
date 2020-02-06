---
layout: post
title: Brute force JWT tokens
categories: [Java, Security]
---

By default I'm not a big fan of JWT tokens let's look at a couple of examples of why they should be used with
much more care then we currently do in our applications. Let's look at some attacks and see how we can mitigate the 
possible attack in our simple demo application.

## Introduction 

The JWT specification consists of couple of RFC:

- [JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
- [JSON Web Signature (JWS)](https://tools.ietf.org/html/rfc7515)
- [JSON Web Encryption (JWE]((https://tools.ietf.org/html/rfc7516))
- [JSON Web Algorithms (JWA)](https://tools.ietf.org/html/rfc7518)

All of the specifications consists of a 'Security considerations' sections with recommendations on what to look out 
for while using JWTs. There is kind of a vicious circle going on: the specification writers point at the library writers 
but the library writers at there end point to developers because the library writers implemented the specification
by the letter. This leaves all the responsibility at the developers end and we should know all the little details
about the specification. I have read the specification and created lessons in WebGoat for this and held several 
presentations about it at my company and while I enjoy reading such specification I can imagine this is not 
every ones cup of tea. 

In this article I will just use a default Java library for JWT tokens, it is not to single out one library it is 
just to show some of the attacks work.

Just a small refresher a JWT looks like `base_64(header).base_64(payload).base_64(signature)`, an example:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

If we decode the first part:

```
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Second part:
```
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

The last part contains the signature over the complete message in order to validate the token is not
tampered with.

## 1. Usage of brute force

In [RFC 7518](https://tools.ietf.org/html/rfc7518#section-3.2) it states:

>   A key of the same size as the hash output (for instance, 256 bits for
    "HS256") or larger MUST be used with this algorithm.  (This
    requirement is based on Section 5.3.4 (Security Effect of the HMAC
    Key) of NIST SP 800-117 [NIST.800-107], which states that the
    effective security strength is the minimum of the security strength
    of the key and two times the size of the internal hash value.)

If we look at [RFC 2104: HMAC: Keyed-Hashing for Message Authentication](https://tools.ietf.org/html/rfc2104) the 
key length is defined as follows:

>   The key for HMAC can be of any length (keys longer than B bytes are
    first hashed using H).  However, less than L bytes is strongly
    discouraged as it would decrease the security strength of the
    function.  Keys longer than L bytes are acceptable but the extra
    length would not significantly increase the function strength. (A
    longer key may be advisable if the randomness of the key is
    considered weak.)

>   Keys need to be chosen at random (or using a cryptographically strong
    pseudo-random generator seeded with a random seed), and periodically
    refreshed.  (Current attacks do not indicate a specific recommended
    frequency for key changes as these attacks are practically
    infeasible.  However, periodic key refreshment is a fundamental
    security practice that helps against potential weaknesses of the
    function and keys, and limits the damage of an exposed key.)

```
B := byte length of block the hash function operates on (For SHA-256 this is 512 bits or SHA-384 and SHA-512 this is 1024 bits.
L := byte length of the output produced by the hash function (For SHA-256, this is 256 bits, for SHA-384 it is 384 bits, and for SHA-512 thi is 512 bits.
```

Now that we have the formal definitions explained let's take a look at what happens when we generator a token
and use HMAC-256 to sign it.


```kotlin
fun main() {
    val token = JWT.create().withClaim("user", "John Doe").sign(Algorithm.HMAC256("test"))
    println(token)
}
```

So we get a token signed with a **not so random** key which one obtained even by you. Let's automate the process for 
brute forcing the token. There are several tools to do this we will use Jack the Ripper for the job. For this experiment
we can use the following Docker file:

```
FROM 
```


## 2. Check protocol

The JWT specification allows the creator of the token to specify the type `alg` in the header. In the 
first implementations of the JWT libraries it was allowed to change the `alg` to `none` and sends it 
to the other side. The other side reads the token and the library validated according to the given `alg` using `none` as indicating that no signature checking needed to be performed. However this leaves the door open for someone to 
adjust the token before sending it. 

In [RFC 7519:  JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519) unsecured 
JWTs are introduced: 

>    To support use cases in which the JWT content is secured by a means
     other than a signature and/or encryption contained within the JWT
     (such as a signature on a data structure containing the JWT), JWTs
     MAY also be created without a signature or encryption.  An Unsecured
     JWT is a JWS using the "alg" Header Parameter value "none" and with
     the empty string for its JWS Signature value, as defined in the JWA
     specification [JWA]; it is an Unsecured JWS with the JWT Claims Set
     as its JWS Payload.

The JWT will look like: `eyJhbGciOiJub25lIn0.eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ.`

JWT libraries quickly updated once this vulnerability was found, most of them refuse to parse the claims when a key has been set.

### Example attack




