# HPURL: Hash Parameter URLs Specification  

*Version 0.1 — July 29, 2025*

## Abstract

This document specifies a convention for encoding URL query parameters within the fragment identifier (hash) portion of a URL. These are referred to as "Hash Parameter URLs" (HPURLs). This format is designed for scenarios where parameters must not be sent to the server and instead remain client-side, while preserving familiar URL query semantics.

## 1. Introduction

Traditional URLs place query parameters in the "search" portion (following the `?`), which is always sent to the server during HTTP requests. In client-heavy applications, such as Single Page Applications (SPAs), there is often a need to encode query-like parameters that are retained entirely client-side.

HPURL (Hash Parameter URL) introduces a simple convention for encoding parameters in the fragment portion of the URL using a query string-like format, such as `#?a=b`.

This document defines the structure, behavior, and parsing rules for HPURLs.

## 2. Terminology

- **HPURL**: A URL containing a hash segment with parameters in a query string format.
- **Fragment**: The portion of the URL after the `#` symbol.
- **HPURL Parameters**: Key-value pairs encoded in the HPURL.

## 3. Syntax

An HPURL follows the standard URL syntax with the following rule applied to the fragment component:

```
fragment := [prefix]?key1=value1[&key2=value2...]
```

Valid examples:

- `https://host.ext/path#?a=b`
- `https://host.ext/path?x=y#?a=b`
- `https://host.ext/path#someanchor?a=b`

In all cases above, the HPURL parameters are:

```json
{
  "a": "b"
}
```

Notes:

- The HPURL portion begins at the first `?` within the fragment.
- Everything after the `?` is parsed using the standard query parameter rules.
- Everything before the `?` in the fragment (if present) is application-defined and ignored by default by HPURL parsers.

## 4. Parsing Rules

HPURL parsers **MUST**:

- Extract the fragment component (e.g., `location.hash` in browsers).
- Locate the first occurrence of the `?` character in the fragment.
- Pass the substring beginning with `?` to a query string parser (e.g., `new URLSearchParams()`).
- Interpret the parameters exactly as standard search params are interpreted.

HPURL parsers **SHOULD** ignore the portion of the fragment before the `?`, unless application logic requires otherwise.

HPURL parsers **MUST NOT** strip or truncate the query string after encountering other characters like `/`, `=`, or `#`. The full query string (after the first `?`) must be interpreted as-is.

### Example

```
URL: https://example.com/page#prefix?x=1&y=2/extra
Fragment: prefix?x=1&y=2/extra
HPURL query string: ?x=1&y=2/extra
Parsed result:
  {
    "x": "1",
    "y": "2/extra"
  }
```

## 5. Use Cases

- Client-side routing parameters in SPAs.
- Privacy-preserving analytics or tracking tokens.
- Embedding state in shareable URLs without triggering server reloads.

## 6. Compatibility

HPURLs are fully backwards-compatible with standard URLs. Since the fragment is never sent to the server in HTTP requests, use of HPURLs does not interfere with traditional web infrastructure.

Existing routing or anchor mechanisms using fragments can coexist with HPURLs if the application defines a convention for distinguishing them.

## 7. Security Considerations

- Sensitive data should not be encoded in HPURLs, as the fragment may be exposed through browser history, copy/paste, or analytics scripts.
- Applications should validate and sanitize HPURL inputs as with any user-supplied parameters.

## 8. IANA Considerations

This document has no actions for IANA.

## 9. References

- [RFC3986] T. Berners-Lee, R. Fielding, L. Masinter, "Uniform Resource Identifier (URI): Generic Syntax", STD 66, RFC 3986, January 2005.

## X. Extensions

### X.1 Reserved Parameters Extension

The reserved parameters used by HPURL parsers are:

- `$` — the Ed25519 signature.
- `@` — the Ed25519 public key.
- `!` — the Ed25519 signature scope.

Parsers typically use these values in the following sequence:

1. Read the signature from `$`.
2. Read the public key from `@`.
3. Read the scope from `!` to determine which parts of the URL the signature covers.

**However, HPURL does not require these parameters to appear in this order within the URL hash fragment.** Parsers **MUST** recognize and process the parameters regardless of their order in the fragment.

This approach allows flexibility in URL construction while maintaining a clear, consistent parsing and verification model.

### X.2 Signature Extension

The `$` parameter carries an Ed25519 signature covering the HPURL parameters, enabling verification of parameter integrity and authenticity.

#### Signature Computation

- The signature covers the whole HPURL, **except** the `$` parameter itself.
- By default, all parameters (search and hash) are lexicographically sorted by key before signing.
- Serialization is done as `key=value` pairs joined with `&`.
- The resulting UTF-8 byte string is signed using Ed25519.

#### Signature Verification

- Upon receiving an HPURL with a `$` parameter, the signature **MUST** be verified using the expected public key.
- If verification fails, the HPURL parameters should be considered untrusted.

### X.3 Signature Features

The `!` parameter encodes a bitmask representing enabled HPURL signature scope and features.

#### Signature Scope Bits

These bits specify the signature scope and features:

| Bit Position | Description                                     |
|--------------|------------------------------------------------|
| 0            | Parameter sorting control:<br>- 1 = parameters are lexicographically sorted by key<br>- 0 = parameters keep their original order in the URL |
| 1            | Protocol (scheme) is included in signature.     |
| 2            | Hostname (domain) is included in signature.     |
| 3            | Port is included in signature.                   |
| 4            | Path is included in signature.                   |
| 5            | Search parameters (query) are included.          |
| 6            | Fragment parts before the HPURL parameters are included. |
| 7            | Reserved for future use                          |

#### Signature Scope Semantics

- If the `!` parameter is **not present** but the `$` signature parameter **is present**, the signature is assumed to cover all standard URL parts (protocol, host, port, path, search, and fragment) with parameters lexicographically sorted (equivalent to bits 0–6 set).
- If the `!` parameter **is present**, its bits explicitly define parameter sorting (bit 0) and which URL components (bits 1–6) are covered by the signature.

#### Canonicalization for Signature Computation

Based on the feature flags bitmask, the canonical string to sign is constructed by concatenating the selected URL components in this order:

```
[protocol] + '://' + [hostname] + ':' + [port] + [path] + '?' + [search params] + '#' + [fragment excluding the `$` signature parameter]
```

- Only components whose bits are set (bits 1–6) are included.
- Parameters are lexicographically sorted by key if bit 0 is 1, or kept in original order if bit 0 is 0.
- The HPURL parameters (in the fragment after `?`) are serialized including reserved parameters except `$`.
- Components are concatenated as raw strings without decoding or re-encoding.