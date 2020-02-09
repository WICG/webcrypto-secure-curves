# Curve25519 in WebCrypto

## Problem Statement

Today web developers are getting around the unavailability of
[Curve25519][rfc7748] in browser by either including an implementation of its
operations in JavaScript or compiling a native one into WebAssembly. Aside from
wasting bandwidth shipping algorithms that are already included in browsers that
support TLS 1.3, this practice also has security implications, e.g. side-channel
attacks as studied by [Daniel Genkin et al][key-extraction].

## Support Curve25519 in the Web Cryptography API

We solve the above problem by adding support for Curve25519 algorithms in the
Web Cryptography API, namely the signature algorithm Ed25519 and the key
agreement algorithm X25519.

### Ed25519

The following operations are supported with the recognized algorithm name
"Ed25519":

1. generateKey
2. sign
3. verify
4. importKey
5. exportKey

### X25519

The following operations are supported with the recognized algorithm name
"X25519":

1. generateKey
2. deriveKey
3. deriveBits
4. importKey
5. exportKey

For key serialization and deserialization, the supported formats include the raw
format for X25519 public keys as an array of raw bytes, as well as the SPKI, the
PKCS#8, and the JWK formats for the public or/and the private X25519 or Ed25519
keys.

## Alternatives Considered and Security Implications

Shipping cryptographic algorithms in JavaScript or WebAssembly, including
Curve25519, has been alternative solutions to using equivalents provided by the
browser. In addition to the vulnerability to side-channel attacks as noted in
the problem statement, these alternatives are also susceptible to attacks like
cross-site scripting as key materials are typically represented in plaintext in
the portable code. A native implementation of cryptographic algorithms by the
browser provides a safe and standardized solution against known side-channel
attacks, with the availability of non-extractable key handles, namely the
CryptoKey object in WebCrypto. Applications should still note that as a
general security consideration in WebCrypto, a persistent oracle can be allowed
if CryptoKey objects are shared between origins such as through the use of
postMessage. 

## Code Examples

### Key generation


```js
// Use a string of the recognized algorithm name.
const ed25519_key = await window.crypto.subtle.generateKey(
  'Ed25519', true /* extractable */, ['sign', 'verify']);
// Or use a dictionary with only the name property.
const x25519_key = await window.crypto.subtle.generateKey(
  {name: 'X25519'}, true /* extractable */, ['deriveKey', 'deriveBits']);
```

### Digital signature with Ed25519

```js
// |data| is to be signed.
// The digital signature parameter has only the name property:
//   name, a string that should be set to 'Ed25519'.
const alg = {name: 'Ed25519'};
window.crypto.subtle.sign(alg, ed25519_key.privateKey, data).then(signature =>
  window.crypto.subtle.verify(alg, ed25519_key.publicKey, signature, data))
```

### Key agreement with X25519
```js
const private_x25519_key = x25519_key.privateKey;
// Received the peer public key |peer_public_x25519_key|.
//
// The key derivation parameters:
//   name, a string that should be set to 'X25519'.
//   public, a CryptoKey object representing the public key of the peer. 
const key_derive_params = {name: 'X25519', public: peer_public_x25519_key};
const result = window.crypto.subtle.deriveBits(
  key_derive_params, private_x25519_key, 256 /* number of bits to derive */);
```

### Export and import of keys

```js
// An example using the raw format for X25519 public keys.
const raw_public_key =
  await window.crypto.subtle.exportKey('raw', x25519_key.publicKey);
// The key import parameter has only the name property:
//   name, a string that should be set to 'Ed25519' or 'X25519'
const key_import_param = {name: 'X25519'};
const result = window.subtle.importKey('raw', raw_public_key, key_import_param,
  true /* extractable */, [deriveKey', 'deriveBits]);

// An example using the SPKI format. An example for the PKCS#8 format would be
// similar.
const spki_public_key =
  await window.crypto.subtle.exportKey('spki', x25519_key.publicKey);
const key_import_param = {name: 'X25519'};
const result = window.subtle.importKey('spki', spki_public_key, key_import_param,
  true /* extractable */, [deriveKey', 'deriveBits]);

// An example using the JWK format.
const jwk_private_key =
  await window.crypto.subtle.exportKey('jwk', ed25519_key.privateKey);
const key_import_param = {name: 'Ed25519'};
const result = window.subtle.importKey(
  'jwk', jwk_private_key, key_import_param, true /* extractable*/, ['sign']);
```

## Reference

1. [RFC 7748, Elliptic Curves for Security][rfc7748]
2. [Daniel Genkin et al, Drive-By Key-Extraction Cache Attacks from Portable
Code][key-extraction].


[rfc7748]: https://tools.ietf.org/html/rfc7748
[key-extraction]: https://www.cs.tau.ac.il/~tromer/drivebycache/
