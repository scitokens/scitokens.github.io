
SciToken Verification
=====================

SciTokens are intended to be utilized as part of a distributed high-throughput computing infrastructure.  Accordingly, an important aspect is _distributed verification_ -- being able to determine whether a given token has been issued by a known VO and whether the claims have been changed or altered.  This page provides a technical description of how a given token must be verified.

Each SciToken is issued by a single VO.  In order to verify the token itself, we must:
1. Determine the VO the token claims to be issued by.
2. Establish a set of trust roots to map a VO name to a set of known public keys.
3. Demonstrate that the token was signed by a known public key.
4. Determine the claims contained in the token follow the SciTokens domain-specific rules.

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

Establishing Trust Roots
------------------------

The base token in each chain must be associated with a given VO, using the `vo` claim.  To verify the base token, one must be able to map the VO name to a set of public keys representing the VO.  In this section, we describe how to perform this mapping process in a POSIX environment.

Given a VO name, `vo.example.com`, we define the search path:
1. If the `SCITOKENS` environment variable is set and a valid directory managed by the current effective UID, then utilize `$SCITOKENS/vo.example.com`.
2. If the current effective UID has a valid home directory, `~/.scitokens/vo.example.com`
3. `/etc/scitokens/vo.example.com`

To establish a set of trust roots, first establish a list of search files using the following pseudo-code algorithm:

```
search_files = []
for dir_path in search_path:
   files = list_valid(dir_path)
   sort(files)
   for file in files:
       search_files.append(file)
```

* `list_valid` lists all the files in a given directory NOT matching the regex ` ^((\..*)|(.*~)|(#.*)|(.*\.rpmsave)|(.*\.rpmnew)|(.*\.dpkg-old)|(.*\.dpkg-dist)|(.*\.cfsaved))$`
* The `sort` function sorts the file names in lexigraphical order.

For each file in the search path:

* If the filename ends in `.jwks`, its contents should be parsed as a JSON object representing a JSON Web Key Set as specified by section 5 of RFC 7517.
* If the filename ends in `.jku`, then the file contents should be parsed as a JWK Set URL as specified in section 4.1.2 of RFC 7515.  This mechanism allows the VO (or a delegate organization) to post a set of keys that should be used as trust roots (assisting with key rotation).
   * If a `.jku` file is used, an implementation MAY cache the set of keys locally.  If the keys are retrieved via a HTTP GET request and the server responds with a `Cache-Control` header as defined by RFC 7234, then the implementation MUST use this expire time.  Otherwise, max object age in the cache is implementation-defined.  An implementation SHOULD default the max age to less than 24 hours.
* Otherwise, the file is ignored.  Additional parsing rules may be added at a later date.

The above algorithm results in an ordered list of JSON Web Keys corresponding to each known VO name; these will be used in the next section.

Note the VO name is used as part of a path name in the validation; it is recommended the VO name be based on a known DNS name (such as `vo.example.com`), although historical names may be used for existing communities (`cms`, `atlas`, `ligo`, etc).  Case-sensitive string-matching rules from JSON should be used when doing the directory lookup.  (Author's note: this definition of VO name appears quite problematic.  We want to have a human-friendly name such as `ligo`, but really would benefit from having a global unambiguous identifier such as a URL).

Verifying a SciToken
--------------------

A SciToken MUST be a properly-formatted [JSON Web Token](https://jwt.io) (JWT), as described by [RFC 7519](https://tools.ietf.org/html/rfc7519).

The SciToken MUST be signed with an asymmetric key; the public portion of the key MUST be determined by the value of least one of the following JOSE header attributes:
* `pwt`: The parent web token (TODO: write up separate document and link to this), a SciToken used to sign another SciToken.  The parent web token itself MUST be verified according to this document.  The `pwt` MUST specify a "child web key" (`cwk`) and the SciToken MUST be signed by that `cwk`.
* `vo`: This header attribute is utilized to look up an ordered list of keys as defined in the previous section.  The SciToken MUST be signed by one of these keys.

If both headers are present, then they MUST both specify the key used for the token.  The definition of chained tokens utilizing the `pwt` attribute is currently incomplete and evolving.

If the `key` attribute is in the SciToken headers, then it must match (Author's note: here, "match" is relatively ambiguous) the public key used to sign the token.

Proof of Key Possession
----------------------

Author note: This section has not yet been completed.

Some SciTokens use cases require the client prove they are in possession of the private key associated with the token.  This Section will document the semantics for proving key possession.

SciToken-specific Claim Processing
----------------------------------

See the [claims definition](Claims.md) page for information on the SciTokens claim language.  In this section, we describe how to perform validation of claims.

* All claims MUST be considered valid by the entity performing validation.  If there are any unknown claims attributes - or claim values that cannot be validated - the entire token must be considered INVALID.  A token must be considered completely valid or invalid.
* Claim validation should proceed with the base token in any given chain; all parent token claims MUST be processed before a child's claims.
* New claim attributes MUST only be present in the base claim.  Any claim attributes in child claims MUST be also be present in the base claim.  Hence, the chaining mechanism may only be utilized to reduce or limit the authorizations in a claim - new ones MUST NOT be added.
* The `vo` attribute MUST be set in the base claim.

