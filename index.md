---
layout: default
---

Overview
========

The SciTokens project aims to build a federated ecosystem for authorization on distributed scientific computing infrastructures.

We believe that distributed, scientific computing community has unique authorization needs that can be met by utilizing common web technologies, such as OAuth 2.0 and JSON Web Tokens (JWT).  The SciTokens team, a collaboration between technology providers and domain scientists, is working to build and demonstrate a new authorization approach at scale.

In distributed computing, a natural unit of organization is the "virtual organization" (VO), typically a group or community representing a scient domain or experiment that might span several physical institutions (such as a university of lab).  The VO has its own mechanisms to determine membership and access policies for resources it owns.  SciTokens aims to provide an infrastructure that allows the VO to issue bearer tokens that focus on the _capabilities_ the bearer should have within the VO's namespace, as opposed to the _identity_ of the bearer.  This frees resource providers from needing to duplicate VO authorization policies based on identity mapping.

# [](#header-1)Email Lists

As SciTokens ramps up, we will post draft technical documents describing the overall architecture, the token format, and the runtime environment.  In the meantime, feel free to subscribe to one of our project email lists:

*   announce@scitokens.org ([subscribe](mailto:announce+subscribe@scitokens.org)) ([archives](https://groups.google.com/a/scitokens.org/d/forum/announce))
*   discuss@scitokens.org ([subscribe](mailto:discuss+subscribe@scitokens.org)) ([archives](https://groups.google.com/a/scitokens.org/d/forum/discuss))
