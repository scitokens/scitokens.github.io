
SciToken Claims and Scopes Language
====================================

Each SciToken has one or more claim associated with it.  These claims are typically used to indicate the authorizations the SciToken bearer may have -- but other use cases exist (such as adding monitoring or auditing information into the token chain).

In this document, we outline the claims language utilized by SciTokens.  Inherent to the SciTokens concept is flexibility: any downstream user or VO may extend our claims with their own, domain-specific ones!

Standard Claims
---------------

As a SciToken is a [JSON Web Token](https://jwt.io) at its base, we inherit a specific claims language from [RFC 7519](https://tools.ietf.org/html/rfc7519).  In this section, we outline the SciTokens-specific usage of the claims, denoting any changes in claim criticality.

* *sub* (Subject): Typically indicates the individual or entity this token was originally issued to.  The subject is considered to be locally unique in the context of the VO.  Usage of this claim is OPTIONAL.  The token validators SHOULD NOT utilize the subject for authorization or identity mapping.  Suggested use cases are auditing, monitoring, or tracing.  Due to privacy concerns, some VOs may issue non-human-readable subjects or multiple anonymized subjects per individual.  A VO SHOULD NOT use the same subject for multiple individuals.

* *nbf* (Not Before): The interpretation for `nbf` is unchanged from the RFC, but this is considered a CRITICAL attribute for a SciToken.

* *exp* (Expiration Time): The interpretation for `exp` is unchanged from the RFC, but this is considered a CRITICAL attribute for a SciToken.  The motivation behind the combination of `nbf` and `exp` being marked as critical is to allow resources to enforce maximum token lifetime limits.

* *iss* (Issuer): The issuer of the SciTokens; this MUST be populated in a token chain.  It MUST contain a unique URL for the organization; this unique key will later be used for validation and bootstrapping trust roots.  This is used to identify the virtual organization (VO) that issues the token; it is expected that services maintain a map of issuer URLs to coarse-grained authorizations.

* *aud* (Audience): A service URI the SciToken is authorized to access (note: requesting this to be a URI is a slight narrowing of the definition from RFC7519).  For example, if the VO has write access to several storage services, this claim may be utilized to limit a token to a single endpoint.  The `aud` claim is OPTIONAL.


SciToken-specific Claims
------------------------

The SciTokens project defines JWT claims specific to our problem domain.  From RFC 7519:

>   A producer and consumer of a JWT MAY agree to use Claim Names that
>   are Private Names: names that are not Registered Claim Names
>   (Section 4.1) or Public Claim Names (Section 4.2).  Unlike Public
>   Claim Names, Private Claim Names are subject to collision and should
>   be used with caution.

In the long-term, we hope to have our claim names registered with IANA.  Until then, we will
utilize the URI form.  A SciTokens validator MUST accept either URI or non-URI form.

* *authz*: A list of authorized activities the bearer of this token may perform.  For SciTokens, we aim to define a common set of storage authorizations, but envision additional authorizations will be added to meet new use cases.  The interpretation of this is a list of operations the bearer is allowed to perform.  Known operations are:

   * `read`: https://scitokens.org/v1/authz/read. Read data.
   * `write`: https://scitokens.org/v1/authz/write. Write data.
   * `queue`: https://scitokens.org/v1/authz/queue. Submit a task or a job to a queueing service.
   * `execute`: https://scitokens.org/v1/authz/execute. Immediately launch or execute a task.

   The operation definitions are currently kept open-ended and intended to be interpreted by the specific user community.

   When rendered in JSON, the value of the `authz` claim should be a JSON list if multiple claims are present.  For example, a read-only token may have a claim of `"authz": "read"` while a read/write token may have a claim of `"authz": ["read", "write"]`.

   The `authz` claim is REQUIRED.

* *site* (Site): https://scitokens.org/v1/site.  The "site name" of the service the SciToken is authorized to access.  Unlike the `aud` claim, the the site name is considered to be within the VO context, although site naming scheme may be organized by a community (such as a grid organization) or between the VO and site.  Since the `site` is within the VO context, a single service may recognize several site names.  There is no mechanism to ensure a site name is globally unique.

* *path* (Path): https://scitokens.org/v1/path.  The path this token is authorized to access, relative to the VOs base path on a storage system.  When examining the contents of this attribute, paths _must_ be normalized according to [section 6 of RFC 3986](https://tools.ietf.org/html/rfc3986#section-6).  Hence, `///foo/bar/../baz` and `/foo/baz` are considered the same path for the purpose of determining access permissions.  Multiple path components may be specified as a JSON list; in such a case, the *authz* permissions are applied to all paths.  If `read` or `write` authorizations are given, then `path` MUST be specified; no default value may be assumed.

SciTokens Scopes
----------------

The claims language defined by JWT is based on JSON, making it extremely flexible.  However, when creating an authorization token using the OAuth2 protocol, the client may only provide a space-delimited set of scopes; it is up to the authorization server to generate an appropriate SciToken with a list of claims.  SciTokens aims to standardize a set of scopes and the corresponding mapping to claims.

* *site:NAME*.  Use this scope to request a site claim for a site `NAME` in the returned authorization token.
* *authz:AUTHZ:PATH*.  Use this scope to request authorization against a given path.

The server-side parsing of scopes should follow [Section 3.3 of RFC6749](https://tools.ietf.org/html/rfc6749#section-3.3).

Care must be taken by the server in generating SciTokens in response to a request with multiple `authz` scopes.  Consider:

```
authz:read:/foo
authz:write:/foo
```

The corresponding token claims would be:

```
{
   "authz": ["read", "write"],
   "path":  "/foo"
}
```

The authorizations in the token match the requested scopes.  However, this example scope request is problematic:


```
authz:read:/foo
authz:write:/bar
```

A naive corresponding SciToken may be:

```
{
   "authz": ["read", "write"],
   "path":  ["/foo", "/bar"]
}
```

However, this is problematic as the token provides more authorization than requested: it can write to `/foo`, while the client only
requested write access to `/bar`.  Even if the identity has write access to `/bar`, it is possible the client intended to drop that
for this token.

A server MUST NOT return tokens with more authorization granted than requested.  A server MAY return a token with less authorization;
this may result in unexpected behavior for the client so a server SHOULD NOT return a token with less authorization.
For example, if the client requests the following scopes:

```
authz:read:/foo
authz:write:/foo/subdir
```

the server MAY return:

```
{
   "authz": ["read", "write"],
   "path":  "/foo/subdir"
}
```

However, it is suggested that the server respond with an error instead.

Examples
--------

In this section, we only show the token payload, not base64-encoded, and remove the standard claims in order to improve readability.

Suppose we have two token-issuing services, `https://cms.cern/oauth` for the CMS organization and `https://ligo.org/oauth` for LIGO.

A LIGO user who can read any LIGO file may need the following token:

```
{
   "authz": "read",
   "path":  "/",
   "iss":  "https://cms.cern/oauth"
}
```

This is equivalent to the following URI-form:

```
{
   "https://scitokens.org/v1/authz": "https://scitokens.org/v1/authz/read",
   "https://scitokens.org/v1/path":  "/",
   "iss":                            "https://ligo.org/oauth"
}
```

Note that the `path` claim is implicitly relative to a base authorization for the LIGO organization.  Here, `/` allows access to all LIGO files, _not_ all files in the storage service.

To stageout to `/store/user/bbockelm`, a part of the CMS namespace at the CMS site `T2_US_Nebraska`, a user would utilize the following token:

```
{
   "authz": "write",
   "path":  "/store/user/bbockelm",
   "iss":   "https://cms.cern/oauth",
   "site":  "T2_US_Nebraska"
}
```

The implementation of `site` is purposely ambiguous; hence, the `aud` claim may be used to restrict a token to a specific endpoint:

```
{
   "authz": "write",
   "path":  "/store/user/bbockelm",
   "iss":   "https://cms.cern/oauth",
   "aud":   "https://transfer.unl.edu"
}
```

If delegating tokens is supported, one can further restrict the permission given to an individual job.   To further restrict the above token to a specific sub-directory, the token chain would look like the following:

```
{
   "scope": "write",
   "path":  "/store/user/bbockelm",
   "iss":   "https://cms.cern/oath",
   "site":  "T2_US_Nebraska"
}
{
   "path":  "/store/user/bbockelm/job_1"
}
```

