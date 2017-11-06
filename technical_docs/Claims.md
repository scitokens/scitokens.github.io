
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

* *aud* (Audience): A service the SciToken is authorized to access.  For example, if the VO has write access to several storage services, this claim may be utilized to limit a token to a single endpoint URI.  As in RFC7519, the `aud` claim is not necessarily a URI: for example, it might be the name of a target site.   The service may accept several different possible audiences; the service endpoint at `https://storage.example.com` may accept an audience of either `Site_Example` or `https://storage.example.com` but ought to reject an audience of `https://www.google.com`.  The `aud` claim is OPTIONAL.


SciToken Claim Semantics
------------------------

The SciTokens project defines JWT claims behavior specific to our problem domain.  From RFC 7519:

>   A producer and consumer of a JWT MAY agree to use Claim Names that
>   are Private Names: names that are not Registered Claim Names
>   (Section 4.1) or Public Claim Names (Section 4.2).  Unlike Public
>   Claim Names, Private Claim Names are subject to collision and should
>   be used with caution.

Currently, the SciTokens domain-specific claims include:

* *scp*: A list of authorized activities the bearer of this token may perform.  For SciTokens, we aim to define a common set of storage authorizations, but envision additional authorizations will be added to meet new use cases.  The interpretation of this is a list of operations the bearer is allowed to perform.  Known authorizations are:

   * `read`: Read data from a resource.
   * `write`: Write data from a resource.
   * `queue`: Submit a task or a job to a queueing service.
   * `execute`: Immediately launch or execute a task.

   The operation definitions are currently kept open-ended and intended to be interpreted by the specific user community.

   SciTokens scopes may additionally provide a resource path, which further limits the authorization.  These paths are provided in the form `$AUTHZ:$PATH`.  For example, the scope `read:/foo` would provide a read authorization for the resource at `/foo` but not `/bar`.  Resources allow a hierarchical relationship to be expressed; an authorization for `write:/baz` implies a write authorization for the resources at `/baz/qux`.  Resources accepting SciTokens MUST handle these resource-based authorizations.

   For `read` and `write` scopes, `$PATH` must be specified; if not specified in the scope, the path may be assumed to be `/`.  When examining the contents of this attribute, paths _must_ be normalized according to [section 6 of RFC 3986](https://tools.ietf.org/html/rfc3986#section-6).  Hence, `///foo/bar/../baz` and `/foo/baz` are considered the same path for the purpose of determining access permissions.  As in RFC 3986, each component of the path must be URL-escaped.

   When rendered in JSON, the value of the `scp` claim should be a space-separated list (as opposed to a JSON list) in order to match the behavior of scopes in OAuth2.

   The `scp` claim is REQUIRED.  Note that the `scp` claim is proposed for standardization as part of the [OAuth token exchange](https://datatracker.ietf.org/doc/draft-ietf-oauth-token-exchange/) draft RFC.

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
   "scp": "read:/foo/subdir write:/foo/subdir
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
  "scp": "read:/store write:/store/user/bbockelm",
  "nbf": 1509988190
}
```

Then, the corresponding base64-encoded and signed payload would be (line breaks are given only for clarity; should not be included in the actual token):

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJiYm9ja2VsbSIsImV4cCI6MTUxMDAwMT
I1MiwiaXNzIjoiaHR0cHM6Ly9kZW1vLnNjaXRva2Vucy5vcmciLCJpYXQiOjE1MTAwMDA2NTIsInNjc
CI6InJlYWQ6L3N0b3JlIHdyaXRlOi9zdG9yZS91c2VyL2Jib2NrZWxtIiwibmJmIjoxNTEwMDAwNjUy
fQ.oFYtTmTDDYiBpxj2jxKnFdRXoxKijspu3vTP990w4tnFVb91-5ahSCcHB82RpzyszzaDbCnU1PXo
rN-psHawXh4pSKuv-OxtupSZlNE1_djPBgn84voRTj5pjgyS5L6EDBuyKzo1R0h9UuOJu7-VtgtbObY
4TNx1GXdYZqQPYQFFqfiEsvG8LaRHpguCr-siTpoHFLQoYCRP8l8pHhyaXwwkNGfZJNAtNtj0UzuR_0
lkrAjpx118PxmPa0pMA8FPdSNRLtY8T4CDUN4FbUk-E4-V9OE8HCqdYyRtLTPUASyGtRzMfn_clynW8
BehtZjh9EneN9ceHPF_ddAFMrN_fA
```

Examples SciToken scope
--------

In this section, we only show the token payload, not base64-encoded, and remove the standard claims in order to improve readability.

Suppose we have two token-issuing services, `https://cms.cern/oauth` for the CMS organization and `https://ligo.org/oauth` for LIGO.

A LIGO user who can read any LIGO file may need the following token:

```
{
   "scp": "read:/",
   "iss":  "https://cms.cern/oauth"
}
```

This is equivalent to the following URI-form:

```
{
   "scp": "https://scitokens.org/v1/authz/read:/",
   "iss": "https://ligo.org/oauth"
}
```

Note that the resource in the `scp` claim is implicitly relative to a base authorization for the LIGO organization.  Here, `/` allows access to all LIGO files, _not_ all files in the storage service.

To stageout to `/store/user/bbockelm`, a part of the CMS namespace at the CMS site `T2_US_Nebraska`, a user would utilize the following token:

```
{
   "scp": "write:/store/user/bbockelm",
   "iss":   "https://cms.cern/oauth",
   "site":  "T2_US_Nebraska"
}
```

The implementation of `site` is purposely ambiguous; hence, the `aud` claim may be used to restrict a token to a specific endpoint:

```
{
   "authz": "write:/store/user/bbockelm",
   "iss":   "https://cms.cern/oauth",
   "aud":   "https://transfer.unl.edu"
}
```
