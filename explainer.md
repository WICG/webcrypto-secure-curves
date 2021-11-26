# Secure Curves in WebCrypto

## Problem Statement

The Web Cryptography API currently only specifies the [NIST P-256, P-384, and
P-521 curves][nist-186-4], and does not specify any ["safe curves"][safecurves].
Among the safe curves, Curve25519 and Curve448 have gained the most traction,
and have been specified for use in TLS 1.3, for example. They have also been
[recommended][rfc7748] by the Crypto Forum Research Group (CFRG) of the Internet
Research Task Force (IRTF), and are [expected to be approved by
NIST][nist-186-5].

In addition, Node.js [has implemented][node-webcrypto] a nonstandard extension
to Web Crypto, adding Curve25519 and Curve448 under a vendor-prefixed name.
We would like to avoid other implementations doing the same, and encourage
intercompatibility going forward by providing a standard specification.

Today, web developers are getting around the unavailability of Curve25519 and
Curve448 in the browser either by using less secure curves, or by including an
implementation of Curve25519 and/or Curve448 in JavaScript or WebAssembly. In
addition to wasting bandwidth shipping algorithms that are already included in
browsers that implement TLS 1.3, this practice also has security implications,
e.g. side-channel attacks as studied by [Daniel Genkin et al][key-extraction].

## Support Curve25519 and Curve448 in the Web Cryptography API

We solve the above problem by adding support for Curve25519 and Curve448
algorithms in the Web Cryptography API, namely the signature algorithms
[Ed25519 and Ed448][rfc8032], and the key agreement algorithms
[X25519 and X448][rfc7748].

## Alternatives Considered and Security Implications

Shipping cryptographic algorithms in JavaScript or WebAssembly, including
Curve25519 or Curve448, has been an alternative solution to using
implementations provided by the browser. In addition to the vulnerability to
side-channel attacks as noted in the problem statement, these alternatives are
also potentially more susceptible to attacks like cross-site scripting, as the
private key material must be available to JavaScript. An implementation in
WebCrypto can solve this issue through the availability of non-extractable key
handles (though applications should note that as a general security
consideration in WebCrypto, a persistent oracle can be allowed if CryptoKey
objects are shared between origins such as through the use of postMessage).

Once the [constant-time proposal for WebAssembly][ct-wasm] is accepted and
implemented, it will become possible to implement Curve25519 and/or Curve448 in
WebAssembly while protecting against timing attacks. However, this still
requires the private key material to be accessible to script, potentially
exposing it to cross-site scripting and other attacks. In addition, the ready
availability of Web Crypto, compared to a WebAssembly library, may encourage
developers to continue to use the curves available in Web Crypto only. To
encourage the use of modern and secure curves like Curve25519 and Curve448, we
believe it would be good to implement them in Web Crypto.

## Supported algorithms

### Ed25519 and Ed448

The following operations are supported with the recognized algorithm names
"Ed25519" and "Ed448":

1. generateKey
2. sign
3. verify
4. importKey
5. exportKey

### X25519 and X448

The following operations are supported with the recognized algorithm names
"X25519" and "X448":

1. generateKey
2. deriveKey
3. deriveBits
4. importKey
5. exportKey

### Import and export formats

For key serialization and deserialization, the supported formats include the raw
format for public keys as an ArrayBuffer containing the raw bytes, as well as
the SPKI format for public keys, the PKCS#8 format for private keys, and the JWK
format for public and private keys.

## Code Examples

### Key generation

```js
// Pass a string of the recognized algorithm name.
const ed25519_key = await crypto.subtle.generateKey('Ed25519',
  true /* extractable */, ['sign', 'verify']);
// Or pass an object with only the name property.
const x25519_key = await crypto.subtle.generateKey({ name: 'X25519' },
  true /* extractable */, ['deriveKey', 'deriveBits']);
```

### Signing and verification using Ed25519

```js
// Some message to be signed.
const data = new TextEncoder().encode('Some message');
// Pass a string or an object with only the name property, as above.
const signature = await crypto.subtle.sign('Ed25519', ed25519_key.privateKey,
  data);
const verified = await crypto.subtle.verify('Ed25519', ed25519_key.publicKey,
  signature, data);
```

### Signing and verification using Ed448 with a context

```js
// Some message to be signed.
const data = new TextEncoder().encode('Some message');
// Some context for the message, to provide domain separation.
const context = new TextEncoder().encode('Some context');
// The context is optional, and defaults to the empty octet string.
// If the context is not needed, the string 'Ed448' can be passed as well,
// instead of an object.
const signature = await crypto.subtle.sign(
  { name: 'Ed448', context }, ed448_key.privateKey, data);
const verified = await crypto.subtle.verify(
  { name: 'Ed448', context }, ed448_key.publicKey, signature, data);
```

### Key agreement using X25519 or X448

```js
const private_x25519_key = x25519_key.privateKey;
// Assume we received the peer's public key, |peer_public_x25519_key|.
//
// The key derivation parameters:
//   name, a string that should be set to 'X25519' or 'X448'.
//   public, a CryptoKey object representing the public key of the peer.
const key_derive_params = { name: 'X25519', public: peer_public_x25519_key };
const result = await crypto.subtle.deriveBits(
  key_derive_params, private_x25519_key, 256 /* number of bits to derive */);
```

### Export and import of keys

```js
// An example using the raw format for X25519 public keys.
const raw_public_key =
  await crypto.subtle.exportKey('raw', x25519_key.publicKey);
// Pass a string or an object with only the name property,
// which should be set to 'Ed25519', 'Ed448', 'X25519' or 'X448'.
const result = await subtle.importKey('raw', raw_public_key, 'X25519',
  true /* extractable */, ['deriveKey', 'deriveBits']);

// An example using the SPKI format. An example for the PKCS#8 format would be
// similar.
const spki_public_key =
  await crypto.subtle.exportKey('spki', x25519_key.publicKey);
const result = await subtle.importKey('spki', spki_public_key, 'X25519',
  true /* extractable */, ['deriveKey', 'deriveBits']);

// An example using the JWK format.
const jwk_private_key =
  await crypto.subtle.exportKey('jwk', ed25519_key.privateKey);
const result = await subtle.importKey(
  'jwk', jwk_private_key, 'Ed25519', true /* extractable */, ['sign']);
```

## References

1. [RFC 7748, Elliptic Curves for Security][rfc7748]
2. [RFC 8032, Edwards-Curve Digital Signature Algorithm (EdDSA)][rfc8032]
3. [FIPS 186-4: Digital Signature Standard (DSS)][nist-186-4]
4. [FIPS 186-5 (Draft): Digital Signature Standard (DSS)][nist-186-5]
5. [Daniel J. Bernstein and Tanja Lange, SafeCurves: choosing safe curves for elliptic-curve cryptography][safecurves]
6. [Daniel Genkin et al, Drive-By Key-Extraction Cache Attacks from Portable Code][key-extraction].
7. [Node.js Web Crypto API][node-webcrypto]
8. [Constant-time Proposal for WebAssembly][ct-wasm]


[rfc7748]: https://tools.ietf.org/html/rfc7748
[rfc8032]: https://datatracker.ietf.org/doc/html/rfc8032
[nist-186-4]: https://csrc.nist.gov/publications/detail/fips/186/4/final
[nist-186-5]: https://csrc.nist.gov/publications/detail/fips/186/5/draft
[safecurves]: https://safecurves.cr.yp.to/
[key-extraction]: https://www.cs.tau.ac.il/~tromer/drivebycache/
[node-webcrypto]: https://nodejs.org/api/webcrypto.html
[ct-wasm]: https://github.com/WebAssembly/constant-time
