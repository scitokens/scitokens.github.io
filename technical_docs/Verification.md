
SciToken Verification
=====================

SciTokens are intended to be utilized as part of a distributed high-throughput computing infrastructure.  Accordingly, an important aspect is _distributed verification_ -- being able to determine whether a given token has been issued by a known VO and whether the claims have been changed or altered.  This page provides a technical description of how a given token must be verified.

Each SciToken is issued by a single VO.  In order to verify the token itself, we must:
1. Determine the issuer the token claims to be issued by.
2. Demonstrate that the token was signed by a known public key associated with the issuer.
3. Determine the claims contained in the token follow the SciTokens domain-specific rules.

Finally, the resource provider must maintain a mapping from allowed issuers to associated resources they can authorize.  For example, the issuer `https://cilogon.org/ligo` may be allowed to issue tokens authorizing access to the storage locally at ``/user/ligo``.  Such mappings must be maintained by the site - but wider organizations (such as a grid community) may help map issuer URIs to well-known communities; (i.e., "LIGO tokens are issued by ``https://cilogon.org").

Long-term, an important concept for SciTokens is the ability to delegate (creating a new token from an existing one with separate cryptographic material) and attenuate (limiting the scope of claims in a delegated) tokens.  This forms an ordered _token chain_ that can be validated as a whole.  To allow distributed verification and delegation together, it may be required that the bearer of a SciToken is able to prove possession of a particular private key.  Many ideas for SciToken verification have been borrowed from the X509 / PKI ecosystem: one important distinction is that tokens are generated (potentially with a private key) and sent to the remote party over a confidential, integrity-protected channel.  This is in contrast with an X509 certificate-signing request, where the recipient is the only entity in possession of the private key: this design decision is considered acceptable as the creator of the token always has a superset of the authorizations of the recipient.

Support for token chaining may not be present in the initial SciTokens implementations (indeed, we intend to split this out to a separate document in the future); to understand the text in this document, a standalone SciToken may be considered a chain of length one.

Locating a SciToken
-------------------

In addition to web services, An important use case for SciTokens is POSIX-like command line and batch system environments.  Hence, we believe it is important to have an unambiguous way for a POSIX process to locate a SciToken it should use for authorization with SciToken-aware services.

Unless a token is explicitly specified by the end-user, SciToken client libraries should search the following locations for a SciToken file:
* `SCITOKEN` environment variable.
* On a POSIX-like platform, `/tmp/scitoken_u$ID`, where `$ID` is replaced by the POIX user ID.

On POSIX-like platforms, the file must be owned by the effective UID.  The file is parsed in a line-oriented form; each non-empty line *not* starting with `#` is to be parsed as a base64-encoded SciTokens.  The client library should determine which token is to be used for each authorization.  (Author note: we should provide guidance on how to determine the most restrictive potentially-valid token available.)

This document does not cover how a new SciToken might be generated or requested.

Verifying a SciToken
--------------------

A SciToken MUST be a properly-formatted [JSON Web Token](https://jwt.io) (JWT), as described by [RFC 7519](https://tools.ietf.org/html/rfc7519).

The SciToken MUST be signed with an asymmetric key (RSA- or EC-based signatures); the public portion of the key MUST be determined by the resource verifying the token.  The SciTokens team is researching additional headers for chaining tokens in a chain of trust.  Currently, suggested way to determine the public portion is:
- From the `iss` claim, use the [OAuth 2.0 authorization server metadata RFC](https://tools.ietf.org/html/draft-ietf-oauth-discovery-07) to determine the JWKS URI.
- The JWKS URI provides a list of public keys associated with the issuer; we currently rely on the use of the "key ID" attribute to determine the public key associated with the token See [RFC7517](https://tools.ietf.org/html/rfc7517) for more information.

Once the public key is determined, the verification of the token can proceed as outlined in RFC7519.

Proof of Key Possession
----------------------

[Author note: This section has not yet been completed.]

Some SciTokens use cases require the client prove they are in possession of a private key associated with the token without sharing the token itself.  This Section will document the semantics for proving key possession.  Initially, SciTokens will used as bearer tokens.

SciToken-specific Claim Processing
----------------------------------

See the [claims definition](Claims.md) page for information on the SciTokens claim language.  In this section, we describe how to perform validation of claims.

A token's version is found in the `ver` attribute.  If absent, the `ver` is `scitoken:1.0`

### Version 1.0

For tokens with no `ver` attribute or `scitoken:1.0`:

* All claims MUST be considered valid by the entity performing validation.  If there are any unknown claims attributes - or claim values that cannot be validated - the entire token must be considered INVALID.  A token must be considered completely valid or invalid.
* Claim validation should proceed with the base token in any given chain; all parent token claims MUST be processed before a child's claims.  [This item will be updated in the future as we develop any use cases for chaining.]
* New claim attributes MUST only be present in the base claim.  Any claim attributes in child claims MUST be also be present in the base claim.  Hence, the chaining mechanism may only be utilized to reduce or limit the authorizations in a claim - new ones MUST NOT be added.

### Version 2.0

For tokens with `ver` = `scitoken:2.0`

* Unknown claims are ignored and not used for authorization.
* Required Supported Claims:  ver, sub, nbf, exp, iss, aud, jti, iat, scope
* Signature algorithms and RS256, ES256 MUST be supported.

