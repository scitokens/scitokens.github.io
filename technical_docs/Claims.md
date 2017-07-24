
SciToken Claims Language
========================

Each SciToken has one or more claim associated with it.  These claims are typically used to indicate the authorizations the SciToken bearer may have -- but other use cases exist (such as adding monitoring or auditing information into the token chain).

In this document, we outline the claims language utilized by SciTokens.  Inherent to the SciTokens concept is flexibility: any downstream user or VO may extend our claims with their own, domain-specific ones!

Standard Claims
---------------

As a SciToken is a [JSON Web Token](https://jwt.io) at its base, we inherit a specific claims language from [RFC 7519](https://tools.ietf.org/html/rfc7519).  In this section, we outline the SciTokens-specific usage of the claims, denoting any changes in claim criticality.

* *sub* (Subject): Typically indicates the individual or entity this token was originally issued to.  The subject is considered to be locally unique in the context of the VO.  Usage of this claim is OPTIONAL.  The token validators SHOULD NOT utilize the subject for authorization or identity mapping.  Suggested use cases are auditing, monitoring, or tracing.  Due to privacy concerns, some VOs may issue non-human-readable subjects or multiple anonymized subjects per individual.  A VO SHOULD NOT use the same subject for multiple individuals.

* *nbf* (Not Before): The interpretation for `nbf` is unchanged from the RFC, but this is considered a CRITICAL attribute for a SciToken.

* *exp* (Expiration Time): The interpretation for `exp` is unchanged from the RFC, but this is considered a CRITICAL attribute for a SciToken.  The motivation behind the combination of `nbf` and `exp` being marked as critical is to allow resources to enforce maximum token lifetime limits.

* *iss* (Issuer): The issuer of the SciTokens; this MUST be populated.  It MUST contain a unique URL for the organization; this unique key will later be used for validation and bootstrapping trust roots.

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

* *vo* (Virtual Organization): https://scitokens.org/v1/vo. The virtual organization issuing the token.  This MUST be globally unique for the community utilizing the SciToken.  It MUST be utilized to determine the appropriate trust roots for token verification.  If present in a token, it MUST also be present in the header.  A token chain MUST have the VO specified in the base token.

* *scope* (OAuth scope): This is identical in semantics to the scope attribute in an OAuth token.  However, for SciTokens, we aim to define a common set of storage scopes.  The interpretation of this is a list of operations the bearer is allowed to perform.  Known operations are:

   * `read`: https://scitokens.org/v1/scope/read. Read data.
   * `write`: https://scitokens.org/v1/scope/write. Write data.
   * `queue`: https://scitokens.org/v1/scope/queue. Submit a task or a job to a queueing service.
   * `execute`: https://scitokens.org/v1/scope/execute. Immediately launch or execute a task.

   The operation definitions are currently kept open-ended and intended to be interpreted by the specific user community.  The `scope` claim is REQUIRED.

* *site* (Site): https://scitokens.org/v1/site.  The "site name" of the service the SciToken is authorized to access.  Unlike the `svc` claim, the the site name is considered to be within the VO context, although site naming scheme may be organized by a community (such as a grid organization) or between the VO and site.

* *path* (Path): https://scitokens.org/v1/path.  The path this token is authorized to access, relative to the VOs base path on a storage system.  _NOTE_: it is understood this path should be normalized to some extent (to consider `/ligo` and `//ligo` the same location), but defining this normalization has not yet been done).


Examples
--------

In this section, we only show the token payload, not base64-encoded, and remove the standard claims in orderto improve readability.

A LIGO user who can read any LIGO file may need the following token:

```
{"scope": "read", "path": "/", "iss": "https://cms.cern/oauth"}
```

This is equivalent to the following URI-form:

```
{"scope": "https://scitokens.org/v1/scope/read", "https://scitokens.org/v1/path": "/", "iss": "https://ligo.org/oauth"}
```

To stageout to `/store/user/bbockelm`, a part of the CMS namespace at the CMS site T2_US_Nebraska, a user would utilize the following token:

```
{"scope": "write", "path": "/store/user/bbockelm", "iss": "https://cms.cern/oauth"}
```

To further restrict the permission given to an individual job, one may chose to further restrict the above token to a specific sub-directory via chaining tokens:

```
{"scope": "write", "path": "/store/user/bbockelm", "iss": "https://cms.cern/oath"}, {"path": "/store/user/bbockelm/job_1"}
```

