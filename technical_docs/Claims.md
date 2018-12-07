
SciToken Claims and Scopes Language
====================================

Each SciToken has one or more claim associated with it.  These claims are typically used to indicate the authorizations the SciToken bearer may have -- but other use cases exist (such as adding monitoring or auditing information into the token chain).

In this document, we outline the claims language utilized by SciTokens.  Inherent to the SciTokens concept is flexibility: each deployment of SciTokens can add domain-specific claims for their own use cases.

Standard Claims
---------------

As a SciToken is a [JSON Web Token](https://jwt.io) at its base, we inherit a specific claims language from [RFC 7519](https://tools.ietf.org/html/rfc7519).  In this section, we outline the SciTokens-specific usage of the claims, denoting any changes in claim criticality.

* *sub* (Subject): Typically indicates the individual or entity this token was originally issued to.  The subject is considered to be locally unique in the context of the VO.  Usage of this claim is OPTIONAL.  The token validators SHOULD NOT utilize the subject for authorization or identity mapping.  Suggested use cases are auditing, monitoring, or tracing.  Due to privacy concerns, some VOs may issue non-human-readable subjects or multiple anonymized subjects per individual.  A VO MUST NOT use the same subject for multiple individuals.

* *nbf* (Not Before): The interpretation for `nbf` is unchanged from the RFC, but this is considered a CRITICAL attribute for a SciToken.

* *exp* (Expiration Time): The interpretation for `exp` is unchanged from the RFC, but this is considered a CRITICAL attribute for a SciToken.  The motivation behind the combination of `nbf` and `exp` being marked as critical is to allow resources to enforce maximum token lifetime limits.

* *iss* (Issuer): The issuer of the SciTokens; this MUST be populated in a token chain.  It MUST contain a unique URL for the organization; this unique key will later be used for validation and bootstrapping trust roots.  This is used to identify the virtual organization (VO) that issues the token; it is expected that services maintain a map of issuer URLs to coarse-grained authorizations.

* *aud* (Audience): A service the SciToken is authorized to access.  For example, if the VO has write access to several storage services, this claim may be utilized to limit a token to a single endpoint URI.  As in RFC7519, the `aud` claim is not necessarily a URI: for example, it might be the name of a target site.   The service may accept several different possible audiences; the service endpoint at `https://storage.example.com` may accept an audience of either `Site_Example` or `https://storage.example.com` but ought to reject an audience of `https://www.google.com`.  Additionally, the `aud` can be a special value of `ANY`, where it would allow any audience to match.  The `aud` claim is OPTIONAL in version 1.0, mandatory in 2.0.

  | Client      | Server      | Result  |
  |-------------|-------------|---------|
  | ANY         | ANY         | Success |
  | ANY         | example.com | Success |
  | example.com | ANY         | Fail    |
  | example.com | example.com | Success |
  | notwork.com | example.com | Fail    |

* *jti* (JWT ID): The interpretation for `jti` is unchanged from the RFC. It is a unique identifier that protects against replay attacks, improves traceability of tokens through a distributed system, and enables revocation.  The `jti` claim is OPTIONAL.

* *ver* (Version): The version of the token.  `ver` is used to version the claim formats and attribute names.  The format of the `ver` claim is: `<profile>:<version>`.  The `ver` claim is optional in SciTokens 1.0 but mandatory in all other versions; if absent, clients MUST process the token as belonging to the SciTokens 1.0 profile

SciToken Claim Semantics
------------------------

The SciTokens project defines JWT claims behavior specific to our problem domain.  From RFC 7519:

>   A producer and consumer of a JWT MAY agree to use Claim Names that
>   are Private Names: names that are not Registered Claim Names
>   (Section 4.1) or Public Claim Names (Section 4.2).  Unlike Public
>   Claim Names, Private Claim Names are subject to collision and should
>   be used with caution.

Currently, the SciTokens domain-specific claims include:

* *scope*: A JSON string containing a space-separated list of scopes associated with this token, in the format described in Section 3.3 of [RFC 6749](https://tools.ietf.org/html/rfc6749)[RFC6749].  For SciTokens, we aim to define a common set of storage authorizations, but envision additional authorizations will be added to meet new use cases.  The interpretation of this is a list of operations the bearer is allowed to perform.  Known authorizations are:

   * `read`: Read data from a resource.
   * `write`: Write data from a resource.
   * `queue`: Submit a task or a job to a queueing service.
   * `execute`: Immediately launch or execute a task.

   The operation definitions are currently kept open-ended and intended to be interpreted by the specific user community.

   SciTokens scopes may additionally provide a resource path, which further limits the authorization.  These paths are provided in the form `$AUTHZ:$PATH`.  For example, the scope `read:/foo` would provide a read authorization for the resource at `/foo` but not `/bar`.  Resources allow a hierarchical relationship to be expressed; an authorization for `write:/baz` implies a write authorization for the resources at `/baz/qux`.  Resources accepting SciTokens MUST handle these resource-based authorizations.

   For `read` and `write` scopes, `$PATH` must be specified; if not specified in the scope, the path may be assumed to be `/`.  When examining the contents of this attribute, paths _must_ be normalized according to [section 6 of RFC 3986](https://tools.ietf.org/html/rfc3986#section-6).  Hence, `///foo/bar/../baz` and `/foo/baz` are considered the same path for the purpose of determining access permissions.  As in RFC 3986, each component of the path must be URL-escaped.

   The `scope` claim is REQUIRED.

SciTokens Scopes
----------------

The claims language defined by JWT is based on JSON, making it extremely flexible.  However, when creating an authorization token using the OAuth2 protocol, the client may only provide a space-delimited set of scopes; it is up to the authorization server to generate an appropriate SciToken with a list of claims.  SciTokens aims to standardize a set of scopes and the corresponding mapping to claims.

* *aud:NAME*.  Use this scope to request an audience claim for `NAME` in the returned authorization token.  Multiple `aud` scopes may be given.
* *AUTHZ:PATH*.  Use this scope to request authorization against a given resource.

The server-side parsing of scopes should follow [Section 3.3 of RFC6749](https://tools.ietf.org/html/rfc6749#section-3.3).

A server MUST NOT return tokens with more authorization granted than requested.
For example, if the client requests the following scopes:

```
read:/foo write:/foo/subdir
```

the server MAY return a token containing the following:

```
{
   ...
   "scope": "read:/foo/subdir write:/foo/subdir
   ...
}
```


Example JWT
-----------

Suppose we would like to sign the following JWT header and payload:

```
{
  "alg": "RS256",
  "typ": "JWT"
}
{
  "sub": "bbockelm",
  "exp": 1509991790,
  "iss": "https://scitokens.org/cms",
  "iat": 1509988190,
  "scope": "read:/store write:/store/user/bbockelm",
  "nbf": 1509988190,
  "ver": "scitoken:2.0",
  "aud": "https://transfer-server.example.com"
}
```

Then, the corresponding base64-encoded and signed payload would be (line breaks are given only for clarity; should not be included in the actual token):

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJiYm9ja2VsbSIsImV4cCI6MTUwOTk5MT
c5MCwiaXNzIjoiaHR0cHM6Ly9zY2l0b2tlbnMub3JnL2NtcyIsImlhdCI6MTUwOTk4ODE5MCwic2Nvc
GUiOiJyZWFkOi9zdG9yZSB3cml0ZTovc3RvcmUvdXNlci9iYm9ja2VsbSIsIm5iZiI6MTUwOTk4ODE5
MCwidmVyIjoic2NpdG9rZW46Mi4wIiwiYXVkIjoiaHR0cHM6Ly9jbXMuZXhhbXBsZS5jb20ifQ.fCty
ZQQNPaowQ5FXFVIlbt2Qpb4ui8Bkl1qXpwLKI3FQ0AKP64Ozf7NLKI8nRHaAqh9XRQAxB9YtAJAeHri
SN422-CraARoYyBdrZMtwlxphOLPkpuxbIusVYB3r4zIRt4BoB7NlqLqwVV2e5rGtkJGvi9tpY2FNr7
eZ6eBrzAg
```

Examples SciToken scope
--------

In this section, we only show the token payload, not base64-encoded, and remove the standard claims in order to improve readability.

Suppose we have two token-issuing services, `https://cms.cern/oauth` for the CMS organization and `https://ligo.org/oauth` for LIGO.

A LIGO user who can read any LIGO file may need the following token:

```
{
   "scope": "read:/",
   "iss":  "https://ligo.org/oauth"
}
```

This is equivalent to the following URI-form:

```
{
   "scope": "https://scitokens.org/v1/authz/read:/",
   "iss": "https://ligo.org/oauth"
}
```

Note that the resource in the `scope` claim is implicitly relative to a base authorization for the LIGO organization.  Here, `/` allows access to all LIGO files, _not_ all files in the storage service.

To stageout to `/store/user/bbockelm`, a part of the CMS namespace at the CMS site `T2_US_Nebraska`, a user would utilize the following token:

```
{
   "scope": "write:/store/user/bbockelm",
   "iss":   "https://cms.cern/oauth",
}
```

The `aud` claim must be used to restrict a token to a specific endpoint:

```
{
   "authz": "write:/store/user/bbockelm",
   "iss":   "https://cms.cern/oauth",
   "aud":   "https://transfer.unl.edu"
}
```
