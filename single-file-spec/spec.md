Delegated Authority Evaluations
==================

**Specification Status:** [[badge: Editor's Draft]]

**Latest Draft:**
  [https://identity.foundation/delegated-authority-evaluations](https://identity.foundation/delegated-authority-evaluations)

Editors:
~ [Dmitri Zagidulin](https://www.linkedin.com/in/dzagidulin/) - ([Interop Alliance](http://interopalliance.org/))

Contributors:

~ [Deb Bucci](https://www.linkedin.com/in/debbie-bucci/) ([Deb B Labs](https://docs.google.com/document/d/e/2PACX-1vQoFsZd-vSLZ0nr278sQ4t2UBK3-zrVFi1yt-uhM4u47_gOIL2LP4dd6SGEXkMBlL5dWwv08Nx9IIr4/pub))
~ [Juan Caballero](https://www.linkedin.com/in/juan-caballero/)  ([learningProof UG](https://learningproof.xyz/))
~ [Agne Caunt](https://www.linkedin.com/in/agne-c-a4495879) - ([Dock Labs](https://www.dock.io/))
~ [Makki Elfatih](https://www.linkedin.com/in/makki-elfatih-b835752a9/) - ([HKDolts](https://hkdolts.wixsite.com/mysite))
~ [Sachio Iwamoto](https://www.linkedin.com/in/sachio-iwamoto/) - ([Kyndryl](https://www.kyndryl.com/us/en))
~ [Alan Karp](https://www.linkedin.com/in/alanhkarp/) - [SitePassword](https://github.com/alanhkarp/SitePassword/)

**Related Specifications:**

- [Problem Space Report](https://identity.foundation/delegated-authority-report/)
- Specification Evaluations (You are here)
- [Threat Model](https://identity.foundation/delegated-authority-threat-model/)
- [Governance Considerations](https://identity.foundation/governance-of-delegated-authority-report/)
- [Agentic-Delegation User-Story Walk-through](https://www.youtube.com/watch?v=u-uWl_s0PPM)

Participate:

~ [GitHub repo](https://github.com/decentralized-identity/delegated-authority-evaluations)
~ [File a bug](https://github.com/decentralized-identity/delegated-authority-evaluations/issues)
~ [Commit history](https://github.com/decentralized-identity/delegated-authority-evaluations/commits/master)

------------------------------------

## Introduction

This report evaluates authorization and access control specifications specifically through the lens of agentic permissions and delegation.

Why agents specifically? Conventional access control was already strained by ordinary human sharing, but LLM-based agents completely break its assumptions. Agents are often short-lived, sometimes existing for only seconds, so provisioning each agent with accounts or ACL entries doesn't scale, and an agent's own identifier can never be the responsible party. Responsibility has to trace back to the principal operator who gave it the task. Agents can't be trusted to keep secrets (a clever prompt may extract a credential), and, unlike humans, they can't be deterred by post-facto punishments or personal consequences. In cases where conventional software has a defined API, an agent's behavior is effectively unbounded: the permissions it will need can't be fully known in advance, making "least privilege" a dynamic problem rather than a provisioning-time one. These properties push the authors toward forms of authorization that are delegated, attenuated, and verifiable at request time, rather than derived from an identity. Agentic AI’s fundamental risks suggest that any proof-of-possession keys must be held outside the agent itself (by a more deterministic component trusted to use them on the agent's behalf).

To address the problem of Agentic authorization and delegation holistically, we assessed specifications addressing various authorization layers. Their strengths and weaknesses are addressed primarily against their scope and goals, with an eye to their various possible recombinations and configurations. A given spec may address one or several of the following:

* Data model: how the various mandatory and optional data elements of an authorization get expressed.  
* Protocol: how tokens are requested, presented, and (optionally) revoked.  
* Proof of Possession mechanism: how tokens are secured against theft and replay, most commonly through some proof of possession of keys or other cryptographic material.  
* Chainable proof methods: how a multi-hop chain of delegations is expressed and verified.

Delegation is central to the problem of agentic authorization because the only alternative to delegation is impersonation. Delegation can be done without capabilities, using identity-based (authentication-centric) systems, but that entails some degree of ambient authority, which opens the door to “confused deputy” attacks, whereby too much faith is placed in fallible (or second-hand) authentication. Cedar and similar systems illustrate the point: they support more flexible delegation while remaining identity-based, so they remain subject to the confused deputy problem, and they also place policy evaluation on the critical path of every request, incentivizing attacks on the identity layer rather than the policy layer. The same applies one layer down, at the decision protocol. AuthZEN standardizes how an enforcement point asks a policy decision point for a verdict, but the authority consulted still lives at that server and its trust model is still based on  identity.

A quick note on the [confused deputy](https://en.wikipedia.org/wiki/Confused_deputy_problem) problem, which was central to our motivation and inquiry here. Typically, people refer to the slippage between logged identities and actual-executing, actually-liable parties as a confused-deputy situation, but we took a more existential approach: we think of impersonation and credential-sharing and exceptions to rigid logging or access policies as invitations, which make authentication and credential data the weakest links. Agents are nothing if not expert weakest-link finders! To remove the possibility of confused deputies, designation and authorization must be combined, explicitly stating which actions are allowed to be performed on which resource, verb, or API endpoint, rather than leaving that to be solved indirectly via authentication and policy. This combination is definitive of a family of approaches generally grouped under the category of "capability systems", explicitly formalizing capabilities, as opposed to systems relying on ambient authority (e.g. assigning authority by identity and/or role).

Capability systems can be categorized by which parts of the system do the enforcement. The main types are:

* hardware-backed capabilities (the oldest form)  
* object references enforced by the platform (which require a memory-safe language to be securable at scale)  
* opaque tokens  
* certificates  
* cryptographic-only (encryption-based) systems.

In certificate systems, metadata is bundled *with* the permission and presented together with it; in opaque token, platform, and hardware systems, the metadata is stored somewhere else and merely referenced. Certificate systems themselves come in two types: bearer tokens and proof of key possession.

Multi-hop (chainable) delegation is important. Without it, after a "single hop" delegation (traditional OAuth2), the system lends itself to impersonation (for example, copy-and-pasting an OAuth2 token into an agent’s prompt or context window for it to act as you in the system). In this increasingly common scenario, the audit trail is degraded, and it’s not possible to revoke the agent’s privilege without revoking the human’s privilege as well. At scale, this is an untenable workaround.

Attenuation, or the ability to delegate a smaller scope of capabilities, is an essential part of the picture. Delegation should enforce the **principle of least privilege**, giving an actor working on your behalf as little privilege (and incurring as little risk) as necessary. When enlisting help from an agent, the user should give it only the minimum permissions necessary to perform the task. Attenuation enforced by the capability system itself prevents escalation attacks, which were less frequent and existential a risk when enterprise-internal systems were only being used by human employees.

Traditional capability systems are intentionally not identity-based, and in many cases can be configured to be almost identity-blind. For agentic use cases, however, it’s important to constrain identifier choices (and maintain any maps or indirections necessary) so as to maximize accountability and logging, *particularly* because agentic use-cases involve so many liable parties, with non-determinism and risk distributed across so many components. Complicating identifier choices and classical best practices are the inherent challenges of cross-domain, cross-organizational identifiers, since agents often perform tasks across different web domains, trust zones, and execution contexts, which may in turn require partial confidentiality or bring their own logging and authentication requirements. Given all of the above, we take the strong stance that bearer tokens are dangerous to use at any layer of such a system. The primary danger is theft. A stolen bearer token is as good as the original, and tokens tend to appear in logs and in agent context windows. Replay of an intercepted token is the secondary risk. Luckily, this position is becoming [less controversial](https://www.ietf.org/archive/id/draft-klrc-aiagent-auth-03.html#section-7) over time across the security and identity communities, as more and more research and security incidents indicate the near- or total impossibility of trusting frontier models with such systems.

### Specification Evaluation Checklist

In this report we evaluate major authorization and/or access-control specifications that address any of the design questions listed in the Introduction: primarily, we study how delegation is supported by the data models, invocation protocols, and chainable proof methods, but also other aspects of protocol and logging are addressed as well. Validity checks must be done at the time of request (by a component trusted by the resource provider), and there is much room for configuration and profiling across all these systems, but we have focused on the core affordances of these specifications at a high-level.

The assessment criteria largely track Stiegler's seven aspects of sharing (see the [Delegated Authority Problem-space report](https://identity.foundation/delegated-authority-report)), with a few adaptations for spec evaluation. The adaptations are listed in the Appendix, under [Relationship to Stiegler's Seven Aspects of Sharing](#relationship-to-stieglers-seven-aspects-of-sharing).

For this report, the final list of assessment criteria is:

1. **Attenuated delegation**: delegation of a subset of permissions as a first-class data element.
2. **Accountability:** Specification must allow stable, in-band differentiation between the agent and its principal operator (e.g., natural or legal person)
3. **Confused-deputy Resistance:** relative infeasibility of impersonation and authentication attacks,  mostly from reducing “ambient authority.”
4. **Representability of authorization policies** (either by embedding or linking to them)
   1. This includes **functional policies** like “only usable during work hours” or “only from this network”
   2. This includes **re-delegation policies** like “only delegate within X boundary or role”
   3. "**Please**": A request containing desired rules for redelegating, where violations are not refused. Ideally the delegatee also has a place to record why it did not honor the request.
   4. "**Disclosure** **governance**": Confidentiality of tokens and logs is not just configurable but governable.
5. **Chainability**: Authorization must be able to be delegated: At root, authorization is delegated from principal/operator to agent. But also, agents must be able to delegate to further subsystems or other agents.
6. **Cross-organizational execution** (validation checks are independently verifiable, locally verifiable). At minimum, different organizations/groups should be able to express authorization policies without having accounts on each other's IAM systems.
7. **Self-revocability:** The capability holder, somebody in the delegation chain, or a holder of an explicit revocation capability, must be able to revoke a delegated permission, modulo network partitions.

A rule of the form "do not delegate further" is deliberately absent from the functional-policy examples. Forbidding re-delegation is widely regarded as an anti-pattern in capability systems. A delegatee can always proxy requests or share its key, so a prohibition only pushes the delegation out of view. The per-spec chapters still report whether such a rule is expressible, because it is frequently requested, but this report does not treat its absence as a shortcoming.

**Not required but nice to have:**

8. **Authentication grounded in first-class, in-band Proof of Possession** (numbered 8 in the scorecards so that row numbering stays continuous)

Two further nice-to-haves were considered and dropped from the checklist. Privacy of the delegation chain is discussed under Delegation Chain Privacy Considerations below. Offline capability is replaced by a per-stage account of which endpoints must be reachable, under Offline Use Considerations below.

### General Data Model (In Common)

Despite differences in encoding and terminology, the data models under evaluation share a common shape. A delegated permission tends to carry the following fields:

* "agent"—how the agent is identified (most often a DID or a key descriptor).  
* "principal"—who this permission is on behalf of.  
* "resource"—the resource the permission is given for.  
* "actions" —actions/operations allowed on that resource.  
* "caveats" / "conditions"—additional restrictions and conditions on use.  
* "delegation chain"—the cumulative record of the upstream delegations made in a multi-hop delegation system.


Alongside these fields, each spec also addresses several mechanisms that sit partly in the data model and partly in the protocol:

* "protocol"—how to request this permission.  
* "revocation"—how a permission is revoked. This is mostly protocol, with a small data model aspect, such as where a revoker submits a revocation. Only the verifier needs to know where revocation status is checked.  
* "invocation" (separate protocol and data model)—how to invoke a permission for a given resource.  
* signature suites—how delegations and invocations are signed, and how key possession is proven. Signing is the only cryptography a delegation system requires. Selective encryption of fields is an optional addition, touched on under Delegation Chain Privacy Considerations below.


A table mapping each spec's concrete field names onto this common shape appears in the Synthesis chapter, under Data Model Correspondence.

### A Note on Terminology

To discuss delegation, we need to at least have names for the parties involved. We use the following terms:

* Delegat**or**:The party who is *doing* the delegating. The source of the authorization. This one is easy, most texts agree on this one.  
* Delegat**ee**:The party who *is being delegated to*. The target of the delegation. This term is harder. Some texts also refer to this party as a “delegate” (noun). This  makes sense, but in written form, it's easy to confuse with 'delegate' (verb). Some texts even use “delegee” for this party. For ease of use, this report standardizes on 'delegat**ee**'.


Of course, these roles are transitive, and a delegatee becomes in turn the delegator the moment they sub-delegate whatever permission or authorization they have to yet another party in the chain.

### Delegation Chain Privacy Considerations

Earlier drafts of this report listed privacy of the delegation chain as a nice-to-have criterion. The question was whether a system can hide some or all of the delegation data and metadata in the proof chain from the agent or token carrier, and from intermediate counterparties, while the principal and the verifier still know the full story. The criterion was dropped because it does not discriminate among the candidates. The answer is the same for every specification surveyed, and it is settled by architecture rather than by any feature of a given spec.

None of the surveyed specifications encrypts, blinds, or selectively discloses the parties in a delegation chain. Where the chain is a portable artifact (zCaps, UCANs, dSD-JWT), the whole chain reaches the verifier on invocation and is visible to the invoking agent that carries it. UCAN says this directly. Chains are provenance logs, and UCANs do not offer confinement. dSD-JWT inherits selective disclosure from SD-JWT, but that mechanism hides alternative pre-signed payloads, not the hops themselves. Where there is no portable chain (OAuth, AAuth, Cedar, AuthZEN), the delegatee presents only its own token or identity and learns nothing about the rest of the graph. That is only because the graph lives at a server or policy store that sees everything. An OAuth Authorization Server performing Token Exchange, for instance, can record every delegation it brokers; that record is visible to the server and hidden from delegatees. The Person Server in AAuth and the policy store in Cedar hold whatever delegation history exists, and AuthZEN defines no chain representation at all. Carrier-side privacy in these systems is a side effect of centralization, not a mechanism.

In practice this matters less than the original criterion implied. What a chain exposes is a sequence of identifiers and the permissions passed between them. Using ephemeral, pairwise, or one-off identifiers for each delegation provides sufficient privacy of the delegation chain for most agentic use cases. zCaps already use urn:uuid capability identifiers to reduce correlation, and any DID-based system can mint a fresh key per delegation. A verifier or an intermediate agent then sees that a chain exists and that each edge is validly signed, but cannot correlate the parties across chains or link them to a real-world entity. Any additional information about the actors in a chain is stored out of band, as DID document contents, verifiable credentials, or other identifier-anchored metadata, and is disclosed on its own terms. This is compatible with the accountability criterion (item 2). An auditor can resolve pairwise identifiers to responsible parties through that out-of-band layer without the chain itself carrying global identities.

Selective disclosure or blinding of chain interiors against the verifier remains an open design problem. The "disclosure governance" sub-item of the policy criterion (item 4) is where a specification would express rules about who may see what. We note chain privacy here as a consideration for future work rather than as an evaluation axis.

### Offline Use Considerations

Earlier drafts of this report listed offline capability as a nice-to-have criterion, defined as the ability to create, delegate, and present tokens without network access. The criterion was dropped because a single verdict cannot carry the answer. For every specification surveyed, the honest verdict was a Partial that had to be unpacked by lifecycle stage. The question that does discriminate among the candidates is narrower. At each step of a delegated operation, which endpoints must be reachable, and what can be cached in advance so that the step succeeds without them?

The table below answers that question for five stages. Verdicts are drawn from the per-spec chapters, which hold the reasoning.

* Obtain authority: the first grant, from the principal or resource owner to the agent.  
* Extend the chain: a delegatee grants a subset of its authority to a further agent.  
* Invoke: the agent exercises the authority against the resource.  
* Verify: the enforcement point checks that the presented authority is valid.  
* Check revocation: the enforcement point learns whether the authority has been withdrawn.

| Stage | OAuth 2.0/2.1 | AAuth | zCaps / UCANs | KYA-OS | Cedar | AuthZEN | dSD-JWT |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| Obtain authority | Authorization Server token endpoint | Agent Provider for the agent token. Identity-based mode needs no further issuance; resource-managed and PS-asserted modes need the resource or the Person Server | None. The resource owner signs a root capability locally | The resource owner or its authorization service signs the root credential, typically after a consent challenge returned by the server | Policy store write by an administrator | Not defined; delegation is outside the specification | Issuer signs the base credential and delivers it to the holder |
| Extend the chain | Authorization Server token exchange endpoint (RFC 8693). No holder-side extension | Person Server and Mission Log; server-mediated | None. The delegatee signs a child capability with its own key. Only the next delegatee's DID or key is needed in hand | None. The delegatee signs the next hop's credential locally, as for zCaps | Policy store write, then distribution to edge PDPs | Not defined | None. The holder signs the next hop locally |
| Invoke | Resource Server | Resource, by signed request | Resource Server (zCaps) or executor (UCAN). A local or LAN resource suffices | MCP server, by a signed request proof carried in the call; stdio or HTTP | Local embedded PDP (Cedarling) | PEP calls the PDP for each decision; the PDP may be local or remote | Verifier |
| Verify | Local for JWT access tokens (RFC 9068) given cached Authorization Server keys. Opaque tokens need the introspection endpoint (RFC 7662) | Local signature check against keys from well-known metadata and JWKS. Keys MUST be cached and SHOULD still be used for up to 24 hours after a failed fetch; initial discovery is online | Local chain check. zCaps carry the full chain in the invocation; UCAN resolves proofs by CID from a local store or the network. DID resolution for chain keys needs the network unless DID documents are cached or the method is self-certifying (did:key) | Local chain check once the credentials are in hand, but the invocation carries a locator and the chain is resolved from a delegation registry. DID resolution as for zCaps; the recommended one-off did:key delegatee needs none | Local once the policy store is loaded. Identity evidence is JWTs checked against pre-configured issuer keys | PDP decision; there is no artifact to verify locally | Local. The root key comes from DID resolution or issuer metadata unless cached; hop keys are embedded JWKs |
| Check revocation | Revocation is requested at the Authorization Server (RFC 7009). The Resource Server learns of it through introspection or by waiting out short lifetimes; a locally verifying JWT Resource Server does not learn of it before expiry | At the servers holding authority (Person Server, Access Server); the resource learns through its interaction with them | zCaps: revocation is posted to the Resource Server revocation endpoint, and the RS consults its own state at invocation. UCAN: the executor's revocation store, filled by whatever dissemination the deployment uses (gossip, DHT); behavior under partition is undefined | Bitstring Status List fetched from each hop's issuer, cascading root to leaf. An unreachable list reads as revoked; the top conformance level requires a live read. A verifier that issues its own delegations may keep the state locally | Policy store update, reaching edge PDPs on refresh | Policy change at the PDP's store | Token Status List at the issuer, for the base credential only. Nothing below the root; short expiry is the fallback |

Three patterns stand out.

The certificate capability family and dSD-JWT are the only columns in which obtaining and extending authority need no reachable endpoint. KYA-OS shares the family's local chain extension but obtains its root through a server-side consent flow, and its verifier resolves the chain from a registry rather than receiving it with the invocation. Delegation is a local signing act, so a chain can grow with no round trip and while partitioned from every server. This is the efficiency advantage the earlier criterion was reaching for, and it is also where Stiegler's "dynamic" aspect reads off the table directly. In the server-mediated columns, every hop is a protocol exchange with an Authorization Server or Person Server. In the policy decision columns, a grant is an administrative write to a store.

Verification is local in nearly every column, provided the right material is cached. The divide is in what must be cached and how stale it may be. A JWT Resource Server caches Authorization Server keys. AAuth bounds its key cache at 24 hours. A zCap or UCAN verifier needs the DID documents of every key in the chain, or a self-certifying DID method, to check the chain with no network. Cedar needs a loaded policy store. AuthZEN is the exception, since the decision is the PDP's and there is nothing to cache in its place.

Revocation is online in every column. Each surveyed mechanism is state at some enforcement point that a party must reach, either to post the revocation or to learn of it. The capability family moves that point to the resource itself, which helps the local-machine and LAN use cases, but no column offers revocation semantics under partition. This is the gap recorded as item 4 in the Synthesis chapter's Gaps list.

For the local-machine and LAN-only agentic use cases, the practical reading is by column. The capability family and Cedar operate fully within the LAN once DID documents or the policy store are in place. OAuth does so only when the Authorization Server is itself on the LAN. AAuth and AuthZEN depend on the Person Server or PDP being reachable, wherever it sits.

## OAuth Family Evaluation

_Hardening, Consolidation, and the Push Toward Autonomous Authorization_

This chapter covers what is actually deployed today—OAuth 2.0 (RFC 6749/6750), as hardened by RFC 9700 (2025 Security Best Current Practice)—then traces the evolution toward OAuth 2.1. AAuth, a separate draft by an OAuth co-author, is evaluated in the chapter that follows.

Each section below opens with an italicized tag, one of:

* Deployed standard


* Adjacent RFC/profile


* OAuth 2.1—consolidation draft


These standards clarify the most common solution in use for  Google, Okta, Auth0, GitHub, or a typical enterprise IdP today, versus what's still in development and limited/niche deployments.

### Overview

OAuth (Open Authorization) is an open standard for **access delegation** anchored in HTTP calls, session state, and redirects. It allows users to grant third-party applications secure, scoped access to their data hosted on another service without ever handing over their passwords. Much like a "valet key" for a car, which lets a valet drive and park the vehicle but prevents them from unlocking the glovebox or trunk, OAuth uses limited-scope, session-bound **Access Tokens** rather than shared passwords or other static, long-lived credentials to protect user data.

The standard running in production today—what every major identity provider implements for most clients at scale— is OAuth 2.0: RFC 6749 (the core authorization framework) and RFC 6750 (bearer token usage), both from 2012, with additional security on higher layers. OAuth 2.0 is very much a framework, not a fixed protocol: it deliberately leaves many security-relevant decisions to implementers, architects, and configuration. This flexibility is precisely why a decade of real-world attacks produced a long list of follow-on RFCs tightening specific gaps (PKCE, JWT access tokens, token exchange, sender-constraining, DPoP, and so on).

In January 2025, the IETF published RFC 9700, "Best Current Practice for OAuth 2.0 Security," which consolidates that decade of lessons into concrete guidance (summarized under RFC 9700: The Hardening Layer in Force Today, below). RFC 9700 is guidance on top of RFC 6749/6750. It doesn't supersede or deprecate them.

OAuth 2.1 (draft-ietf-oauth-v2-1) is the next step, versioning the protocol rather than just profiling it. It is an attempt to fold RFC 9700's hardening directly into a single normative document that deprecates RFC 6749/6750. As of this writing, it is still an Internet-Draft in its [seventh year of iteration](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/). "OAuth 2.1" is best understood as the direction the ecosystem is consolidating toward, with most major 2.0 deployments largely compliant with minor changes, not a separately deployed protocol version.

Separately, AAuth (draft-hardt-oauth-aauth-protocol) is an independent, early-stage draft built specifically for agent-to-resource access, designed to coexist with OAuth and OIDC, not to extend either. It is evaluated in its own chapter, which follows this one.

### What's Actually Deployed: OAuth 2.0 plus RFC 9700

#### The Core Framework

*Deployed standard.* RFC 6749 defines the four classic grant types (Authorization Code, Implicit, Resource Owner Password Credentials, Client Credentials) plus Refresh Tokens. RFC 6750 defines how bearer tokens are presented to a resource server. Some form of Client Registration is [assumed but out of scope](https://datatracker.ietf.org/doc/html/rfc6749#section-2) of RFC 6749; many later RFCs extend alternate mechanism. This is the substrate nearly every production OAuth deployment is built on.

#### RFC 9700: The Hardening Layer in Force Today

*Deployed standard / BCP.* RFC 9700 is the document that defines  a secure OAuth 2.0 deployment.

RFC 9700 mandates the following:

* Don't use the Implicit grant (`response_type=token`). Tokens returned in a URL fragment are exposed to browser history, referrer leakage, and any script on the page.


* Don't use Resource Owner Password Credentials. Password-based delegation trains users to type credentials into third-party UIs and gives the client raw password access.


* Use PKCE (RFC 7636\) on the Authorization Code flow for all clients, not just public ones. PKCE it closes authorization-code-interception attacks that affect confidential clients too.


* Match redirect URIs exactly; no substring or wildcard matching.


* Prefer sender-constrained tokens (DPoP or mutual TLS) over plain bearer tokens where feasible.


* Use state for CSRF protection and PKCE's `code_verifier` together. Each protect against different attacks.


None of this requires specific use of "OAuth 2.1" as a separate thing to implement. OAuth 2.1's contribution is mostly editorial consolidation: combining these BCP recommendations and making the safe behaviors the only correct specified behaviors.

#### Client Authentication: Required vs. Common Practice

*Deployed standard / Adjacent RFC.* RFC 6749 only requires that confidential clients authenticate to the token endpoint somehow. `client_secret_basic` and `client_secret_post` are the baseline methods, widely used today.

Asymmetric client authentication has become common best practice for higher-assurance deployments:

* `private_key_jwt` (RFC 7523). The client signs an assertion with its own private key instead of sending a shared secret.


* Mutual TLS client authentication (RFC 8705). The client authenticates via its TLS certificate.


Both methods still require the client to hold a long-lived secret, a private key in one case and a certificate key in the other. As the Introduction notes, agents cannot be trusted to keep secrets. In an agentic deployment the key must therefore be held by a component outside the agent, which signs on its behalf.

These are additions layered onto OAuth 2.0, not outright mandates of  RFC 9700 or OAuth 2.1.

#### Proof of Possession: DPoP and mTLS

*Adjacent RFC.* By default, OAuth 2.0 bearer tokens (RFC 6750\) are bearer instruments. Whoever holds the token can use it.

Two specs address this:

* DPoP (RFC 9449). The client generates a key pair and signs a proof bound to each request's HTTP method and URI. A stolen token is useless without the matching key.


* Mutual TLS (RFC 8705). Binds the token to the client's TLS certificate.


RFC 9700 recommends sender-constraining. It doesn't require it, and most OAuth 2.0 deployments still issue plain bearer access tokens.

#### Token Format: Opaque vs. Self-Contained

*Deployed standard / Adjacent RFC.* OAuth access tokens can be opaque (validated via token introspection, RFC 7662\) or self-contained JWTs (verifiable offline per the JWT Access Token profile, RFC 9068). The choice is left to implementers by RFC 6749.

#### Revocation

*Deployed standard / Adjacent RFC.* RFC 7009 allows a client or resource owner request explicit revocation. Short-lived access tokens provide a passive backstop. OAuth 2.0 has no native notion of delegation-chain revocation. The only chain it has is the act claim an Authorization Server records when it performs Token Exchange (RFC 8693), and revoking one token in such a chain does not cascade by specification. Delegation semantics are difficult to expose to the user, but delegations can be tracked across the logs of the Authorization Server.

### OAuth 2.1 : Where the Consolidation Is Heading

*OAuth 2.1 consolidation draft.* OAuth 2.1 (draft-ietf-oauth-v2-1) takes everything in Part 1's hardening layer and includes it in the core spec.

Key changes in OAuth 2.1 vs. today's hardened OAuth 2.0:

* Implicit and ROPC grants are removed from the spec text, not just discouraged.


* PKCE is required, not recommended, for every client on the Authorization Code flow.


* Redirect URI exact-matching is specified, not left as guidance.


* Sender-constrained tokens (DPoP/mTLS) remain a SHOULD, not a MUST. OAuth 2.1 does not mandate proof-of-possession tokens.


* Client secrets are still supported. The draft explicitly requires AS support for clients sending a plain `client_secret`. Asymmetric client authentication is not the OAuth 2.1 baseline.


A well-hardened, RFC 9700-compliant OAuth 2.0 deployment today is very close to "OAuth 2.1 compliant". OAuth 2.1's main value is editorial: a single document new implementers can build against without needing to piece together a  decade’s worth of separate BCP RFCs.

### Part 4: Comparative Scorecard

*Note: A comparative scorecard placing OAuth 2.0 and AAuth alongside the capability systems previously appeared here. It has been superseded by the merged scorecard in the Synthesis chapter, which derives its verdicts from each specification's own chapter.*

### Closing Note

In production today, almost all systems are using OAuth 2.0, hardened to RFC 9700's recommendations, with client secrets still alive and well at most token endpoints and bearer tokens still the default token type. OAuth 2.1 is where that hardening is headed as a single normative document. However, the changes are fairly minor. It doesn’t mandate PoP nor does it eliminate client secrets.

AAuth, meanwhile, isn't a future version of OAuth at all. It's a parallel, signature-based protocol built because its author judged some core assumptions of the OAuth model itself a poor match for autonomous agents. AAuth, however, pushes complexity and risk into a fiduciary agent. This super-agent manages delegations, token-swaps, and other intermediary security and trust decisions. While AAuth ceremonies are different than those in OAuth, most of the token formats are lifted directly from OAuth, making it more of a logical variant than a complete overhaul of OAuth. UCAN/zCAP-LD remain the only one of the three with native, offline, multi-hop delegation (the credential-based dSD-JWT draft in the Appendix shares that property) \-- the clearest architectural line separating capability systems from the many HTTP Servers defining both the OAuth lineage and its AAuth variant (which adds a third Server).

## AAuth Evaluation

### Overview

AAuth is an emerging authorization architecture designed for authenticated interactions among people, AI agents, resources, Person Servers (PS), and Agent Providers (AP). Rather than treating software agents as extensions of end users, AAuth introduces authenticated agents as first-class participants capable of invoking protected resources, receiving asynchronous events, and operating across organizational boundaries. Crucially, the “container” for these cross-organizational distributed transactions is called a “mission” (dovetailing with [other work](https://notes.karlmcguinness.com/mission-handbook/) by OAuth Working Group insiders). Although AAuth grew out of the OAuth community and reuses several of its token formats, it is evaluated here in its own chapter, as a bespoke protocol for managing and auditing agentic distributed transactions.

Unlike capability-based delegation systems such as UCAN and zCaps, AAuth does not represent delegated authority as a self-contained, cryptographically attenuated artifact carried entirely within a delegated credential. Instead, delegation is server-mediated. Authority is anchored within the Person Server, with delegation history represented through the act claim, Mission References, and the Mission Log maintained by the Person Server. AAuth therefore defines explicit delegation mechanics while intentionally separating delegation state from portable capability objects.

AAuth is evaluated here as a server-mediated delegation architecture. It authenticates participants, establishes trusted execution between them, records delegation history, and supports authenticated event-driven execution. Questions concerning delegation semantics, attenuation policy, authority evolution, and the policy logic behind execution-time decisions remain outside the protocol itself and are expected to be supplied by complementary architectural layers.

Three reference points are used:

*  [draft-hardt-oauth-aauth-protocol-10](https://datatracker.ietf.org/doc/html/draft-hardt-oauth-aauth-protocol-10) (6 August 2026), together with [draft-hardt-httpbis-signature-key-08](https://datatracker.ietf.org/doc/html/draft-hardt-httpbis-signature-key-08), the companion specification to which AAuth defers key conveyance, discovery, and algorithm determination;


* The AAuth [Events draft](https://dickhardt.github.io/AAuth/draft-hardt-aauth-events.html), which extends the architecture with authenticated asynchronous event delivery between resources and agents through Agent Providers; and


* public implementation discussions describing the intended architecture and deployment model.


Where protocol specification and implementation practices differ, this distinction is noted throughout the evaluation.

A note on method: This chapter was drafted with AI research collaborators under a gated review process, with final editorial authority held by the author. Load-bearing claims were verified directly against the AAuth specification repository rather than model recall — including the token-exchange question, the act claim's provenance, credential key-binding, and the Appendix B.3.7 attenuation rationale.

### Origins and Architecture

Caveat: this is a single-author Internet-Draft, not an IETF working-group product. It has already been renamed and restructured more than once. Treat the below as a snapshot of a moving target.

#### Why It Exists

AAuth (draft-hardt-oauth-aauth-protocol) was created by an OAuth 2.0/2.1 co-author who concluded that OAuth, in any version, is a poor fit for agent-to-resource access because:

* Agents need authorization decisions made mid-task against resources they've never interacted with before.


* Agents have no durable `client_id` that travels across organizations the way OAuth assumes.


* Copying long-lived API keys into autonomous workloads recreates the shared-secret leak surface OAuth 2.0 and 2.1 are trying to move away from.


AAuth is explicitly designed to coexist with OAuth 2.0 and OpenID Connect, not extend either one.

#### Architecture

Roles (See [Terminology](https://www.ietf.org/archive/id/draft-hardt-oauth-aauth-protocol-10.html#section-3), in newest draft):

* Person/Principal  
* Agent Provider (AP): The Agent Provider is a centralized entity which manages agent identity and issues agent tokens, providing centralized governance over a distributed agent fleet.  
* Agent  
* Person Server (PS): server that represents the principal to rest of the protocol. The person chooses their PS; it is not imposed by any other party. The PS manages missions, handles consent, asserts user identity, and brokers authorization on behalf of agents.  
* Access Server (AS): A policy engine that evaluates token requests, applies resource policy, and issues auth tokens on behalf of a resource.  
* Resource


Token types:

* `agent_token` establishes the agent's identity


* `resource_token`, issued by the resource, states which authorization (resource, scope, and optionally account) the resource requires before it will serve the agent's request. The agent presents it to the Person Server or Access Server to obtain an `auth_token`


* `auth_token` grants access to a specific resource, where applicable


Core mechanism: Signed HTTP requests using HTTP Message Signatures, not bearer tokens trusted on presentation. The simplest mode requires no token issuance flow at all.

#### Four Resource Access Modes

AAuth defines four modes, increasing in complexity:

* Identity-based access: The agent signs requests with its `agent_token`. The resource verifies the signature and applies its own access policy. No authorization flow, no additional tokens. This replaces API keys with cryptographic identity.


* Resource-managed access (two-party): The resource handles authorization itself, optionally returning an opaque token for subsequent calls.


* PS-asserted access (three-party): A Person Server vouches for claims about the person on whose behalf the agent acts, with a consent step and an `auth_token` issued on approval.


* Federated access (four-party): Extends the three-party model across an additional trust boundary.


Key discovery: Agents and resources discover signing keys via well-known metadata and JWKS-style endpoints, similar in spirit to OAuth's JWKS discovery, but used for HTTP-signature key lookup, not bearer-token verification.

### Scorecard

#### Attenuated

Verdict: Partial — differs by mode

Sub-agent authorization supports attenuation: every request passes through the parent, which can refuse, attenuate, or rate-limit. Call chaining does not — downstream authorization is intentionally not required to be a subset of upstream scope. Cryptographically enforced attenuation is not intrinsic to the protocol in either case.

#### Accountable (agent vs. principal/operator)

Verdict: Yes

AAuth distinguishes authenticated people, agents, Person Servers, Agent Providers, and resources. Delegation history records acting parties through the act claim.

#### Resistant to confused deputy (no ambient authority)

Verdict: Yes in the three token-based modes; No in identity-based mode

The resource-managed, PS-asserted, and federated modes bind an authorization to a designated resource and scope through the resource token, combining designation with authorization. The **optional** account parameter added in draft-10 narrows that binding further, naming which account at the resource the authorization covers and propagating through the resource token into the auth token. Identity-based access does not do this: the resource authorizes on the agent's identity alone and applies its own access control, which is ambient authority.

#### Represent authorization policies

Verdict: Partial

Authorization context and Mission References are represented, but expressive delegation policy—for example, undelegatable authority, advisory policy, or attenuation constraints—is intentionally out of scope of the protocol, or inside the logic and authority of the Person Server.

#### Chainable

Verdict: Yes

Delegation chains are represented through the act claim and the Mission Log. Chains are server-mediated rather than portable capability artifacts. A sub-agent may not have sub-agents of its own. Workflows that need more depth use call chaining instead, in which each agent in the chain is a top-level agent holding its own grant from the Person Server, rather than a sub-agent of the one before it.

#### Cross-organizational / locally verifiable

Verdict: Yes

Designed for authenticated interactions across organizational boundaries while preserving local trust relationships.

#### Self-revocable

Verdict: Partial

Token revocation, mission revocation, and propagation through a parent's grant are all specified. Revocation is exercised by the servers holding authority rather than by a holder acting on a credential in hand, and mission state administration beyond completion is deferred to a companion specification.

#### Authentication / Proof of Possession

Verdict: Yes

Core function of the protocol. `draft-10` requires a fully-specified alg identifier, recommends Ed25519, and prohibits none, symmetric algorithms, and the polymorphic EdDSA identifier that RFC 9864 deprecated.

### Detailed Evaluation

#### Attenuated

Attenuation in AAuth differs by delegation mode. Sub-agent authorization supports it directly: every sub-agent request passes through the parent, which can refuse, attenuate, or rate-limit. Call chaining, by design, does not support delegation.  Downstream authorization is intentionally not required to be a subset of upstream scope, and downstream scope is constrained by the downstream resource's own policy together with Person Server evaluation against mission context.

Unlike UCAN or zCaps, attenuation is not represented as a cryptographically enforced property of a portable delegation artifact. Instead, attenuation depends upon server-managed mission state and policy evaluation.

The protocol therefore leaves attenuation semantics outside its definition, supplying contextual evaluation in place of algebraic constraint.

This is deliberate design rather than omission. AAuth does not adopt the RFC 8693 token-exchange model for attenuation (it borrows only that RFC's act claim structure for representing delegation chains), and Appendix B.3.7 states the rationale directly: downstream scope is intentionally not required to be a subset of upstream scope, because "the PS evaluates each hop against the mission context, providing governance-based constraints... more flexible than algebraic attenuation rules." The one place attenuation exists within an agent's own purview is sub-agent authorization, where the parent "can refuse, attenuate, or rate-limit" the sub-agent's authority.

#### Accountable (agent vs. principal/operator)

AAuth explicitly distinguishes among authenticated people, AI agents, Person Servers, Agent Providers, and protected resources. Rather than treating software agents merely as extensions of user sessions, the protocol gives each participant its own authenticated identity and architectural role.

Delegated actions are represented through the act claim, allowing delegation history to be maintained across direct delegation, sub-agent delegation, and chained execution. Mission References further associate requests with the governing mission maintained by the Person Server.

Unlike certificate capability-based systems, accountability derives from authenticated protocol exchanges combined with server-maintained mission history rather than from cryptographically self-contained delegation credentials.

The specification also permits a resource to authorize an agent based solely on its identity, without interaction or mission context; working-group discussion has flagged that identity-only authorization reintroduces confused-deputy exposure, since the execution context rather than the requester's identity is what validates an action.

#### Resistant to confused deputy (no ambient authority)

The confused deputy problem arises where a party exercises its own permissions on a resource designated by someone else. The defense is to combine designation with authorization, so that what may be done is bound to what it may be done to.

In the resource-managed, PS-asserted, and federated modes AAuth does this. The agent obtains a resource token naming the resource and scope; the auth token issued against it carries that binding, and the resource enforces it. The OPTIONAL account parameter, added in \-10, narrows the binding further: it names which account at the resource the authorization is for, drawn from the resource's own namespace, and is echoed through the resource token into the auth token, so an auth token for one account grants nothing at another.

One caution applies. The account parameter scopes an authorization to an existing account at the resource. Deployments that lean on it may let agents operate within a person's account instead of under agent-specific grants, which drifts toward credential sharing and the audit and revocation problems that come with it.

Identity-based access is the exception, and a deliberate one. The agent signs requests with its agent token, and the resource applies its own access control based on who the agent is. There is no authorization flow and no designation carried with the request. The specification presents this as a replacement for API keys, which it is; but authority derived from identity alone is ambient, and an agent acting on a request supplied by another party has no way to signal that the request is not its own.

#### Ability to Represent Authorization Policies

AAuth represents authorization context through missions, authenticated requests, and protocol-defined execution relationships. Mission References bind requests to an existing mission while allowing authorization policy to remain under the control of the Person Server.

The protocol intentionally does not standardize a rich delegation policy language. Concepts such as undelegatable permissions, advisory delegation preferences, attenuation constraints, or policy composition remain responsibilities of external policy engines or governance frameworks.

This separation allows AAuth to remain focused on authenticated execution while supporting a wide variety of higher-level authorization models.

#### Chainable

AAuth provides explicit delegation mechanics through the act claim and the Mission Log. These mechanisms record delegation history across direct delegation, chained delegation, and sub-agent authorization.

Unlike certificate capability-based systems, delegation history is maintained by the Person Server rather than embedded entirely within a portable delegation credential. Verification therefore depends upon authenticated interaction with the server maintaining mission state rather than validating an independently portable capability chain.

Accordingly, AAuth supports delegation chains, but those chains are server-mediated rather than self-contained cryptographic artifacts.

Sub-agent nesting is limited to a single level: a sub-agent must not have sub-agents of its own, and the specification enforces this from both directions. Workflows that need more depth use call chaining instead. In call chaining each agent is a top-level agent that holds its own grant from the Person Server, rather than a sub-agent of the agent that called it, which is also why the upstream-subset rule does not apply to it. What the specification defers to a companion document is mission state administration beyond completion: revocation state transitions, delegation-tree queries, and administrative interfaces. Delegation-chain lifecycle is not deferred; sub-agent revocation propagates through the parent's grant.

#### Cross-Organizational / Locally Verifiable

Supporting authenticated interactions across organizational boundaries is one of AAuth's principal architectural objectives.

Person Servers, Agent Providers, agents, and protected resources may participate across independent trust domains while preserving authenticated relationships between participants. Authentication evidence can therefore span organizations without requiring centralized identity management.

Trust remains anchored in authenticated protocol interactions and configured trust relationships rather than portable authorization credentials.

From a capability-based perspective, the need for Access Server federation may indicate that authority remains server-mediated rather than fully represented by the delegation credential itself. If the delegation credential carried sufficient authority for cross-domain verification, no ongoing relationship between authorization servers would be required.

#### Self-Revocable

AAuth includes mechanisms for token revocation and lifecycle management.

\-10 specifies named revocation scenarios: a Person Server revoking an auth token it issued or provided, an Access Server revoking one it issued, an agent provider revoking an agent token, and a Person Server revoking a mission, after which subsequent token requests referencing that mission are denied. Revocation also propagates structurally. Revoking a parent's grant causes the next sub-agent authorization to fail, and tokens already issued expire on their own at the end of their short lifetimes. The draft does not fix an auth token lifetime; its examples use one hour, and agent tokens SHOULD NOT exceed 24 hours.

The specification is candid about the limits. Verifying an auth token does not consult its issuer, so nothing in the verification path reports that a token has been revoked; a party that no revocation request reaches is bounded only by token lifetime. What remains unspecified is mission state administration beyond completion, deferred to a companion specification. Revocation is therefore exercised by the servers holding authority rather than by a holder acting on a credential in hand.

### Summary

AAuth represents a distinct approach to delegated authorization for AI systems. Rather than expressing delegated authority as portable cryptographic capabilities, it anchors authority within the Person Server and maintains delegation continuity through authenticated protocol exchanges, mission state, and recorded delegation history. In the taxonomy of capability models, AAuth authorizations are neither bearer capabilities nor certificate capabilities: they are server-mediated proof-of-possession credentials, and the specification assigns multi-hop safety to mission-context evaluation rather than to algebraic attenuation, expecting complementary layers to supply the latter where required. Delegation in AAuth references stable agent identifiers rather than individual keys. The cnf claim binds each authorization token to the agent's current signing key while the identifier persists across key rotation. This reduces coupling between delegation history and key lifetime, but introduces lifecycle questions concerning delegation validity across key rotations—whether a delegation was issued before or after a rotation, and how invocations arriving after should be treated. Capability systems using delegation-specific or one-time keys largely avoid this issue, since delegated credentials typically expire before key rotation becomes relevant.

Its strengths lie in authenticated interoperability, cross-organizational execution, event-driven interaction, and maintaining coherent delegation history throughout mission execution. Unlike capability-based systems, authority remains associated with server-managed mission state rather than being embodied entirely within portable delegation artifacts.

This architectural choice intentionally separates authenticated execution from higher-level governance concerns. Authentication and delegation establish who is requesting execution and under what delegated authority. The Permission Endpoint now provides a normative execution-time decision surface: an agent may ask whether a specific action is permitted and receive granted or denied, recordable in the mission log. What remains outside the protocol's scope is the policy logic that produces that answer, together with attenuation policy, authority evolution, mission governance, and dynamic authority state.

Accordingly, AAuth is best understood as a server-mediated delegation and execution framework. It provides authenticated execution, delegation continuity, and mission coordination while allowing complementary policy and governance architectures to determine whether delegated authority remains valid as mission context, organizational policy, and authority state evolve.

## Certificate Capabilities Evaluation (zCaps, UCANs, and KYA-OS)

### Overview

Three of the specifications in this survey are realizations of the same underlying model, and this chapter evaluates them together. Authorization Capabilities ("zCaps", standardized in draft as [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/)) and the [User-Controlled Authorization Network](https://ucan.xyz/) ("UCANs") are both certificate capability systems: the authorization is a structured, cryptographically signed artifact that a delegator hands to a delegatee, carrying within itself everything a verifier needs to check it. Both descend from the [object-capability](https://en.wikipedia.org/wiki/Object-capability_model) tradition (UCAN cites [SPKI](https://en.wikipedia.org/wiki/Simple_public-key_infrastructure) as its direct ancestor), and both make the same core design choice: access is decided by possession ("do you hold a valid capability, and can you prove control of its key?") rather than by identity ("who are you, and what does my policy store say about you?"). A fourth artifact of the same general kind, the credential-based dSD-JWT draft, is covered in the Appendix.

The third, [KYA-OS](https://github.com/decentralized-identity/kya-os-mcp) (Know Your Agent Operating System, a DIF Trusted AI Agents Working Group specification with a Model Context Protocol binding), can be thought of as an extended profile of zCaps. Its delegation credential is a W3C Verifiable Credential 2.0 whose `credentialSubject` is an attenuated ZCAP-LD capability, carried one credential per delegation hop. Around that delegation core KYA-OS adds what the other two leave to companion layers:

* a typed identity document for agents and servers (the Entity Card);
* a Responsible Party and Principal in the data model;
* a per-request holder-of-key proof carried in the MCP `_meta` channel;
* a server-signed response proof for audit;
* a status-list revocation mechanism.

KYA-OS also ships an older delegation shape (a VC 1.0 with a custom payload and a non-standard signature). This chapter evaluates only the VC 2.0 plus ZCAP-LD profile, which is the one the Entity Card path issues and verifies and the one the working group ratified.

The shared model works as follows. The root of authority is the resource owner; there is no central authorization server. A capability answers whether a given request, for this action on this resource, is authorized, and under which restrictions. It does not ask who the requester is. Delegation is performed offline, by signing a new capability with a key the parent authorized; no round trip to the resource server or any other party is needed to delegate. Each delegation step must monotonically attenuate: a delegatee can pass on no more than they hold, along whatever axes the spec defines (actions, time, resource scope, policy). At invocation time, the delegatee presents the capability together with its delegation chain and a proof of key possession, and the verifier checks the chain locally and cryptographically.

Because the three specs score as a family against the evaluation checklist, the scorecard below covers them in one table, with the shared reasoning stated once and per-spec nuances called out. KYA-OS departs from the family verdict on three rows (accountability, local verifiability, and self-revocation), and those departures get their own paragraphs. The second half of the chapter covers where the specs genuinely diverge: encoding, chain transport, resource scoping, revocation mechanics, and a few features present in only one of them. The design choice KYA-OS makes, of carrying a capability inside a Verifiable Credential, has consequences beyond this one spec and is treated separately in the next chapter, Modeling Chained Delegation with W3C VCs.

Reference points used in this evaluation:

* the [ZCAP v0.3 specification](https://w3c-ccg.github.io/zcap-spec/) (the data model), and the [zCap Developer's Guide](https://github.com/interop-alliance/zcap-developer-guide), which documents the deployed state of the art; where deployment has diverged from the spec text (most notably: typed caveat objects are largely unused in production, replaced by URL-suffix attenuation, allowedAction, and expires), this is called out explicitly;  
* the [UCAN specification set](https://ucan.xyz/specification/): the overview plus the Delegation, Invocation, and Revocation sub-specs. (The Promise sub-spec is skipped as still marked draft.)  
* the [KYA-OS protocol specification](https://www.kya-os.org/mcp/introduction) (version 1.0.0, in particular the delegation chapter and its VC 2.0 plus ZCAP-LD profile), the KYA-OS Entity Card specification (the per-request proof, the delegation join, and the verification algorithm), the published JSON Schema for the delegation credential, and the reference implementation's chain evaluator, which is where the attenuation rules are actually enforced.

### Scorecard

Each entry gives the requirement, the verdict for each spec, and the shared reasoning.

**1. Attenuated.** zCaps: Yes. UCAN: Yes. KYA-OS: Yes.

All three enforce monotonic attenuation, on different axes. zCaps: allowedAction subsets, no-later expires, URL path/query suffix narrowing of invocationTarget, and the reserved caveat slot. UCAN: capability sets must not widen, nbf/exp validity intersection, command-lattice narrowing, policy statements. KYA-OS: allowedAction subsets, no-later validity, monotone typed caveats; the invocationTarget is held constant along the chain rather than narrowed. In all three, a caveat or policy statement added at a hop is a further attenuation axis, inherited by every hop below.

**2. Accountable (agent vs. principal/operator).** zCaps: Partial. UCAN: Partial. KYA-OS: Yes.

zCaps and UCAN: every delegation edge is signed and thus attributable to a responsible delegator, but neither data model distinguishes "agent" from "legal-entity operator"; both are opaque DID-identified principals, and identity binding is delegated to a companion layer (VCs and issuer registries for zCaps; DID methods and higher-level identity systems for UCAN). Where a fresh key is minted for each delegation, accountability is hop by hop: each delegator vouches for the next key, and an auditor resolves one edge at a time. KYA-OS builds that layer in: a Responsible Party (the root issuer) and a Principal (the immediate human delegator) are named, typed, and recomputed from the chain on every verification.

**3. Resistant to confused deputy.** zCaps: Yes. UCAN: Yes. KYA-OS: Yes.

By construction, as capability systems: designation and authorization travel together in the signed artifact (invocationTarget plus allowedAction; subject times command times policy), so a delegatee never exercises its own ambient authority on a resource designated by another party. This holds to the extent the designation is specific. A capability whose target is a URL prefix or a broad command designates a set of resources, and the resistance is correspondingly coarser. KYA-OS inherits the zCap pair and adds a per-hop audience and an invocation-time designation rule.

**4. Represent authorization policies (functional restrictions, re-delegation rules, advisory "please").** zCaps: Partial. UCAN: Partial. KYA-OS: Partial.

All three provide an extensible policy slot and none ships a delegation-control vocabulary. zCaps reserve a generic caveat property, unused in deployment. UCAN defines a policy syntax (the pol property) and reserves the /ucan command namespace. KYA-OS standardizes two caveat types (ValidUntil, MaxAmount) and a rule that unknown caveats must be copied verbatim down the chain, but nothing for delegation control. None has an advisory (non-enforced) tier; all are binary enforce/reject.

**5. Chainable.** zCaps: Yes. UCAN: Yes. KYA-OS: Yes.

The headline feature of the family. zCaps: parentCapability plus capabilityChain proof chains. UCAN: hash-linked self-certifying certificates rooted in the resource owner. KYA-OS: one credential per hop, parentCapability links, the same continuity rule as zCaps. All cover operator to agent and agent to sub-agent, with links creatable offline.

**6. Cross-organizational / locally verifiable.** zCaps: Yes. UCAN: Yes. KYA-OS: Partial.

Verification is cryptographic and local in all three; principals are DIDs, so no party needs an account on another's IAM system. zCaps mandate the full chain be carried in the invocation, with no verifier network access required. UCAN resolves proofs by CID from a local store, DHT, gossip, or REST. KYA-OS references the chain by a locator that the verifier resolves from a delegation registry, so verification depends on that registry being reachable or cached. DID resolution and revocation freshness may still touch the network in all three.

**7. Self-revocable.** zCaps: Yes (at RS). UCAN: Yes (per Revocation spec). KYA-OS: Partial.

Same guarantee in the first two, opposite mechanics. zCaps: any controller in the chain POSTs the zCap to an RS revocation endpoint; out-of-band, RS-enforced, online-only. UCAN: revocation is a first-class command (/ucan/revoke) with a Revoker role, expressed inside the same capability model; dissemination is left open (DHT, gossip, etc.). Earlier UCAN versions (before 1.0) described revocation as a free-standing signed statement rather than a command, so deployments on those versions differ. KYA-OS: the direct issuer or any ancestor may revoke, with cascading effect, via W3C Bitstring Status Lists; the holder of a delegation may not revoke it, and revocation as a delegable permission is deferred to a future version.

**8. Authentication / Proof of Possession (nice-to-have).** zCaps: Yes. UCAN: Yes. KYA-OS: Yes.

zCaps: HTTP Message Signatures (Cavage, migrating to RFC 9421) or a Data Integrity capabilityInvocation proof; the request itself is signed. UCAN: the Varsig envelope is signed by the issuer key, with a required nonce, time bounds, and signed invocations. KYA-OS: a detached JWS per request over a canonicalized claim set that binds the request body, the recipient's DID, and a nonce, with an optional RFC 9421 sibling signature. All are DPoP-analogous: an intercepted invocation cannot be replayed without the key.

### Detailed Evaluation

#### Attenuated

All three specs enforce monotonic attenuation, checked by the verifier at every link: a delegator can hand out no more than they hold. The axes differ.

zCaps attenuate along three deployed axes: allowedAction in a child must not be less restrictive than the parent's; expires must not be later than the parent's (and every delegated zCap must have an expiry); and invocationTarget may be narrowed by appending a URL path or query suffix (/bars/123 to /bars/123/bazzes/456?day=tuesday). The caveat property is reserved as a fourth axis but unused in practice.

UCAN attenuates abstractly: each direct delegation must restate or diminish its permissions, the validity interval is the intersection across the chain (latest nbf to earliest exp), and commands narrow along a lattice, where shorter command paths prove longer ones in the same namespace (/crud can prove /crud/read but not /stack/pop). Policy statements and arguments provide further attenuation axes.

KYA-OS attenuates along the zCap action and time axes and along typed caveats, and drops the resource axis. A child's `allowedAction` must be a subset of its parent's; the credential's `validUntil` must be no later than the parent's; for each parent caveat the child must carry a caveat of the same type that is at least as narrow, with `MaxAmount` compared as exact decimals; and the `invocationTarget` must be identical at every hop. Any broadening hop invalidates the whole chain. The verifying resource is told to assert its own identity as the expected target, so a chain minted for a different resource fails at the root rather than being accepted by an unrelated verifier.

The limitations are mirror images. zCap attenuation of the resource is restricted to URL-suffix shaping, fine for hierarchical REST resources but awkward for non-hierarchical scoping. A URL prefix also designates a set of resources rather than one, so designation is only as specific as the URL structure allows. A single zCap carries exactly one invocationTarget (multiple target/action combinations are listed as future work). UCAN's abstraction is more flexible but less prescriptive: resource scoping is left to how cmd and args are interpreted, with no normative URL rules in the core spec. KYA-OS gives up resource narrowing entirely: the target is a DID naming a server or service, and everything below it (which tools, which files) is expressed as action strings, so a delegator who holds authority over a whole server cannot hand down authority over one path of it except by action naming conventions.

#### Accountable: agent vs. principal/operator

zCaps and UCANs satisfy the accountability half of this criterion structurally and the differentiation half only implicitly. KYA-OS satisfies both.

The actor is named directly: the controller field of a zCap, the issuer/audience/subject DIDs of a UCAN, or the invoker of a KYA-OS capability. Accountability across the chain is a structural guarantee in all three. Every zCap delegation carries a capabilityDelegation proof signed by a controller of its parent, every UCAN must be signed by the issuer DID's key, and every KYA-OS hop is a credential signed by the delegator whose own hop named it as delegatee. For every edge in the delegation graph there is a cryptographically attributable, non-repudiable responsible party, and walking the chain back to the root recovers the complete provenance of any authority.

What zCaps and UCANs do not provide is a semantic distinction between "this DID is an autonomous agent" and "this DID is the legal entity that stands behind it." A principal is an opaque DID; the root controller is the operator only by convention. Both specs push the binding of key identifiers to real-world legal entities explicitly out of scope, and for similar reasons: it is not always desirable or possible, and forcing it would be a privacy violation in many settings. Where the mapping is needed, both point at the same companion pattern: use an identity layer (Verifiable Credentials, issuer registries, DID methods) for the human-judgement call at the entry point, and use the capability chain for the flow of authority thereafter. An auditor can always identify which key delegated which authority to which other key; note that this auditability can be fulfilled with local pairwise identifiers only, without global identities.

KYA-OS is the one surveyed specification that builds the companion layer into the same artifact set. Its Entity Card gives every agent and server a DID anchored identity document with a closed `entityType` vocabulary, so the data model can say whether a DID is an agent, a human, an organization, or a service. The Card names a `responsibleParty` and, when distinct, a `principal`, but asserts neither: the verifier recomputes the Responsible Party as the issuer of the root credential in the chain, and recomputes that the leaf delegatee equals the key that signed the request. Reputation, KYC and KYB attestations, and trust-registry standing attach to the Responsible Party rather than to the agent's (often ephemeral) key, which is the arrangement the Introduction asks for. Two caveats apply. First, the root of authority is defined inconsistently: the protocol specification defines the Responsible Party as the organization or person accountable for the agent, while the ratified delegation profile requires the root issuer to equal the resource owner when one is asserted, and the consent flow has the resource's authorization service issue the root credential. In a flow where a user's agent acts against a third party's server, those are different parties, and which of them the chain is "rooted at" is left to deployment. Second, the identity binding is purchased by wrapping the capability in a Verifiable Credential, with the consequences discussed in the next chapter.

#### Resistant to confused deputy

Resistant by construction, and this shared property is the reason the family exists. In all three specs, designation and authorization are inseparable parts of the signed artifact: a zCap binds invocationTarget to allowedAction, a UCAN capability is defined as subject times command times policy, and a KYA-OS capability carries the zCap pair. The verifier evaluates the presented chain, not the invoker's own ambient standing, so there are no deputy permissions to confuse. A delegatee asked to act on a resource designated by someone else either holds a capability whose designation covers that resource, or the invocation fails.

KYA-OS adds two rules on top of the inherited pair. An invocation must designate the specific action it exercises, and a verifier must reject an invocation that relies on a multi-action delegation without naming one. And every non-root credential must carry an `audience` binding it to a specific verifying server, in addition to the `invocationTarget`. Since the target is itself a DID in this profile, the two fields frequently name the same entity, and the audience rule is best read as a defense against a credential minted for one server being presented to another that happens to host the same resource identifier.

#### Ability to represent authorization policies

The least emphasized area for all three specs as currently deployed, and the clearest shared gap.

All have the extension point. The zCap spec defines a generic caveat property: typed restriction objects (the spec's examples are ValidWhileTrue and DriveNoMoreThan) inherited down the chain, whose meaning "must be understood between the entity adding a caveat and the target evaluating it." UCAN goes further on machinery: policy is a dedicated top-level property (pol) with a [defined syntax](https://ucan.xyz/delegation/#policy) of statements and operators, alongside arguments (args) that constrain the command, and a reserved /ucan command namespace for community-blessed commands. KYA-OS keeps the zCap caveat slot and is the only one of the three to standardize entries for it: `ValidUntil` (a date, no later than the parent's) and `MaxAmount` (a decimal limit in a named currency, no greater than the parent's). It also specifies how a verifier treats a caveat type it does not know: the child must carry it verbatim, so a delegatee cannot shed a restriction by re-delegating through a verifier that does not understand it.

None ships a delegation-control vocabulary. In zCap deployments no caveat notation is currently used at all; attenuation is expressed implicitly through allowedAction, expires, and URL-suffix narrowing. UCAN's policy syntax expresses conditions on invocation arguments (the spec's examples are of the form "from this email address"), but standardizes nothing for delegation control. KYA-OS's two caveats constrain time and spend, not re-delegation.

Against the commonly requested policy examples, the three specs fail identically:

* "Do not delegate further" / "this permission is undelegatable": not natively expressible in any of them. zCaps have no depth-limit or non-delegation flag; the only depth control is a verifier-global chain-length cap (the RS's blanket rule, not a per-capability instruction). UCAN likewise defines no such caveat; delegation is constrained only by the attenuation rules. KYA-OS takes the zCap position explicitly: a chain-length cap is the verifier's local resource limit, not a protocol rule, and a chain one verifier refuses is not thereby invalid elsewhere. (In the zCap design philosophy this is deliberate: forbidding delegation is considered an anti-pattern, since a delegatee can always proxy or share the controlling key.)  
* Re-delegation policies: not standardized in any; all would need custom caveats or resource-specific policy semantics.  
* Advisory "please" rules (desired re-delegation rules whose violation is not refused): no notion of advisory, non-enforced policy in any of the three. Caveats and policies are evaluated at invocation and a failure invalidates the invocation. The model is binary enforce/reject, with no honor-if-possible tier. KYA-OS does have an advisory concept, but at the transport layer rather than the policy layer: its outbound delegation headers include advisory scope and chain hints that a verifier must ignore for authorization decisions. Its trust registry is likewise advisory, on the stated grounds that refusing a well-formed delegation pushes the same authority through credential sharing instead.  
* "Who gets to know what is a governance topic": consistent with the object-capability philosophy, all three deliberately encode authority, not knowledge or identity policy; disclosure governance is out of scope.


So all three data models can carry policy through an extensible slot, none ships a shared vocabulary for the policies the checklist cares about, and current deployments do not use the slots for delegation control. This is arguably the highest-value area of future work for the family, if it is to serve agentic delegation.

#### Chainable

The headline strength of the family and the reason the model exists.

In zCaps, delegation is expressed by parentCapability (pointing up at the parent or the root) plus a capabilityDelegation proof whose capabilityChain enumerates the ancestry. The root of authority is the resource itself: a root zCap controlled by the operator, from which authority flows root to agent to sub-agent. In UCAN, each token is a self-certifying certificate attesting a subset of its delegator's authority, hash-linked to its proofs and rooted in a resource owner; the Subject (sub) field marks whose authority the chain is about, and the chain is by definition a provenance log. In KYA-OS, each hop is a separate credential whose `parentCapability` names the parent capability's id, whose issuer must equal the parent's delegatee, and whose root `parentCapability` must equal the `invocationTarget`, which is the zCap root convention restated.

All three directly satisfy the two senses the checklist asks for: principal/operator to agent (root to first delegation), and agent to further subsystems or other agents (the zCap spec's valet example, car to Alyssa to Ben to Lem, is precisely a multi-hop agent-to-agent chain). In all three, each link is created offline, simply by signing a new child artifact with a key the parent authorized; no involvement of the resource server is required to delegate. KYA-OS does add a server-mediated step for obtaining the root: a server that lacks a delegation covering the request returns a signed `needs_authorization` challenge with a consent URL, and the resource's authorization service issues the root credential after the user consents. Extending the chain below that root is a local signing act, as in the other two.

#### Cross-organizational / locally verifiable

Strong in all three by design, with one qualification for KYA-OS. Verification is purely cryptographic and local to the verifier: it traverses the chain from the root down, checking each delegation proof against the key authorized by the step above, and checking that each step does not expand authority. Principals are DID-based key identifiers, so two organizations can delegate to each other purely by exchanging signed capabilities, with no accounts on each other's IAM systems.

The specs differ in how the chain reaches the verifier. zCaps fix this normatively: a delegated zCap must carry its entire chain in the invocation, and a verifier must not be required to make network requests or database queries to dereference it; the root of trust is the RS, which dereferences the root zCap's controller from its own store. UCAN relies on content addressing: tokens are identified by CID, and proofs are resolvable from wherever the deployment keeps them (local store, DHT, gossip, REST), which makes chain transport a policy choice rather than a normative embedding rule. KYA-OS references the chain: the invocation carries a delegation id, the Entity Card carries a `delegationRef` locator described as "resolved, not inlined", and the protocol defines a Delegation Registry that stores the credentials and answers queries by issuer, subject, scope, and status. The chain evaluator itself takes the resolved credentials as input and is local, but getting them in hand is a resolution step against a store the verifier must trust and reach. Within a single deployment, where the verifier issues and indexes its own delegations, this is a local lookup. Across organizations it is a dependency the other two specs do not have, and it is why KYA-OS scores Partial on this row.

Verification involves several steps, some of which (like DID resolution) may need to access the network, and some (like checking a local revocation registry) can be done offline. To put it another way, chain structure verification can be performed offline; key revocation checks and other verification steps may need network access.

#### Self-revocable

All three meet the checklist requirement that someone in the delegation chain be able to revoke (modulo network partitions), and this is the criterion where their mechanics diverge most sharply. Only zCaps and UCANs let the holder revoke its own capability.

zCaps treat revocation as a resource-server state change. Any controller appearing in a delegated zCap's chain may revoke it (revoking everything downstream) by POSTing the zCap to the RS's revocation endpoint; the spec sketches a /zcaps/revocations subpath whose root zCap is controlled by all controllers in the chain, precisely so that any ancestor can revoke. Mandatory expires on every delegated zCap bounds how long a revocation must be remembered and bounds damage from a lost-but-unrevoked capability. The mechanism is out-of-band and RS-enforced: online-only, with the RS as the single enforcement point.

UCAN treats revocation as a first-class operation inside the same capability model. The Revocation sub-spec defines a Revoker role (an issuer listed in a proof chain who revokes a UCAN) and revocation is expressed as a command in the reserved /ucan namespace; executors must validate revocation state at execution time. How revocation notices propagate is left to the deployment (DHT, gossip, and similar patterns are all permitted), so there is no mandated always-online central point, at the cost of unspecified dissemination guarantees.

KYA-OS treats revocation as credential status. Each hop carries a W3C Bitstring Status List entry, the issuer flips the bit, and the verifier walks the chain root to leaf, short-circuiting on the first revoked hop, so a revoked ancestor invalidates the subtree. The walk is fail-closed: an unreachable, malformed, or unsigned status list reads as revoked. The verdict also reports whether every hop was read from a live list, and the highest conformance level requires that it was. Revocation rights are restricted to the direct issuer, any ancestor issuer, and the Responsible Party at the root; the specification states that the subject of a delegation may not revoke it, on the grounds that revocation requires authority from above. A delegable revocation permission in the UCAN style is listed as future work. The holder restriction is why KYA-OS scores Partial on this row: the checklist asks that the capability holder, somebody in the chain, or a holder of an explicit revocation capability be able to revoke, and the first and third of those are absent. The spec also allows a verifier that issues and gates its own delegations to keep revocation state locally rather than publishing a list, which makes the single-deployment case equivalent to the zCap RS model.

The shared characteristic worth stating plainly for agentic use cases: in none of the three is revocation embedded cryptographically in the token itself. It is state that the enforcement point (the RS, the executor's revocation store, or the issuer's status list) must learn about, so offline revocation semantics are unavailable in all three.

### Nice-to-haves

#### Authentication / Proof of Possession

Possession alone is never sufficient in any of the three specs; invocation requires proving control of the authorized key.

zCaps define two binding mechanisms: an HTTP Message Signature (deployments use Cavage Draft 12 today, migrating to RFC 9421\) over a Capability-Invocation header, plus Digest/Content-Digest for bodies; or a Data Integrity capabilityInvocation proof attached to a request document. Either binding mechanism requires a digital signature. Note that as with all proof-of-possession signature mechanisms, additional replay attack mechanisms are required (and used by zCaps), such as short expiration periods and unique ids tracked by the resource server/verifier. 

UCAN builds proof of possession into its envelope: every token is signed by the issuer DID's key over the canonical DAG-CBOR payload, and replay protection is layered (a required nonce, nbf/exp time bounds, and signed invocations at the transport layer). The intent is the same as zCaps' HTTP signatures; the expression lives in the UCAN envelope rather than in a separate signing convention.

KYA-OS defines a per-request proof object carried in the MCP `_meta` channel alongside the JSON-RPC call. The covered claims are the caller's DID and key id, the recipient's DID as audience, a nonce, a validity window of at most sixty seconds, and a hash of the canonicalized request body with the `_meta` member removed; a detached JWS over those claims is the proof. The verifier recomputes the request hash from what it received, checks that the key id belongs to the caller's DID document, checks nonce freshness against a store it must keep, and then asserts that the proof's DID equals the leaf delegatee of the resolved chain. That last equality is the join between possession and delegation, and it is the same check a zCap verifier performs when it matches the invocation signature to the leaf controller. Two details are specific to KYA-OS. The proof binds the JSON-RPC operation rather than an HTTP method and URL, because every MCP tool call is a POST to one endpoint and the stdio transport has no HTTP envelope at all, so the same proof verifies on both transports. And where an OAuth authorization server has issued a DPoP sender-constrained token, the proof's `cnf.jkt` thumbprint is required to match the token's, tying token possession and per-request possession to one DID key. There is nothing in zCaps for that fusion to apply to, since zCaps carry no bearer token.

One point on key custody. The Introduction suggests that proof-of-possession keys be held outside the agent, by a more deterministic component trusted to use them on its behalf. KYA-OS's trust model states that the agent process holds its own secret key by default, and lists proxy-managed and hardware-attested custody as deployment options rather than requirements. zCaps and UCANs are silent on custody, which leaves the choice to the deployment in the same way.

### Where the Specs Diverge

The evaluation above shows the three specs scoring as a family. This section highlights features that differ between them, or exist in only one. (The revocation divergence is covered under criterion 7 above.)

#### Multiple actions per capability

zCaps: a delegated zCap may carry an allowedAction that is a single string or an array of strings (for example "allowedAction": \["write", "read"\]), so one capability can directly encode "read and write but not delete", with verifiers enforcing that a child's array is no less restrictive than its parent's.

UCAN: a payload has exactly one cmd string; the spec defines no list form. To grant multiple methods, UCAN relies on either a more general command (such as /crud) whose policy and arguments restrict which operations are allowed, or multiple UCANs, each with its own cmd.

KYA-OS: `allowedAction` is always an array, with the zCap subset rule, plus the invocation-time designation rule that the caller must name which one it is exercising.

#### Resource scoping

zCaps make URL structure part of the attenuation rule: a child's invocationTarget must equal the parent's or extend it with a path or query suffix, with normative rules for the suffix forms.

UCAN has no invocationTarget equivalent; resources are governed by the Owner/Subject relationship and command semantics, and narrowing happens along the command lattice (segment structure matters: shorter commands prove longer paths in the same namespace) plus arguments, with no normative URL rules.

KYA-OS keeps the zCap `invocationTarget` field but makes it a DID and holds it constant along the chain. Sub-resource scoping is expressed only through action strings and caveats.

#### Root of authority

zCaps: the resource, via a root zCap whose id is the resource and whose controller is the resource owner.

UCAN: the resource owner, named as the Subject of the chain.

KYA-OS: the issuer of the root credential, which the protocol calls the Responsible Party and the ratified delegation profile requires to be the resource owner when one is asserted. See the accountability paragraph above for the tension between those two definitions.

#### Capability composition and authority union

UCAN defines a spec-level algebra for combining authority: the set of capabilities delegated by a UCAN is its "authority", merging capability authorities must follow set semantics (the result includes all capabilities from the inputs), and authorities can be combined in any order. This supports compositionality across resources, with no distinction between co-located and distributed resources. zCaps and KYA-OS define only single-root, single-target chains: one parentCapability per delegated capability, one invocationTarget per capability, and no defined algebra for merging multiple chains into a combined authority. (As discussed in the evaluation checklist, composability is trivially satisfied by any capability system at the model level; the asymmetry here is about spec-level machinery, not achievability.)

In UCAN's own terms the set of capabilities a delegation conveys is its authority, so a single UCAN artifact may carry more than one capability in the strict, single-resource sense of the word, while a zCap or KYA-OS credential carries exactly one.

#### Container and signature format

zCaps are JSON-LD documents (with a zcap context plus a security suite context) signed with Data Integrity proofs under the capabilityDelegation proof purpose; there is no content-addressing requirement.

UCANs are DAG-CBOR payloads inside a Varsig envelope, identified by CIDv1: native IPLD objects, with the signature scheme described by the Varsig header.

KYA-OS delegations are W3C Verifiable Credentials 2.0, typed `DelegationCredential`, carrying the zCap context and the capability as the credential subject, signed with a Data Integrity proof (the `eddsa-jcs-2022` cryptosuite) under the assertionMethod proof purpose. The capability's own proof is the wrapper's proof; there is no inner capabilityDelegation proof and no capabilityChain.

This is the divergence that most shapes each spec's natural habitat: zCaps sit on the web-standards stack (JSON-LD, Data Integrity, HTTP signatures), UCAN on the content-addressed, local-first stack (IPLD, CIDs, replicated stores), and KYA-OS on the Verifiable Credential stack (VC 2.0, status lists, DID documents, credential registries). What the VC wrapper buys and costs is the subject of the next chapter.

#### Chain structure and transport

zCaps fix the wire representation of the chain: the capabilityChain carries the root zCap by id (the verifier dereferences it from its own store) and every delegated zCap in the chain in full; verifiers must not be required to perform network requests to dereference the chain.

UCAN defines chains logically and relies on content-addressed storage and transport: proofs may be carried inline or referenced by CID and resolved from a delegation store, DHT, gossip, or REST endpoint, with no prescribed embedding pattern.

KYA-OS carries a locator. The invocation names a delegation id, the Entity Card names a chain by its hop ids joined with `>`, and the verifier resolves the credentials from a registry before evaluating them. Outbound calls to downstream services forward the leaf credential as a VC-JWT in a header, together with a short-lived proof JWT, but not the chain above it.

zCaps trade transport size for guaranteed verifier self-sufficiency; UCAN trades a resolution step for flexible, deduplicated proof storage; KYA-OS trades verifier self-sufficiency for a small invocation and a queryable registry.

#### Invocation mechanism

zCaps define invocation conventions for two carriers: an HTTP capability-invocation header (with the zCap attached compressed in a header parameter) signed with an HTTP signature, or a Data Integrity capabilityInvocation proof on an arbitrary JSON-LD document.

UCAN reuses its own envelope: the Invocation sub-spec describes an Invoker sending UCANs plus args to an Executor, which validates signatures, chains, nonce, time bounds, and revocation; transport is left open, and Data Integrity is not used.

KYA-OS carries the invocation proof inside the MCP message, under a reverse-DNS key in `_meta`, binding the JSON-RPC method and params rather than an HTTP request. An optional RFC 9421 signature over the same claims lets an HTTP edge verify the proof with a stock message-signature verifier. KYA-OS also signs the server's side: every response carries a detached JWS over the request hash and response hash, so a tool call is non-repudiable for both parties, and an audit profile builds transparency-log receipts on those proofs. Neither zCaps nor UCAN sign responses.

The specs differ on resource arguments. A UCAN invocation carries the arguments of the command in args, which policy statements can constrain. A zCap invocation has no argument structure of its own: the arguments are simply the HTTP request's query and body, covered by the request signature and digest but not by the capability's attenuation rules. KYA-OS follows the zCap pattern: the request hash covers the JSON-RPC params, and only its typed caveats (such as MaxAmount) reach into them.

#### Audience

zCaps and UCANs have no audience field separate from the resource; the target of the capability is the party expected to verify it.

KYA-OS requires a per-hop `audience` naming the verifying server on every non-root credential, and a separate `audience` on every request proof naming the recipient's DID. The per-hop rule is enforced by default with an audited opt-out for legacy integrations.

#### Obtaining the first grant

zCaps and UCANs say nothing about how an agent asks for authority it does not have; the resource owner signs a root capability and hands it over by whatever protocol the deployment uses (GNAP, VCALM, CHAPI, or similar). Some zCap deployments go further and deliberately answer an unauthorized request with a not-found status, so that a request reveals nothing about whether the resource exists.

KYA-OS defines the opposite behavior: a server that finds no sufficient delegation returns a signed `needs_authorization` challenge carrying a consent URL and a resume token, the user grants authorization at that URL, the authorization service issues the root credential, and the agent retries. The challenge is signed so that an intermediary cannot substitute the consent URL. This is an OAuth-shaped, discoverable-consent flow, and it is a philosophical split from the existence-hiding posture rather than a gap on either side.

#### Powerline (UCAN only)

UCAN's delegation work defines Powerline, a pattern for automatically delegating all future delegations to another agent regardless of Subject. Once established, it acts as a standing delegation conduit: whenever any issuer delegates along that line, authority transparently flows to the designated sink agent without being explicitly named in each delegation. It is built from normal UCAN pieces (principals, subjects, delegation rules) rather than a new primitive, and it is useful for centralizing audit, enforcement, or mediation in a specific agent, for example a storage or orchestration service that should automatically receive all capabilities granted in a namespace. The delegation spec itself flags the pattern as "very powerful. Use with care," and forbids a Powerline delegation as the root delegation to a resource. Neither zCaps nor KYA-OS has an analog.

#### Typed entities and discovery (KYA-OS only)

KYA-OS defines an Entity Card: a DID-anchored, claim-minimal identity document carrying the entity's type, display name, declared capabilities, accountability locators, and a revocation pointer, published through the DID document and projected into MCP, A2A, and NANDA discovery formats. A verifier recomputes a conformance level from the Card's referenced credentials and the live request proof, and never trusts the level the Card declares. The working group ratified exactly two pieces of this as normative primitives, the entity type vocabulary and the per-request holder-of-key binding, and treats the rest of the specification as reference implementation. Neither zCaps nor UCAN defines an identity document; both expect one from a companion layer.

### Summary

zCaps and UCAN are two realizations of the same certificate-capability model, and KYA-OS is a profile of zCaps that wraps each hop in a Verifiable Credential and surrounds the chain with an identity and audit layer. Against the evaluation checklist they score as a family: strong on the mechanics of delegated authority (chaining, monotonic attenuation, proof of possession, offline extension of the chain), and weak in the same place on policy, where all three reserve the extension point and none ships a delegation-control vocabulary. Revocation in all three is state the enforcement point must learn, not a property of the token.

KYA-OS moves two verdicts in opposite directions. It is the only surveyed specification that scores Yes on accountability, because it names and recomputes a Responsible Party and types every principal, which is exactly the companion identity layer the other two chapters say the family needs. It pays for that with a Partial on local verifiability, because the chain is resolved from a registry rather than carried, and a Partial on self-revocation, because the holder of a delegation cannot revoke it. Whether those costs are inherent to the identity layer or only to the choice of carrying it in Verifiable Credentials is the question the next chapter takes up.

The differences among the three are real but sit a level down, in encoding and ecosystem rather than capability model: JSON-LD and Data Integrity on the web-standards stack, DAG-CBOR and CIDs on the content-addressed local-first stack, VC 2.0 and status lists on the credential stack; normative chain embedding versus resolvable proof stores versus a registry locator; URL narrowing versus a command lattice versus a fixed target; an RS revocation endpoint versus a revocation command versus a status list. For delegated authorization for AI agents, choosing among them is largely a deployment-ecosystem decision; the properties that matter for the agentic checklist, and the work still needed, are common to all three.

## Modeling Chained Delegation with W3C VCs

### Why this chapter exists

Two of the systems examined in this survey carry delegated authority inside a Verifiable Credential: KYA-OS, whose delegation credential is a W3C VC 2.0 wrapping a ZCAP-LD capability, and dSD-JWT, which appends delegation hops to an SD-JWT VC as disclosures. Neither is an accident. The VC toolchain is the most widely deployed signed-document stack in the decentralized identity community, it already has wallets, issuers, status lists, and verifier libraries, and a working group that needs delegation is tempted to reuse it. This chapter sets out why that is the wrong default, and then, since the temptation will win in some projects, what a VC-based delegation design has to do to reach the properties the certificate capability chapter scores.

The short version: a Verifiable Credential is a claim, and a delegation is a permission. The VC data model and its verifier semantics are built to answer "did this issuer assert this about this subject?" A delegation verifier has to answer "does the presenter hold authority over this resource, for this action, by an unbroken and only-narrowing chain from the resource owner?" Everything that goes wrong below follows from forcing the first machinery to answer the second question.

### The case against

Four things are lost when a capability is carried as a credential, and two more have to be rebuilt.

#### Proof purpose

A zCap delegation is signed under the `capabilityDelegation` proof purpose, and a zCap invocation under `capabilityInvocation`. A DID controller can therefore list some keys as authorized to delegate, others only to invoke, and others only to make assertions, and a verifier checks that the signing key was published for the purpose it is being used for. A Verifiable Credential is signed under `assertionMethod`. The moment a delegation is a VC, any key the controller has published for making claims can delegate authority, and the purpose separation is gone. KYA-OS's delegation credentials are signed under assertionMethod, and the specification has no replacement for the lost check.

#### The subject slot

VC tooling assumes `credentialSubject.id` identifies the party the credential is about, and holder binding in presentation protocols compares that id to the presenter's key or DID. A capability's subject is the capability itself, and the party who may exercise it is one of its properties. KYA-OS resolves the conflict by putting the capability's URN in `credentialSubject.id` and the delegatee in an `invoker` property inside it. That is the correct choice for the capability, and it means stock holder-binding checks give the wrong answer and must be replaced with a custom join between the leaf invoker and the invocation key. A design that instead puts the delegatee's DID in the subject id, as KYA-OS's older profile did, gets holder binding for free and loses the capability's own identity, which the chain needs for `parentCapability` links.

#### Claims and permissions look alike

A generic VC verifier accepts any credential from a trusted issuer whose proof verifies. Nothing in the data model distinguishes a credential that says "this agent is employed by Acme" from one that says "this agent may transfer funds", and a verifier that reads claims as authority is a confused deputy in the making. KYA-OS closes this with two rules: the `type` array must include `DelegationCredential`, and the credential subject must carry nothing beyond the capability's own properties, with a verifier required to reject any credential whose subject carries an extra claim. Those rules are necessary and they are a patch, replacing a distinction that zCaps obtain from proof purpose and from not being credentials.

#### Root of authority

A VC verifier trusts issuers. The question it asks of the root is "is this issuer on my allowlist?", which is identity-based trust in a trusted-issuer registry. A zCap verifier asks "is this the root capability for the resource I am, controlled by my owner?", which it answers from its own state. KYA-OS inherits the issuer model, defines a trust registry, and then has to argue that the registry is advisory and the cryptographic chain is the real gate. The argument is correct, but the shape of the problem comes from the carrier: a credential has an issuer, a capability has an owner, and the two are not the same relationship.

#### Nothing in the model for a chain

A VC has issuer, subject, validity, status, and proof. It has no parent, no chain, and no attenuation rule, and a Verifiable Presentation is a bag of credentials with no ordering. The chain has to be modeled in the subject payload (which KYA-OS does by reusing `parentCapability` from zCaps) and the chain walk has to be written from scratch (which KYA-OS does in its own chain evaluator). The next section covers why no VC library does this for you.

#### Revocation is per credential, per issuer

Status lists are published by issuers, one bit per credential. A chain has one issuer per hop, so cascading revocation of a subtree means the verifier walks every hop and consults every hop's issuer's list, with a policy for what an unreachable list means. KYA-OS specifies that walk and makes it fail closed. The result is sound but it is a verifier the VC stack did not provide, and its cost grows with chain depth and with the number of distinct issuers.

### If you do it anyway

The practices below are what the two surveyed VC-based designs did, or should have done, to recover the certificate capability properties. KYA-OS is the running example because it goes furthest; dSD-JWT is cited where it diverges.

#### Subtype the credential and fix the subject shape

Require a type such as `DelegationCredential` and reject any credential that lacks it. Define the exact set of properties the subject may carry and reject any extra. A subtype alone is not enough, because a verifier that merges claims from the subject will still read an extra property as authority. KYA-OS does both, and its schema is published so that the shape rule is machine-checkable. dSD-JWT does neither: the delegate payload is arbitrary JSON and the only validation rejects reserved keys.

#### Put target and actions in the subject, and reuse the zCap vocabulary

Designation must be explicit in the artifact or the design is not a capability. Use `invocationTarget` and `allowedAction` with the zCap context rather than a new vocabulary, so that the semantics and the subset rules are already specified. KYA-OS includes the zCap context and uses both fields. dSD-JWT defines neither and defers them to a profile, which is why it scores Partial on confused deputy and No on attenuation.

Decide early whether the target may narrow along the chain. zCaps allow URL-suffix narrowing. KYA-OS holds the target constant and makes it a DID, which simplifies the continuity check and gives up sub-resource attenuation; a delegator holding authority over a server cannot hand down authority over one path of it except through action naming.

#### Define a caveat vocabulary and a rule for unknown caveats

Typed caveat objects, each with a stated monotone comparison (a date no later than the parent's, a limit no greater than the parent's in the same currency), are the attenuation axis the family has reserved and left empty. A VC-based design should ship at least the caveats it needs and, more importantly, say what a verifier does with a caveat type it does not understand. KYA-OS's answer is the right one: the child must carry every parent caveat verbatim, and a verifier that cannot evaluate a caveat rejects rather than ignores, so re-delegating through an ignorant verifier cannot shed a restriction.

#### State the attenuation invariants and check every hop

The invariants are the zCap ones restated at the credential layer, and a verifier has to check all of them at every link, not only at the leaf:

* actions: the child's `allowedAction` is a subset of the parent's;
* validity: the child's `validUntil` is no later than the parent's;
* caveats: each parent caveat is present in the child, at least as narrow;
* continuity: the parent's delegatee is the child's issuer, the child's `parentCapability` is the parent's id, and the root's `parentCapability` is the target;
* target: constant, or narrowed only by the rule the design has chosen.

Two VC-specific pitfalls deserve a line each. Expiry now lives in two places, the credential's `validUntil` and any `ValidUntil` caveat, and a design must say which governs when they disagree; KYA-OS requires both to narrow and checks both. And a VC's `issuer` is the delegator while the capability's `invoker` is the delegatee, so the continuity check crosses the wrapper boundary (outer issuer of hop n+1 equals inner invoker of hop n), which is easy to get wrong when the inner and outer layers are parsed by different code.

#### Carry the chain, or say what the verifier depends on

A VC has no chain slot, so the design must choose between inlining the chain (an ordered presentation, or the immediate parent embedded as zCaps require) and referencing it by locator. Inlining preserves the verifier's self-sufficiency, which is the property the certificate family is scored on. Referencing makes the verifier depend on a registry or store. KYA-OS references, and scores Partial on local verifiability for it. dSD-JWT inlines by construction, since every hop is a disclosure appended to the same token.

#### Treat invocation as a separate proof and specify the join

A credential is not an invocation. Presenting one, even with VC-style holder binding, proves that the presenter controls the subject key at presentation time, not that this request is the one being authorized. The design needs a per-request proof of possession over the request itself (zCaps use HTTP signatures or a `capabilityInvocation` proof; KYA-OS uses a detached JWS over the canonicalized JSON-RPC call; dSD-JWT uses a Key Binding JWT with audience and nonce), and it needs a stated join: the key that signed the request equals the delegatee named by the leaf hop. KYA-OS writes that equality down and recomputes it on every verification; the recomputation, not the credential, is what prevents a delegation to key A being exercised by key B.

#### Use a real cryptosuite

Sign with a standard Data Integrity cryptosuite or a standard JOSE or COSE envelope, so that at least the per-hop signature check can be delegated to an existing library and so that the artifact is verifiable by a second implementation. The chain walk will be custom; the signature should not be. KYA-OS's ratified profile uses `eddsa-jcs-2022`. Its older profile labeled a non-standard construction with a standard suite name, which is the failure mode this practice guards against: a credential a W3C verifier appears to recognize and then cannot verify.

### Why current VC libraries cannot verify a delegation chain

The practices above add up to a verifier that no VC library ships, and it is worth being precise about what is missing, because the gap is routinely underestimated by projects that start from a VC stack.

A VC library verifies one credential. It checks the proof against the issuer's key, the validity window, the status list entry, and optionally that the issuer is on an allowlist and that the credential matches a schema. A presentation verifier does the same for each credential in a presentation, checks the presentation's own proof, and, if holder binding is in use, checks that the subject id matches the presenter. Every one of those checks is per credential and none of them relates two credentials to each other.

A delegation chain verifier additionally has to:

* order the credentials into a chain and reject cycles, forks, and gaps;
* check continuity across the wrapper boundary at every link;
* check the attenuation invariants at every link, including caveat comparison by type;
* hold the target constant, or apply a narrowing rule;
* walk every hop's status entry against its own issuer's list, with a fail-closed policy and a freshness verdict;
* resolve or receive the chain, and decide whether to trust the source;
* verify a separate per-request proof and join its key to the leaf delegatee.

KYA-OS's reference implementation is the clearest evidence of where the stack ends. Its chain evaluator takes already-parsed credentials and performs the structural and revocation walk, and its specification states that per-hop signature verification is "a separate injected concern". The VC library is the injected concern. Everything that makes the artifact a delegation rather than a claim is the new code. dSD-JWT makes the same point from the other side: its SDK delegates chain verification to a separate crate, and the draft specifies the chaining mechanics while leaving every semantic check to a profile that does not yet exist.

The practical consequence for a working group is that a VC-based delegation profile does not inherit a verifier from the VC ecosystem. It inherits signature verification, status lists, and a serialization. The chain semantics, which are the entire content of the certificate capability chapter's criteria 2, 4, 6, and 7, have to be specified and implemented as if the credential layer were not there. A profile that does that work carefully, as KYA-OS largely has, arrives at a zCap verifier with a credential parser in front of it. A profile that does not, as dSD-JWT currently does not, arrives at a chain of signatures with no authority semantics.

### How the two surveyed attempts fare

| Practice | KYA-OS (VC 2.0 plus ZCAP-LD) | dSD-JWT |
| :---- | :---- | :---- |
| Subtype and fixed subject shape | Yes: `DelegationCredential` required; extra subject properties rejected | No: arbitrary payload, reserved keys only |
| Target and actions in the artifact | Yes: `invocationTarget`, `allowedAction`, zCap context | No: deferred to a profile |
| Caveat vocabulary and unknown-caveat rule | Yes: `ValidUntil`, `MaxAmount`; unknown caveats copied verbatim | No |
| Attenuation invariants checked per hop | Yes: actions, validity, caveats, continuity, constant target | No: no narrowing rule |
| Chain carried with the invocation | No: referenced by locator, resolved from a registry | Yes: hops appended to the token |
| Per-request proof with a stated join | Yes: detached JWS in `_meta`; leaf invoker must equal the proof key | Yes: Key Binding JWT per hop with audience and nonce |
| Standard cryptosuite | Yes: `eddsa-jcs-2022` (ratified profile) | Yes: JOSE |
| Per-hop revocation | Yes: Bitstring Status List per hop, cascading, fail-closed | No: status list on the base credential only |

Read together with the certificate capability scorecard, the table shows the trade in plain terms. Doing the work recovers the family's verdicts on confused deputy, chainability, and attenuation, and buys the accountability semantics the family lacks, at the cost of the self-sufficiency and the proof-purpose discipline a native capability has. Not doing the work leaves a chaining mechanism without a capability model. For a group that controls both ends of the deployment and already runs a VC stack, the first trade can be reasonable. For a cross-organizational agentic deployment, where the verifier does not share a registry with the issuer and the chain is the only thing it can trust, the native capability formats are the better starting point and the VC layer belongs at the root, binding the operator to the first delegation, as the composed stack in the Synthesis chapter describes.

## Cedar/Janssen Evaluation

### Overview

[Cedar](https://www.cedarpolicy.com/en) is an open-source authorization policy [language](https://docs.jans.io/stable/cedar-intro/?h=cedar) and evaluation engine created by IAM specialists at Amazon with formal methods training (used in AWS Verified Permissions; the language syntax itself is now a CNCF [candidate project](https://www.cncf.io/projects/cedar/)). A Cedar policy answers "**may** this principal **perform** this action **on** this resource **under** these conditions (context)" using `permit` and `forbid` statements validated against a typed schema. Evaluation is deterministic, default-deny, and `forbid` always overrides `permit`.  Over time, the core Cedar [language](https://docs.jans.io/stable/cedar-intro/?h=cedar) and its canonical reference implementation in Rust have been generalized from Amazon’s production implementation in Keycloak/AWS contexts to be hardened and governed at CNCF within the Linux Foundation (as mentioned above); a cloud-agnostic IAM framework generalizing the Keycloak/AWS tooling is governed in a dedicated Linux Foundation Project called [Janssen](https://docs.jans.io/stable/getting-started/), which also governs and develops the younger “Cedarling,” a lightweight version of Cedar’s enforcement engine that can be run “practically anywhere” as a WASM component inside of not just servers but even clients and agents/harnesses.


**Cedar has no native, enforced delegation primitive.** It is a policy expression language plus a Policy Decision Point (PDP), with no native concept of a credential, a holder, a signature, or a delegation chain. It is *identity-based* in the classic sense: the verifier decides access by evaluating who the principal is (and their attributes and group memberships, as made known to the verifier by tokens from server-side tooling like Janssen) against policies the verifier holds, in contrast to possession-based models like zCaps, where presenting a valid signed capability constitutes the “just-in-time” proof of authority. This identity-vs-possession distinction drives nearly every row of the scorecard.

Cedar is therefore evaluated here **as a component in a delegation stack**: Cedar supplies policy expression and local evaluation in cross-organization use cases where policy must be made intelligible to callers and counterparties across policy boundaries; the delegation mechanics (chain construction, portable proof, holder-side revocation) must be supplied by a carrier layer around it, but the policy anchoring the semantics of those delegations and error messages can be more meaningfully communicated by the combination of the layers.

Three reference points are used:

* the [Cedar language and documentation](https://docs.cedarpolicy.com/) (the "spec": syntax, schema, validation, templates, evaluation semantics),  
* the [Janssen Project Cedarling](https://docs.jans.io/stable/cedarling/), the most relevant deployed embedding: a lightweight, embeddable PDP (Rust core with WASM, mobile, Python and Java bindings) that implements Token-Based Access Control (TBAC), deriving Cedar principal entities from JWTs issued by trusted issuers and evaluating policies locally at the edge, and  
* [Clawdrey Hepburn](https://clawdrey.com/), a first-person agentic case study: an autonomous AI agent whose operator-defined action boundaries (which accounts it may follow, what it may do with its own credentials and payment card) are enforced by an embedded Cedar engine checked before every action, with a typed schema modeling the agent, its operator, and its action space.


Where Cedar (the language) and runtime-evaluation deployments like Cedarling differ in what they provide, this is called out, since several checklist items are satisfiable only by the deployment layer.

### Scorecard

Each entry gives the requirement, the verdict, and the reasoning.

**1. Attenuated.** Verdict: Partial.

Forbid-overrides-permit gives monotonic restriction: layering additional forbid policies (or a stricter verifier-side policy set) can only shrink authority, never expand it. But Cedar has no runtime check that a delegatee's grant is a subset of the delegator's; subset/equivalence can be proven offline with Cedar's symbolic analysis tooling. Attenuation enforcement is a stack responsibility.

**2. Accountable (agent vs. principal/operator).** Verdict: Yes (at the modeling level).

The schema can type the distinction directly (e.g. `entity Agent in [User]`); Cedarling derives separate User and Workload principals from the `id_token` and `access_token` and can require both to be authorized. But the distinction is data the verifier trusts an Authorization Server and token exchange to provide, not a cryptographically attributable delegation edge.

**3. Resistant to confused deputy.** Verdict: No.

Identity-based by design: the deputy exercises its own standing permissions against a caller-designated resource, and the model does not combine designation with authorization. Policies can condition on request context to mitigate specific deputy scenarios, but that is mitigation, not structural immunity.

**4. Represent authorization policies (embed/link; functional restrictions, re-delegation rules, advisory "please").** Verdict: Yes, except advisory.

This is Cedar's core competence. "Do not delegate further" and re-delegation rules are expressible if delegation itself is modeled as an action. No advisory tier: decisions are binary permit/forbid, with no obligations or native "honor-if-possible" semantics.

**5. Chainable.** Verdict: No (stack-dependent).

No delegation primitive, no chain artifact. Delegation can be modeled as verifier-side **state** (entities or instantiated policy templates), but chains are not portable, signed, or first-class. In a delegation stack, the chain must live in the carrier or elsewhere in the Authorization stack; Cedar evaluates each hop atomically.

**6. Cross-organizational / locally verifiable.** Verdict: Partial.

Evaluation is fully local (Cedarling runs in browsers, mobile, gateways; multi-issuer JWTs supported without shared IAM accounts). But the policy store is unilateral; it belongs to the verifier. There is no standard or guidance for exchanging policy semantics across organizations in an ad hoc way, and trusted issuers must be pre-configured. Core contributors do work to keep Janssen [in lockstep with AuthZEN](https://github.com/JanssenProject/jans/pull/13077) for federation and runtime issuer-list features.

**7. Self-revocable (holder or somebody in chain).** Verdict: Partial.

Revocation is a policy store change (remove a permit or add a forbid), immediate at the next evaluation and very fine-grained. But it is performed by whoever administers the policy store, not natively by a delegator or delegatee in the chain; mapping chain participants to revocation rights is a stack design task. Cascade to downstream grants is not native either: it holds only if delegation is modeled as linked entities whose policy re-checks the ancestry.

**8. Authentication / proof of possession (nice-to-have).** Verdict: No (out of scope).

Cedar does not authenticate. Cedarling validates JWT signatures (proof of issuance by a trusted issuer), which is not holder proof of possession; PoP binding is left to the token layer.

### Cedar Evaluation

#### Attenuated

Cedar has one structural property that maps beautifully onto attenuation, and one missing property that prevents a full Yes.

The structural property is **monotonic restriction by composition**: `forbid` overrides `permit`, and evaluation is over the union of all policies. Adding policies to a set can therefore only ever *shrink* what a forbid touches and a verifier (or any later layer) can stack stricter policy on top of a base grant with a guarantee that the stack cannot widen authority. This is a natural fit for the delegation-stack pattern in which each hop attaches further restrictions: express the delegator's grant as the base policy set and each subsequent restriction as additional forbids or narrower permits evaluated together.

What is missing is **enforced subset semantics between grants**. If a delegation is represented as "delegatee gets policy set B, derived from delegator's policy set A", nothing in the Cedar runtime checks B ⊆ A; an over-broad B is just another valid policy set. The check is possible, and this is where Cedar is unique: its analyzability means "B permits nothing A does not" is a decidable question answerable by symbolic analysis tooling at delegation time or audit time. This is also what makes Cedar a candidate tool for checking that one set of caveats is a subset of another, whether or not Cedar is the runtime evaluator. But it is an offline/toolchain guarantee the stack must invoke, not a property the evaluator enforces per request the way a zCaps verifier enforces allowedAction narrowing.

Expressiveness of attenuation, once enforced, is far richer than URL-suffix narrowing: any condition the language can state (attribute ranges, context conditions, time windows, resource groups) can be the attenuation axis. The same expressiveness is a hazard. The more axes a condition can range over, the easier it is to write a restriction that does not actually restrict, which is why the subset check matters.

#### Accountable, agent vs. principal/operator

Differentiation is a schema design choice, made trivially. Cedar entities are typed and hierarchical, so the agent/operator relationship can be modeled directly, for example `entity Agent in [User] { model: String, runtime: String, trust_level: String }`, which is exactly the pattern the Clawdrey case study uses: the agent is a typed principal nested under its human operator, and policies can condition on either or both. Cedarling operationalizes the same split out of the box: it constructs a **User** principal from the OIDC `id_token` and a separate **Workload** principal from the OAuth access token (the software acting on behalf of the person, or autonomously), and its default decision logic can require that *both* the person and the workload be authorized for the request to proceed. That is precisely the agent-vs-operator distinction the checklist asks for, expressed natively in the data model rather than by convention.

What Cedar does not provide is cryptographic accountability for the *delegation* itself. There is no signed delegation edge: the "fact" that operator O stands behind agent A is an entity attribute or a token claim that the verifier chose to trust. Attribution quality is therefore exactly the attribution quality of the surrounding identity layer (the JWT issuer, in Cedarling's case). An auditor can read the decision logs (Cedarling logs every decision with diagnostics) and reconstruct *which policy permitted which principal*, but cannot derive a non-repudiable "this delegator authorized this delegatee" proof from the Cedar artifacts alone. The accountable party for any grant is, structurally, the policy store administrator.

#### Resistant to confused deputy

Evaluated as-is: no, and the exposure follows directly from the identity-based model. The verifier decides access by evaluating who the principal is against verifier-held policies, so a deputy service acting on a caller-designated resource exercises its own standing permissions; nothing in the model binds the caller's designation to the authority being used. Policies conditioned on request context can mitigate specific deputy scenarios, but that is per-policy mitigation, not the structural designation-plus-authorization coupling that capability systems provide. In a delegation stack, immunity would have to come from the carrier layer, with Cedar evaluating the rules at each hop.

#### Ability to represent authorization policies

Policies are first-class, schema-validated, and arbitrarily conditional on principal, action, resource and request context: attribute comparisons, group membership (`in`), set operations, string and IP functions, and explicit `forbid` statements that override any permit. Policies can be embedded with the verifier or distributed to it (Cedarling fetches signed policy stores over HTTPS or from a Lock Server), satisfying the embed-or-link condition.

Against the commonly requested policy examples:

* **"Do not delegate further" / "this permission is undelegatable":** expressible, with one precondition: delegation must be modeled as an action in the schema (e.g. `action delegate appliesTo { principal: [Agent], resource: [Permission] }`). Then `forbid (principal, action == Action::"delegate", resource == Permission::"x");` makes permission x undelegatable, and since forbid overrides permit, no later grant can reopen it. Depth limits are expressible the same way if depth is carried as an attribute. The precondition matters: Cedar can only forbid delegations that the stack routes through a Cedar check.  
* **Re-delegation policies:** same mechanism, with the full expressiveness of the language (delegate only to principals in a given group, only with attenuated scopes, only before an expiry, and so on).  
* **Advisory "please" rules (a request carrying desired re-delegation rules whose violation is not refused):** not supported. Cedar evaluation returns a binary Allow/Deny plus diagnostics; there is no obligations or advice tier in the decision (a notable contrast with XACML), and policy annotations like `@id(...)` are non-semantic metadata. An advisory tier would have to be built beside Cedar, not in it.  
* **"Who gets to know what is a governance topic":** Cedar is agnostic; it evaluates whatever entity data it is handed. Disclosure governance sits in the surrounding stack (which claims go into tokens, which attributes into the policy store).


One genuinely distinctive property deserves mention here: Cedar is **analyzable**. The language was deliberately kept non-Turing-complete (no loops, no recursion, guaranteed termination) and its evaluator is formally verified, so policy sets can be subjected to automated reasoning: does policy set A permit anything B does not, are these two sets equivalent, is this forbid ever reachable. For a delegation context, that means delegation rules can be *audited and proven* offline, which no other technology in this survey offers.

#### Chainable

Evaluated as-is: no. Cedar has no delegation primitive. There is no parent pointer, no chain, no signed artifact that one party hands to another. Two in-language mechanisms get part of the way and are worth naming precisely:

* **Policy templates.** A template with `?principal` / `?resource` slots can be instantiated at runtime to grant a specific principal access to a specific resource. Template instantiation *is* a grant operation and can be wired up as the delegation act ("delegator triggers instantiation of a template scoped to the delegatee"). But instantiation is an administrative write to the policy store, performed by whoever holds write access, and the resulting policy records no ancestry.  
* **Delegation modeled as data.** The schema can define delegation entities (delegator, delegatee, scope, expiry, parent reference) and policies that grant access when a valid delegation entity exists. This can represent chains of arbitrary depth, including agent-to-subagent links. But the chain is then application state in the verifier's entity store: not portable, not signed, not independently verifiable, and entirely a bespoke convention.


So in a delegation stack, the chain must live in the carrier layer (signed tokens or credentials linked hop to hop), and Cedar's role is to evaluate the rules *at* each hop: may this delegator delegate this, may this delegatee exercise it. Cedar is the rulebook for the chain, not the chain.

#### Cross-organizational / locally verifiable

Half of this requirement is Cedarling's headline feature; the other half is structurally missing.

**Locally verifiable: yes.** Cedar evaluation requires no network call: policies, schema and entity data are in memory, and decisions are sub-millisecond (for simple policies only). Cedarling pushes this to the edge deliberately: a WASM/mobile/server-embeddable PDP that loads a policy store at startup and answers authorization questions locally, so an API gateway, browser or device enforces policy without phoning a central authorization server per request. JWT signature validation against trusted issuers is also local (cached JWKS).

Distributing the policy store to the edge has a privacy cost. A Cedarling instance holds the whole store, including policies about other principals and whatever entity data is bundled with it. Anyone able to read the store (a browser user, a device owner, a compromised gateway) can learn who may do what. The exposure is comparable to the discovery risk noted for AuthZEN Search in the next chapter, and it argues for scoping each edge store to the principals and resources that edge actually serves.

**Cross-organizational: partial.** Cedarling's TBAC model is genuinely multi-party at the *evidence* level: `authorize_multi_issuer` accepts JWTs from several different trusted issuers in one request and maps their claims to Cedar entities, so organization A's verifier can consume identity evidence issued by organization B without B having an account in A's IAM. But the *policy* level is unilateral: the policy store (schema, policies, trusted issuer list) belongs to the verifier alone. There is no standard for one organization to hand another a policy fragment with agreed semantics, no shared schema federation, and the trusted issuer list is pre-configuration, not dynamic trust establishment. Two organizations can interoperate, but only after bilateral setup; the checklist's "express authorization policies without accounts on each other's IAM systems" is met for authentication evidence and unmet for policy exchange.

#### Self-revocable

Revocation in Cedar terms is straightforward and immediate: delete the permit (or the template instantiation, or the delegation entity) or add a forbid, and the very next evaluation reflects it. Forbid-overrides-permit makes "revoke" particularly clean: a targeted forbid kills a grant without having to find and remove every permit that contributed to it. Granularity is excellent: one delegatee, one action, one resource, one condition.

The checklist question, however, is *who* can revoke. Natively: whoever can write to the policy store. That is an administrative role, not a chain position. A delegator can revoke what they delegated only if the stack maps "delegator of grant X" to "may modify/forbid grant X in the store", which Cedar can itself express as policy (delegation management actions authorized by Cedar policies, the recursive pattern) but does not provide out of the box. Propagation is also a deployment property: Cedarling instances cache their policy store and pick up changes on refresh or via Lock Server push, so revocation latency equals policy distribution latency. Within a single trust domain this is near-immediate; across many edge PDPs it is eventually consistent.

On **cascade** (does revoking a grant revoke everything delegated below it): Cedar has no built-in notion of a chain, so there is no built-in cascade either. Whether revocation propagates downstream depends on how delegation is modeled. If each grant is an independent policy, revoking Alice-to-Bob leaves Bob-to-Carol untouched, and Carol keeps her access until that grant is separately removed. Cascade is achievable by modeling delegation as data with parent links plus a permit policy that succeeds only when the whole ancestry is still valid: each delegation is an entity carrying a parent reference, and Carol's evaluation walks up the chain, so revoking Alice-to-Bob cascades to Bob-to-Carol because the walk hits the now-missing or now-forbidden link and fails closed. This is a contrast with zCaps, where cascade is structural (a delegated capability embeds its parent, so rejecting any link fails everything below it by construction); in Cedar cascade is a property you build into the policy and entity model, not one the engine provides.

Net: revocation is a strength of the centralized-policy model (no token to chase, no expiry to wait out, unlike credential-based systems where an issued artifact is "out there"), but holder/chain self-revocation is a stack construct, not a Cedar feature.

#### Composability (spec-level note)

A clear yes. Cedar evaluates every request against the union of all loaded policies; an Allow needs at least one satisfied permit and no satisfied forbid, and the satisfied permit(s) can originate from any source loaded into the store. Principals can belong to arbitrarily many groups and roles simultaneously (the checklist's RBAC single-role counter-example does not apply: Cedar's `in` hierarchy is a DAG, not a single assignment). Cedarling extends composition to the evidence layer: `authorize_multi_issuer` lets one request carry tokens from multiple independent issuers, each mapped to entities, so authority deriving from several principals or organizations is combined in a single decision.

The same property underlies the confused deputy exposure recorded above. Because a decision draws on every permit the principal holds anywhere in the store, a deputy's own standing permissions are always in scope, including for a request it did not originate.

The one caveat mirrors row 6: all the composed sources must already be present in (or mapped into) the verifier's policy store and trusted issuer configuration by stable identifiers. Composition is native at decision time, but onboarding a new source of authority is configuration.

### Nice-to-haves

#### Authentication / proof of possession

Out of scope for Cedar by design: the engine assumes the principal has already been authenticated and evaluates the request it is handed. Cedarling adds issuance-side verification: JWT signature validation against pre-configured trusted issuers, aud/sub cross-checks between tokens, optional status checks, and exp/nbf checks. This proves the tokens were issued by a trusted party and are current, not that the presenter is the rightful holder. Sender-constrained tokens (DPoP/mTLS-bound) can be layered in by the token infrastructure, but Cedar/Cedarling does not itself verify possession.

### Summary

Cedar is, almost row by row, the complement of the capability-based technologies in this survey. It is strongest exactly where zCaps are weakest: rich, analyzable, schema-validated policy expression (including re-delegation rules), native agent-vs-operator modeling, composition of authority from multiple sources in one decision, fine-grained immediate revocation, and fully local evaluation at the edge. It is weakest exactly where zCaps are strongest: there is no delegation chain, no portable signed grant, no holder-side revocation, no offline delegation, and no cryptographic attribution of who authorized whom; all of these are verifier-side state administered through a policy store. One further caution applies to any attribute-based policy, Cedar included. When a principal's attribute changes, the set of permissions that change with it is not visible from the attribute itself, so a routine attribute update can silently grant or remove access. Cedar's analyzability makes such effects discoverable, but only if someone runs the analysis.

For delegated authorization for AI agents, the implication is that Cedar is not a candidate *delegation data model* but a strong candidate component beside one, in two distinct roles. The first is to express the caveats carried in a capability chain and to guarantee, by offline analysis, that no delegation relaxes the caveats of its parent. The second is to decide, at the root of a chain, which capabilities a resource owner should hand to which authenticated principal. In both roles a carrier technology supplies the chain, proof and portability, while Cedar supplies the rulebook, with the unusual bonus that those rules can be formally analyzed. The Cedarling deployment shows the integration pattern already in production form (policies travel to the edge, identity evidence travels as multi-issuer JWTs, decisions stay local), and the Clawdrey case study shows the agentic pattern (operator-authored Cedar policies as the hard boundary on an autonomous agent's action space). What no current Cedar deployment shows is the missing piece the checklist cares most about: a standardized way for a delegator to hand a delegatee a portable, attenuated, chain-verifiable grant.

### References

#### Cedar language and engine

* [Cedar project site](https://www.cedarpolicy.com/en) — positioning, project overview, CNCF status.  
* [Cedar documentation](https://docs.cedarpolicy.com/) — primary language reference.  
  * [Basic Cedar syntax](https://docs.cedarpolicy.com/policies/syntax-policy.html) — permit/forbid effects, scope, when/unless conditions, and the forbid-overrides-permit / default-deny combining rules.  
  * [Terms & concepts](https://docs.cedarpolicy.com/overview/terminology.html) — entities, typed UIDs, the entity hierarchy, static vs. template-linked policies, RBAC/ABAC/ReBAC framing.  
  * [Policy templates](https://docs.cedarpolicy.com/policies/templates.html) and [Roles with policy templates](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles-templates.html) — `?principal` / `?resource` slots, template-linked policies as a runtime grant mechanism, and their coupling to user life-cycle management (the "delegation as template instantiation" argument).  
* [Cedar: A New Language for Expressive, Fast, Safe, and Analyzable Authorization (extended version, arXiv 2403.04651)](https://arxiv.org/pdf/2403.04651) — the design paper; basis for the non-Turing-complete / formally-verified / analyzable claims and the two-slot template restriction.  
* [cedar-policy/cedar (GitHub)](https://github.com/cedar-policy/cedar) and [cedar_policy crate (docs.rs)](https://docs.rs/cedar-policy) — reference implementation; entity store model, local in-process evaluation, validator/typechecking.


#### Janssen Project Cedarling (the deployed embedding)

* [Cedarling overview / getting started](https://docs.jans.io/stable/cedarling/) — embeddable PDP on the Rust Cedar engine, WASM/mobile/server bindings, Token-Based Access Control, local edge evaluation.  
* [Authorization using Cedarling](https://docs.jans.io/v1.15.0/cedarling/reference/cedarling-authz/) — derivation of separate User (`id_token`) and Workload (`access_token`) principals, the "person AND workload must be authorized" decision logic, JWT validation steps.  
* [Cedarling getting started — multi-issuer authorization](https://docs.jans.io/head/cedarling/tutorials/cedarling-getting-started/) — `authorize_multi_issuer` and `authorize_unsigned` methods; multiple trusted issuers in one request.  
* [Policy Store](https://docs.jans.io/head/cedarling/cedarling-policy-store/) — schema \+ policies \+ trusted issuers bundle, claim mapping, role mapping; loaded as static JSON or fetched over HTTPS / from a Lock Server.

#### Agentic case study

* [Clawdrey Hepburn](https://clawdrey.com/) — autonomous AI agent using an embedded Cedar engine as its action-space boundary.  
  * [Why I Built a Policy Engine Before I Built a Personality](https://clawdrey.com/blog/why-i-built-a-policy-engine-before-i-built-a-personality.html) — per-action Cedar checks as enforced (not advisory) boundaries; the "follow only accounts Sarah follows" example.  
  * [Modeling an Agent's World in Cedar](https://clawdrey.com/blog/modeling-an-agents-world-in-cedar.html) — typed schema modeling the agent, operator, and action space; agent nested as a principal under its human operator.

## AuthZEN Evaluation

### Overview

This chapter evaluates AuthZEN against the seven mandatory criteria and the three additional considerations defined in the Overview chapter.

For the purposes of this assessment, we distinguish between the AuthZEN Core Specification and the broader AuthZEN ecosystem.

The term Core Specification refers to the [AuthZEN Authorization API 1.0](https://openid.net/specs/authorization-api-1_0.html) (hereafter referred to as "Core"), which defines the normative interoperability model between a Policy Enforcement Point (PEP) and a Policy Decision Point (PDP). As the primary standards-track specification and the foundation for runtime authorization interoperability, it serves as the baseline for evaluating AuthZEN against each criterion. Its central data model is a runtime authorization request expressed in terms of Subject, Action, Resource, and Context (SARC). The Core defines [single Access Evaluation](https://openid.net/specs/authorization-api-1_0.html#access-evaluation-api) and [multiple Access Evaluations](https://openid.net/specs/authorization-api-1_0.html#access-evaluations-api) endpoints together with [Search](https://openid.net/specs/authorization-api-1_0.html#name-search-apis) operations for discovering permissible resources, subjects, or actions. At the same time, it explicitly excludes policy language, PDP architecture, policy state management, and API authentication from its scope. Accordingly, this chapter distinguishes functionalities that are intrinsic to the AuthZEN protocol from functionalities supplied by the PDP, policy engine, identity system, or another surrounding layer.

The broader AuthZEN ecosystem currently includes three complementary specifications:

* [AuthZEN Policy Store Format](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html) (hereafter referred to as "Policy Store") addresses the policy-management side of the architecture. It defines a portable, PDP-neutral package (a bundle of policy files in a directory structure) that co-locates policies, schema, default entities, trusted token issuers, and supporting metadata, together with a Policy Store API for managing the package. Its purpose is portability, versioning, auditability, and interoperability of policy artifacts across PDPs and tooling. It standardizes the packaging and management format rather than the semantics of any particular policy language or engine.  
* [COAZ: A Framework for Mapping Information Models to AuthZEN Authorization Requests](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html) (hereafter referred to as "COAZ") addresses the boundary between external protocols and AuthZEN. It defines a protocol-neutral framework for declaratively mapping the inputs of an incoming operation into a SARC authorization request. Values may be fixed or computed from operation inputs, normally using Common Expression Language (CEL), so that authorization decisions reflect the actual parameters and context of an operation rather than only a caller identity. Protocol-specific bindings define how a particular protocol is projected into the SARC model.  
* [COAZ-MCP: COAZ Binding for the Model Context Protocol](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html) (hereafter referred to as "COAZ-MCP") applies COAZ to the Model Context Protocol (MCP). It maps MCP JSON-RPC operations into AuthZEN evaluations, providing default mappings for MCP methods and allowing tool-specific mappings where greater precision is required. This enables fine-grained, parameter-level policy evaluation for AI agents at MCP gateways and servers. Future bindings could apply COAZ to other protocols, such as HTTP APIs, OpenAPI-described routes, gRPC services, or additional agent frameworks.

It is important to note that these specifications are not all at the same level of maturity. AuthZEN Authorization API 1.0 is a Final Specification and serves as the normative foundation for this assessment. By contrast, COAZ and COAZ-MCP are currently Draft 1 specifications, while the AuthZEN Policy Store Format remains a working draft ([GitHub Repo](https://github.com/nynymike/AuthZen_Policy_Store/tree/main)). Accordingly, any assessment of the broader AuthZEN ecosystem should be understood as including current draft specifications in addition to the finalized Core specification.

The evaluation therefore follows a layered approach. Each criterion is first assessed against the functionality provided by the Core specification alone. Where relevant, the analysis then identifies how Policy Store, COAZ, or COAZ-MCP materially strengthen, clarify, extend, or operationalize that functionality. This distinction prevents ecosystem features from being incorrectly attributed to the core protocol while still allowing AuthZEN to be evaluated as a practical authorization framework for agentic systems.

Many of the entries in the scorecard are marked as No. In part this reflects that AuthZEN addresses a different layer of the problem. Its primary role is to standardize runtime access decisions and PEP/PDP interoperability, and delegation chains, attenuation, and revocation are left to a capability system or delegation model elsewhere in the architecture. One No is substantive, however. An AuthZEN decision is drawn from the subject's standing permissions, so on its own AuthZEN is an ambient authority framework and is subject to the confused deputy problem. Its fit in a delegation stack is therefore at the edges rather than at the point of access. It can serve as the interface through which a resource owner's policy engine decides which capabilities to hand out to which authenticated principal. And, inside a resource's own verifier, it can evaluate the concrete action, resource, and context of a request at execution time, which is relevant to the dynamic least-privilege problem the Introduction describes.

### Scorecard

Each entry gives the requirement, the verdict, and the reasoning.

**1. Attenuated.** Verdict: No.

The core API has no native delegation or attenuation model. The broader ecosystem also does not establish that downstream authority is a non-widening subset of upstream authority. Any attenuation must therefore be implemented by the underlying policy engine and data model, not by AuthZEN itself.

**2. Accountable (agent vs. principal/operator).** Verdict: Partial.

AuthZEN core can identify a subject but does not standardize relationship between the responsible-principal and the agent. COAZ-MCP separates the human subject from the agent context, so a decision record names both parties, which improves attribution. It does not establish a delegation proof or legal-operator binding.

**3. Resistant to confused deputy.** Verdict: No.

AuthZEN evaluates explicit Subject-Action-Resource tuples, but the decision is drawn from the subject's standing permissions, with the caller-designated resource as a separate input. That is the confused deputy condition. COAZ and COAZ-MCP make the request reflect the actual operation but do not change whose authority answers it.

**4. Represent authorization policies (embed/link; functional restrictions, re-delegation rules, advisory "please").** Verdict: No for core (out of scope); Partial with ecosystem.

Policy language, architecture, and state management of the PDP are outside the scope of AuthZEN. The ecosystem improves policy-artifact portability and request-binding interoperability through Policy Store, COAZ, and COAZ-MCP. However, authorization-policy semantics remain policy-engine-specific and are not inherently interoperable across policy engines or organization.

**5. Chainable.** Verdict: No.

Verdict is "No" because it is not a delegation framework, not because it is incompatible with delegation frameworks. Neither the core data model nor COAZ-MCP contains a normative parent grant, delegator, delegatee, delegation edge, parent capability, or proof-chain primitive. COAZ-MCP improves subject/agent attribution, but not multi-hop delegation. [Access Evaluations](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluations-api) support request batching (boxcarring) but not chaining. Each evaluation is processed independently and cannot depend on the outcome of another evaluation in the same request.

**6. Cross-organizational / locally verifiable.** Verdict: No.

Enables cross-vendor request/decision interoperability. Use across organizations requires trusting a remote PDP or federating identities, which merges the parties into one trust domain. There is no locally verifiable proof of delegated authority.

**7. Self-revocable (holder or anyone in chain).** Verdict: No.

No native self-revocation of delegated authority. Revocation is a responsibility of surrounding policy/token/delegation systems. Policy changes can revoke access, but this is different from delegation-chain self-revocation by the delegator or holder.

**8. Authentication / Proof of Possession (nice-to-have).** Verdict: No (out of scope).

AuthZEN relies on external authentication mechanisms and does not define holder proof-of-possession for delegated authority.

### Detailed Evaluation

#### Attenuated

The Core alone can express narrowly scoped, context-sensitive access questions through SARC, making it well suited to runtime least-privilege enforcement. However, as with the previous sections, given the scope of the specification, neither monotonic attenuation nor the prevention of authority widening across delegation hops is defined by AuthZEN itself.

Adding the three ecosystem specifications does not improve support for attenuation. As discussed in the previous sections, the Policy Store does not establish parent-child delegation semantics or a monotonic subset invariant. COAZ strengthens the trustworthy construction of fine-grained authorization inputs, but it explicitly preserves AuthZEN request semantics rather than defining delegation semantics. COAZ-MCP extends this mapping framework to MCP, and its controls can enforce constraints supplied by policy. However, none of these specifications establishes that a child delegation cannot exceed the authority of its parent.

Thus, AuthZEN provides strong runtime enforcement for attenuated authority defined and validated elsewhere, but it does not provide native delegation attenuation.

#### Accountable (agent vs. principal/operator)

The Core can identify a [Subject](https://openid.net/specs/authorization-api-1_0.html#name-subject), defined as a "user or machine principal," through mandatory type and id fields and optional properties. An [Access Evaluation API](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluation-api) contains one and only one Subject. While that Subject may represent either a user or a machine principal, the specification provides no standard way to model additional Subjects, distinguish an acting agent from an accountable principal/operator, or express a standardized relationship between them. [Context](https://openid.net/specs/authorization-api-1_0.html#name-context) could carry operator, delegator, or chain information, but their semantics would be implementation-specific rather than interoperable. Therefore, an auditor cannot reliably reconstruct the accountable principal or hop-by-hop delegation solely from a conformant AuthZEN request. The specification also excludes policy architecture, state management, and API authentication from its scope.

The result is more favorable when considering the broader ecosystem rather than the Core specification alone.

With COAZ-MCP, the binding explicitly separates the human subject from the acting agent by allowing the human identity to populate subject.id while representing the agent in [context.agent](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim). As a result, auditors can determine not only that an agent acted, but also on whose behalf the authorization decision was evaluated. However, the specification does not define delegation relationships or delegation chains.

COAZ strengthens the trustworthiness of this model through mapping controls and [trust-anchored](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields) fields. Bindings may require critical fields, such as subject identity, to be derived from independently verifiable inputs rather than caller-supplied data. This improves identity provenance and reduces the risk of agent-supplied identity assertions. Nevertheless, COAZ remains an identity-to-authorization mapping framework and does not define delegation relationships or delegation chains.

Thus, the ecosystem improves agent-versus-principal attribution and identity provenance, but it still falls short of providing interoperable delegation provenance and multi-hop accountability.

Two agentic patterns have no standard expression in the Core. An agent that starts another agent cannot record the parent in the Subject, since a request carries one Subject. The relationship can only go into Context, with implementation-specific meaning (Issue \#416, discussed under Chainable, asks for exactly this). Short-lived agents fare worse. A decision can only be made about a Subject the PDP's identity data already knows, so every agent, however brief its life, must be provisioned in the identity layer before it can be authorized. That is the provisioning cost the Introduction identifies as unworkable at agentic scale.

#### Resistant to confused deputy

AuthZEN's [data model](https://openid.net/specs/authorization-api-1_0.html#name-information-model) asks "may this subject perform this action on this resource?", with optional Context. [Resource](https://openid.net/specs/authorization-api-1_0.html#name-resource) names the target and [Action](https://openid.net/specs/authorization-api-1_0.html#name-action) the operation, which allows resource-specific decisions rather than role checks. This is the shape of the confused deputy problem, not a defense against it. The resource is one input and the permission is the output. The decision is drawn from the subject's own standing permissions, and nothing binds the resource the caller designated to the authority being used. The PEP constructs Subject, Resource, and Action from the invocation, API authentication is out of scope, and enforcement is the PEP's responsibility. A deputy acting on a caller-supplied resource is therefore authorized whenever its own permissions cover that resource. That is ambient authority.

COAZ and COAZ-MCP ensure that the authorization request reflects the actual operation. COAZ requires [bindings](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-information-model-and-input) to specify the source and structure of input variables, treats mapping inputs as potentially untrusted, provides a mechanism for [trust-anchored fields](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields), and requires mapping inconsistencies or evaluation failures to be treated as [mapping errors](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-denials-and-errors). COAZ-MCP binds MCP [tool names, arguments, resource URIs, token claims](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-information-model), and [server audience information](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-default-mappings) into authorization requests, with a trust-anchored [subject.id](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim) and [declared mappings](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-declared-mappings) that expose tool parameters to policy. This improves the fidelity of the question asked. It does not change whose authority answers it. The question is still evaluated against the deputy's permissions at the moment of the call, with the caller-designated resource as an input.

Confused-deputy resistance requires a designation-and-authority artifact that travels with the request, which neither the Core nor the ecosystem defines. The verdict is No.

#### Represent authorization policies

Core specification does not standardize an authorization-policy representation. The Access Evaluation model provides SARC, but it does not define a standardized mechanism for embedding or referencing authorization policies themselves. Furthermore, the PDP's policy language, architecture, and state management are explicitly outside the scope of the specification. The specification illustrates advice, obligations, reasons, and step-up instructions as possible uses of [Decision Context](https://openid.net/specs/authorization-api-1_0.html#name-decision-context), but explicitly leaves their semantics and format outside the standard. Consequently, AuthZEN does not provide interoperable semantics for advisory rules, re-delegation constraints, or disclosure-governance policies.

Policy Store standardizes policy packaging and interchange, not policy semantics. Policies remain policy-engine-specific. Consequently, policy artifacts can be exchanged and discovered through a common packaging mechanism, but the semantics of delegation, re-delegation rules, undelegatable authority, and other authorization policies are not inherently interoperable across policy engines or organizations, since their meaning and enforcement are still defined by the underlying policy engine and its policy model.

COAZ and COAZ-MCP standardize how operations are mapped to AuthZEN authorization requests ([COAZ Mapping](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-the-mapping), [COAZ-MCP Declaring a Mapping](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-declaring-a-mapping)), not the semantics of authorization policies. COAZ provides declarative mappings from protocol inputs into AuthZEN SARC requests, while COAZ-MCP allows such mappings to be embedded in MCP tool definitions and enforced by a PEP before execution. Consequently, operations can be bound to authorization requests in a portable and interoperable manner, but the meaning of the policies being evaluated remains defined by the underlying PDP and policy engine rather than by COAZ or AuthZEN.

None of these specifications defines a common authorization-policy language or semantic model. They improve interoperability at the authorization interface layer (requests, responses, and policy artifacts), but not at the policy semantic layer. Consequently, different PDPs can exchange authorization requests and decisions through a common protocol, and policy artifacts may be shared through common packaging mechanisms, but the meaning and enforcement semantics of those policies remain implementation specific.

#### Chainable

If being chainable is interpreted as preserving delegated authority across multiple hops, AuthZEN and its current ecosystem do not provide a standardized delegation-chain model. Delegated authority remains external to the AuthZEN interoperability model. The same observation largely applies to provenance reconstruction. However, AuthZEN itself should not be interpreted as an impersonation-oriented model. Although it does not solve multi-hop delegation, neither does it force workflows into impersonation. Instead, AuthZEN can consume and evaluate externally provided delegation-related context as part of a delegated-authority workflow. Because that context is outside the specification, its meaning is implementation-dependent, and two deployments that both carry delegation context need not interpret it the same way. Delegated authority consumed this way is not interoperable.

The [Access Evaluations API](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluations-api) should likewise not be interpreted as delegation chaining. It supports batching (boxcarring) of authorization requests, but each evaluation remains an independent authorization decision. Multiple evaluations do not create, transfer, attenuate, or propagate authority between actors. Consequently, request batching is fundamentally different from a genuine delegation chain and should not receive chainability credit.

The ecosystem specifications improve interoperability but do not materially change this conclusion. Policy Store, COAZ, and COAZ-MCP improve portability, interoperability, and attribution, but none of them define delegated authority as a first-class, verifiable artifact across multiple hops. Nevertheless, like AuthZEN itself, they do not inherently force delegated-authority workflows into impersonation.

[AuthZEN Issue \#416 "\[Feature Request\] Add 'delegation\_chain' to Subject for AI Agent and Multi-Actor Scenarios"](https://github.com/openid/authzen/issues/416) further illustrates this point, which proposes introducing a `delegation_chain` structure for AI-agent scenarios. The proposal explicitly argues that the current subject model cannot adequately represent multi-hop delegation chains involving users, agents, and downstream services. This should not be interpreted as existing support. Instead, it demonstrates that delegation provenance and multi-hop authority flow are recognized gaps in the current model.

Accordingly, if chainability is viewed as a property of delegated authority itself rather than authorization interoperability, AuthZEN should still receive No. Nevertheless, the reason is not that AuthZEN prevents delegation chains or requires impersonation. AuthZEN deliberately operates at a different architectural layer. It can evaluate and enforce authorization decisions informed by delegated-authority context, but the delegation chain itself, including attenuation, provenance, chain verification, and revocation semantics, must be supplied by another system. AuthZEN is therefore best characterized as a consumer of delegation chains rather than a provider of delegation chains.

#### Cross-organizational / locally verifiable

A PDP makes access decisions for its own organization's resources. The Core makes the request and decision formats interoperable across vendors, so an organization can swap PDP products or combine PEPs and PDPs from different suppliers. That is cross-vendor interoperability, not cross-organizational execution.

For a second organization to rely on an AuthZEN decision, it must trust the first organization's PDP, together with its token issuers and [identity claims](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim). ([COAZ Trust-Anchored fields](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields) improve input trustworthiness but do not establish that trust.) Trusting a remote PDP, or federating identities, merges the two parties into a single trust domain whose boundary happens to span two companies. It does not let them operate as independent domains.

The [Access Evaluation API](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluation-api) returns a decision, not evidence that another party could verify independently, and neither Policy Store, COAZ, nor COAZ-MCP defines an authorization-proof format. There is nothing to verify locally except the PDP's own answer. The criterion is therefore rated No. An organization that must accept delegated authority from parties it does not federate with needs a delegation credential from outside the AuthZEN ecosystem. AuthZEN can then serve as the decision interface inside each party's own verifier.

#### Self-revocable

The core specification provides no protocol mechanism by which a holder or an ancestor in a delegation chain can revoke a specific delegated authority, nor does it define any requirement for revocation to cascade to descendants. While changes to PDP policy may cause subsequent evaluations to return false, the specification does not define who is authorized to make such changes or whether that authority derives from participation in a delegation chain.

The specification also defines no revocation-state object, revocation-propagation protocol, cache-invalidation mechanism, network-partition behavior, or maximum revocation latency. Although expiration timestamps or other short-lived authorization evidence may be supplied as [contextual input](https://openid.net/specs/authorization-api-1_0.html#name-context) by an implementation, this represents expiration rather than explicit revocation and is not standardized by the API.

The ecosystem does not add delegation-chain self-revocation functionality. The Policy Store specification allows administrators to upload [immutable, versioned Policy Store](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html#name-policy-store-api) packages through a [POST-only Policy Store API](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html#name-api-request), while existing policy stores cannot be updated or deleted. This supports policy deployment, versioning, and rollback, but not holder- or ancestor-initiated revocation. COAZ defines how protocol-specific information models are mapped into AuthZEN authorization requests, but it introduces neither delegation lineage nor revocation operations. Similarly, COAZ-MCP does not define holder- or ancestor-driven revocation, child-selective revocation, cascading revocation semantics, or guarantees regarding cache consistency, revocation propagation, or network partitions.

#### Authentication / Proof of Possession

Authentication of the Authorization API is explicitly out of scope. While OAuth 2.0 support is [RECOMMENDED](https://openid.net/specs/authorization-api-1_0.html#name-model), OAuth 2.0 deployments are typically based on bearer tokens and therefore do not inherently provide holder proof-of-possession. Achieving proof-of-possession generally requires additional mechanisms such as DPoP, mTLS, sender-constrained tokens, or comparable cryptographic binding techniques. AuthZEN neither mandates nor standardizes any of these mechanisms. Consequently, while AuthZEN can operate in deployments that provide proof-of-possession through external identity and transport layers, the specification itself does not define holder-bound credentials or cryptographic proof-of-possession for delegated authority.

The ecosystem improves the integrity and trustworthiness of identity-related inputs, but it still does not satisfy the proof-of-possession criterion. COAZ can designate [trust-anchored fields](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields) that must be derived from trusted inputs or verified by the PEP. COAZ-MCP uses [decoded JWT OAuth access-token](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-information-model) claims and [recommends anchoring subject.id to the designated subject-identity claim](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-declared-mappings). It also keeps agent identity separately in [context.agent](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim). These mechanisms reduce the risk of identity-claim substitution, but they do not require sender-constrained tokens, a signed invocation, or proof that the agent possesses a private key bound to delegated authority. The Policy Store packages [trusted-issuer](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html#name-trusted-issuers) configuration and policy artifacts, but it likewise does not establish holder proof-of-possession for delegated authority.

### Additional Considerations

#### Search is discovery, not proof of authority

AuthZEN Search is designed for discovery rather than for conveying authority. The specification states that "Search APIs provide lists of resources, subjects or actions that would be allowed access." Search results indicate what may be permitted under the PDP's current state, but they do not by themselves constitute proof of delegated authority, capability possession, or authorization grants.

Search also carries an information-disclosure risk. Reverse-query APIs expose information derived from the current state of the authorization system. If made available to insufficiently trusted callers, repeated or carefully crafted searches may reveal the existence of resources, access relationships, or characteristics of effective authorization policies. Appropriate authentication, query scoping, rate limiting, response minimization, and careful handling of diagnostic information should therefore be considered.

#### Policy Store administration is not standardized

The Policy Store specification does not define a normative authorization model for administration of the Policy Store itself. Its Security Considerations section indicates that security, authorization, and operational controls are implementation responsibilities. The specification does not prescribe who may create, modify, approve, publish, or delete policy bundles, nor does it define administrative roles, delegation models, or approval workflows for policy governance. Organizations may therefore adopt different approaches, ranging from repository-based access controls and CI/CD approval processes to dedicated PAP solutions. This preserves implementation flexibility, but governance, segregation of duties, and administrative access control for the Policy Store remain implementation-specific.

#### Decision interoperability is not policy or delegation interoperability

As noted throughout the detailed evaluation, AuthZEN achieves interoperability through authorization requests and access decisions, not through policy semantics. A conformant PDP may use Cedar, Rego, Zanzibar-style relationships, or another policy system while exposing the same AuthZEN interface, and one organization can consume access decisions produced by another organization's PDP without either PDP sharing, interpreting, or enforcing the other's policies. The same caution applies to the Policy Store. It makes policy artifacts portable between systems, but the semantics of delegation, attenuation, and revocation expressed in those artifacts remain specific to the underlying policy engine. A portable policy store can therefore produce different delegation behavior across authorization engines and deployments.

### Summary

AuthZEN can serve as a strong runtime authorization and policy-decision layer within an agentic authorization stack. Its primary value lies in standardizing authorization decisions and interoperability between authorization components, while supporting emerging AI-agent and tool-invocation scenarios through initiatives such as COAZ and COAZ-MCP.

However, AuthZEN is not a delegated-authority system. Delegation semantics, including delegation proof, attenuation, and revocation, remain outside the scope of Authorization API 1.0 and are provided by other components of the overall architecture. Likewise, policy representation is intentionally left to the underlying PDP implementation.

Put plainly, AuthZEN on its own is an ambient authority framework. A PDP answers from the subject's standing permissions, and nothing in the protocol binds the caller's designation of a resource to the authority used. It should not be the sole basis for access decisions in a delegation stack. Its appropriate roles are deciding which capabilities to hand out to an authenticated principal, and serving as the internal decision interface of a resource's own verifier.

Understanding these boundaries is important when comparing AuthZEN with policy frameworks such as Cedar and delegation-focused approaches such as UCAN and zCAP.

### References

* [OpenID AuthZEN Authorization API 1.0 Specification](https://openid.net/specs/authorization-api-1_0.html)  
* [OpenID AuthZEN Policy Store Format Specification](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html)  
* [COAZ: A Framework for Mapping Information Models to AuthZEN Authorization Requests - Draft 1](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html)  
* [COAZ-MCP: COAZ Binding for the Model Context Protocol - Draft 1](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html)  
* [AuthZEN GitHub Issue \#416 "\[Feature Request\] Add 'delegation\_chain' to Subject for AI Agent and Multi-Actor Scenarios"](https://github.com/openid/authzen/issues/416)  
* [OpenID AuthZEN Meeting Notes](https://github.com/openid/authzen/wiki/Meetings)  
* [AuthZEN Interop](https://authzen-interop.net/docs/intro/)

## Synthesis: Comparing the Surveyed Specifications

The preceding chapters evaluated each specification on its own terms. This chapter puts them side by side: first a taxonomy that explains why the specs are so different in shape, then the merged scorecard, then the two comparisons the scorecard raises (server-mediated versus certificate delegation, and the policy decision family as a complement rather than a competitor), and finally the gaps that no surveyed spec fills. Those gaps are the working group's candidate agenda.

### Three Families

The surveyed specs are not seven competitors for one job. They fall into three families, distinguished by where authority lives:

* Certificate capabilities (zCaps, UCANs, and KYA-OS, which profiles zCaps inside Verifiable Credentials). Authority lives in a portable, signed artifact. The delegatee holds it, can attenuate and re-delegate it offline, and presents it with proof of key possession. The verifier checks the chain locally; no authorization server exists. KYA-OS adds an identity and audit layer around that artifact and is the one member that resolves the chain from a registry rather than carrying it.  
* Server-mediated authorization (the OAuth 2.0/2.1 lineage, AAuth). Authority lives in a server: the OAuth Authorization Server, or AAuth's Person Server with its mission state. Tokens reference that authority rather than embodying it, and delegation, attenuation, and revocation are acts performed at or by the server. An Authorization Server performing Token Exchange could keep the full delegation chain (the act claim is designed to carry it), though we know of no deployment that exposes such a record. OAuth access tokens are also frequently used in practice as identity assertions, which pulls such deployments toward the policy decision family.  
* Policy decision systems (Cedar, AuthZEN). Authority lives in the verifier's policy store. There is no token of authority at all; the request presents identity evidence and the verifier evaluates rules it already holds. Cedar is the family's policy language. AuthZEN is its wire protocol, standardizing how an enforcement point (PEP) asks a decision point (PDP) for a verdict. The two cover complementary layers of the same architecture, much as zCaps and UCANs do for the certificate family.

One further distinction cuts across the families. Object-reference capabilities (listed in the Introduction) are invocation-centric: a delegation happens only by passing the reference as a parameter of an invocation. The certificate systems surveyed here are delegation-centric: a delegation is an act of its own, performed outside any invocation other than delivering the certificate. The server-mediated systems sit closer to the invocation-centric model, since a delegation is itself a protocol call on the server.

The families also differ in which of the specification types from the Introduction (data model, protocol, possession mechanism, chainable proof methods) each spec actually covers. Much of the apparent unevenness between chapters reflects this: the chapters describe different layers of a potential stack, not six answers to the same question.

| Spec | Data model | Protocol | Possession mechanism                        | Chainable proofs | Policy language |
| :---- | :---- | :---- |:--------------------------------------------| :---- | :---- |
| OAuth 2.0 / 2.1 | Partial (scopes, JWT claims) | Yes (core focus) | Optional (DPoP, mTLS)                       | Server-mediated (Token Exchange, act claim; online) | No (opaque scope strings) |
| AAuth | Partial (act claim, tokens) | Yes (core focus) | Yes (signed requests, core)                 | Server-mediated (Mission Log) | No (deferred to PS policy) |
| zCaps | Yes (core focus) | Out of scope (invocation and revocation conventions only) | Yes (HTTP signatures, Data Integrity)       | Yes (core focus) | Extension point (caveat, reserved) |
| UCANs | Yes (core focus) | Partial (invocation and revocation sub-specs) | Yes (signed envelope)                       | Yes (core focus) | Extension point (pol syntax, no vocabulary) |
| KYA-OS | Yes (VC 2.0 wrapping a zCap; typed Entity Card) | Yes (MCP binding, consent challenge, outbound headers) | Yes (per-request detached JWS in MCP `_meta`) | Yes (one credential per hop, verifier-walked) | Extension point (zCap caveats; two types standardized) |
| Cedar | No (no authorization artifact) | No | No                                          | No | Yes (core focus) |
| AuthZEN | Partial (SARC request shape; no authorization artifact) | Yes (core focus: PEP/PDP evaluation and search) | No (out of scope)                           | No | No (explicitly out of scope; engine-specific) |

### Data Model Correspondence

The family taxonomy is easiest to see at the level of concrete fields. The table below maps each spec's terminology onto the common data model from the Overview: despite the differences in encoding, the same shape recurs wherever a spec defines an authorization artifact at all. (Cedar is absent because it defines no such artifact; its equivalents are schema entities in the verifier's store. AuthZEN's column reflects the request shape a PEP submits, not an authority artifact, per its chapter. GNAP (RFC 9635) is included for field comparison only. It is not evaluated in this draft, although it is arguably more relevant to agentic delegation than some systems that are, and it is a candidate for a chapter in a later revision. dSD-JWT is included on the same basis; see the Appendix.)

| Common field | OAuth 2.1 | AuthZEN | GNAP | zCaps | UCANs | KYA-OS                                                                                                         | AAuth                                                                                                                        | dSD-JWT |
| :---- | :---- | :---- | :---- | :---- | :---- |:---------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------| :---- |
| Agent | `client_id` | context.agent (COAZ-MCP); no agent concept in core | client (rich object) | controller (DID) | Delegation: aud, Invocation: iss | invoker (DID; a one-off did:key is recommended); entityType on the Entity Card                                 | agent identifier (`agent_token` subject, stable across key rotation)                                                         | cnf key (JWK) named by the previous hop; no identifier |
| Principal | sub | subject | subject | implicit (in delegation chain) | Delegation: iss, Invocation: sub | issuer of the root credential (Responsible Party), recomputed; principal locator on the Card                   | person, vouched by the Person Server                                                                                         | holder of the base SD-JWT VC, bound by its cnf |
| Invocation / key binding | Optional, via DPoP | no | Yes (DPoP, HTTPSig, mTLS) | Yes: HTTP signatures plus Capability-Invocation header, or Data Integrity proof | Yes: its own signed-envelope mechanism; keys resolved from DIDs | Yes: detached JWS in MCP `_meta` over the canonicalized call, with audience and nonce; optional RFC 9421 sibling | Yes: HTTP Message Signatures on every request; cnf binds tokens to the current key                                           | Yes: Key Binding JWT per hop, with aud and nonce at presentation |
| Resource | scope or RAR (structured scope); see also htm/htu claims | resource and context | locations and datatypes | invocationTarget (URL or object) | specified in args | invocationTarget (DID, constant along the chain)                                                               | `resource_token` (names resource and scope; optional account parameter)                                                        | none defined; left to the delegate payload (profile-defined) |
| Actions | scope (also htm) | actions | actions | allowedAction (string or array) | cmd | allowedAction                                                                                                  | scope (in resource and auth tokens)                                                                                          | none defined; left to the delegate payload (profile-defined) |
| Caveats / additional policy | none in the token; an AS could define caveat semantics through scope or RAR, but none is known to | context | ? | caveat (reserved, unused in deployment) | pol (policy statements) plus args | caveats (typed; ValidUntil and MaxAmount standardized, unknown types copied verbatim)                          | none in the token; mission context evaluated server-side at the PS                                                           | none defined; draft permits constraints as visible claims, no vocabulary |
| Delegation chain | only via Token Exchange (act claim, held at the AS) | no | not defined by RFC 9635 | Yes: capabilityChain proof chains | Yes: proof chains (CID-linked) | Yes: one VC per hop linked by parentCapability; referenced by a delegationRef locator, not carried             | act claim plus the server-held Mission Log                                                                                   | Yes: hops appended as disclosures, each signing a hash of the prior credential state (sd_hash or issuer_jwt_hash) |
| Protocol (how is authorization requested?) | OAuth 2.1 itself | AuthZEN | GNAP | out of scope (uses GNAP, VCALM, and similar) | out of scope (transport left open) | signed `needs_authorization` challenge with a consent URL, then resume; MCP binding                             | AAuth itself                                                                                                                 | OpenID4VP transaction_data item |
| Revocation | RFC 7009 | n/a (Policy Store rollback is versioning, not revocation) | n/a | Yes: out of band, at the RS | Yes: Revocation sub-spec; also a revocation capability | Yes: Bitstring Status List per hop, cascading, fail-closed; issuer or ancestor only                            | specified scenarios, exercised by the issuing servers; tokens are short-lived (one-hour auth tokens in the draft's examples) | Token Status List on the base credential only; no per-hop mechanism |

Two readings of this table reinforce the scorecard that follows. Empty and "n/a" cells are findings, not omissions: OAuth's token has no place for a caveat, and its chain exists only as the Authorization Server's act claim, which predicts its rows 4 and 5 below. And the two cells where AAuth diverges from the certificate pattern (chain and caveats both live at the server, not in the token) are precisely what produces its server-mediated column throughout.

### The Merged Scorecard

Verdicts below are taken from the per-spec chapters, which carry the detailed reasoning; the notes here are compressed to one line. zCaps, UCANs, and KYA-OS are shown separately for completeness, but as their chapter establishes, they score as a family, with KYA-OS departing on rows 2, 6, and 7. Cedar and AuthZEN likewise share a family, though they cover different layers of it and their verdicts diverge accordingly. dSD-JWT is included from the Appendix for comparison.

**1. Attenuated.**

* OAuth 2.0/2.1: Via the AS only (scope narrowing, Token Exchange); not by the holder.
* AAuth: Partial. differs by mode: sub-agent yes; call chaining intentionally no, and downstream scope may exceed upstream.
* zCaps: Yes. monotonic, verifier-enforced, including caveats added per hop.
* UCANs: Yes. monotonic, verifier-enforced, including policy added per hop.
* KYA-OS: Yes. monotonic, verifier-enforced; target fixed, actions and caveats narrow.
* Cedar: Partial. forbid-composition shrinks authority, but no subset check between grants.
* AuthZEN: No. attenuation left to the underlying policy engine.
* dSD-JWT: No. payload is arbitrary; narrowing deferred to a profile.

**2. Accountable (agent vs. principal/operator).**

* OAuth 2.0/2.1: Yes with asymmetric client auth; weaker with shared `client_secret`.
* AAuth: Yes. distinct authenticated roles, act claim.
* zCaps: Partial. signed edges, but principals are opaque DIDs.
* UCANs: Partial. same as zCaps.
* KYA-OS: Yes. typed entities; Responsible Party and Principal recomputed from the root issuer.
* Cedar: Yes at the modeling level; not a cryptographic delegation edge.
* AuthZEN: Partial. COAZ-MCP separates subject.id from context.agent; no delegation proof or operator binding.
* dSD-JWT: Partial. signed hops, but delegatees are bare JWKs, not identifiers.

**3. Resistant to confused deputy.**

* OAuth 2.0/2.1: Partial. in practice a single-audience token plus the request URL behaves as a capability, but the token itself does not designate, and multi-audience tokens break the binding.
* AAuth: Yes in three of four modes; the identity-based mode is ambient.
* zCaps: Yes. by construction.
* UCANs: Yes. by construction.
* KYA-OS: Yes. by construction; inherits the zCap pair, adds per-hop audience.
* Cedar: No. identity-based, mitigation only.
* AuthZEN: No. ambient authority; the subject's standing permissions decide, and the resource named in the request is a separate input.
* dSD-JWT: Partial. chain evaluated, but no resource or action in the artifact.

**4. Represent authorization policies.**

* OAuth 2.0/2.1: Partial. scopes and RAR; extension-dependent.
* AAuth: Partial. policy intentionally outside the protocol.
* zCaps: Partial. caveat slot reserved, unused, no vocabulary.
* UCANs: Partial. pol syntax defined, no vocabulary.
* KYA-OS: Partial. two caveat types standardized; no delegation-control vocabulary.
* Cedar: Yes. except the advisory tier; core competence.
* AuthZEN: Partial. Policy Store packages artifacts; semantics stay engine-specific (core: out of scope).
* dSD-JWT: No. no policy slot; terminal hop is the only control.

**5. Chainable.**

* OAuth 2.0/2.1: Partial. Token Exchange (RFC 8693) gives server-mediated multi-hop delegation, with the chain in the act claim at the AS; online, not holder-side.
* AAuth: Yes. server-mediated: act claim plus Mission Log; sub-agent nesting single-level.
* zCaps: Yes. parentCapability plus capabilityChain.
* UCANs: Yes. hash-linked proof chains.
* KYA-OS: Yes. one VC per hop, parentCapability links.
* Cedar: No. no chain artifact; stack-dependent.
* AuthZEN: No. consumer of delegation chains, not a provider.
* dSD-JWT: Yes. hash-bound hops rooted in the issuer signature.

**6. Cross-organizational / locally verifiable.**

* OAuth 2.0/2.1: Yes. the client may belong to any organization (the classic photo-printing example); JWT tokens verify locally, opaque tokens need the AS online.
* AAuth: Yes. cross-domain by design; trust via configured relationships.
* zCaps: Yes. local cryptographic verification, DID principals.
* UCANs: Yes. same, via CID resolution.
* KYA-OS: Partial. local check, but the chain is resolved from a registry rather than carried.
* Cedar: Partial. evaluation local, policy store unilateral.
* AuthZEN: No. decisions interoperate across vendors; a trusted remote PDP is federation into one domain, not cross-domain verification.
* dSD-JWT: Yes. local verification, no authorization server.

**7. Self-revocable (holder or chain).**

* OAuth 2.0/2.1: Partial. RFC 7009 plus short lifetimes; no chain concept.
* AAuth: Partial. revocation by the servers holding authority, not the holder.
* zCaps: Yes. at the RS revocation endpoint.
* UCANs: Yes. via the Revocation sub-spec.
* KYA-OS: Partial. issuer or any ancestor via status lists; the holder cannot.
* Cedar: Partial. policy store change by the administrator.
* AuthZEN: No. policy change is administrative, not holder revocation.
* dSD-JWT: No. status list on the base credential only.

**8. Authentication / Proof of Possession (nice-to-have).**

* OAuth 2.0/2.1: Optional (DPoP. mTLS); bearer remains the deployed default.
* AAuth: Yes. every request signed, core mechanism.
* zCaps: Yes. HTTP signatures or Data Integrity proofs.
* UCANs: Yes. signed envelope, nonce, time bounds.
* KYA-OS: Yes. detached JWS per request, audience, nonce, leaf-invoker join.
* Cedar: No. out of scope; evidence-issuance checks only.
* AuthZEN: No. out of scope; bearer OAuth is the recommended default.
* dSD-JWT: Yes. Key Binding JWT per hop, aud and nonce.

### Reading the scorecard

Three patterns are worth drawing out, because they organize everything else in this chapter.

The capability family owns the chain rows. On the criteria that concern the mechanics of delegated authority (attenuated, confused deputy, chainable, cross-organizational, self-revocable: rows 1, 3, and 5 through 7), zCaps and UCANs are the only pair of specs that is Yes across the board, and they achieve it offline and holder-side. KYA-OS, the third family member, keeps the verdicts on rows 1, 3, and 5 and drops to Partial on rows 6 and 7, in both cases because of choices in its credential carrier rather than in its capability model. Every other spec pays for those rows with a server dependency, a partial verdict, or both. This is the clearest architectural line in the survey, and it is a family property, not a property of any one spec individually.

The identity side owns the semantic rows. On accountability (row 2\) and policy expression (row 4), the verdicts invert. The systems that know who everyone is (AAuth's authenticated roles, Cedar's typed principals, OAuth clients with asymmetric auth) can distinguish agent from operator natively; zCaps and UCANs deliberately keep principals opaque and delegate that distinction to a companion identity layer. KYA-OS is the one capability-family spec with a Yes on row 2, and it earns it by building that companion layer into the same specification, at the cost discussed under Modeling Chained Delegation with W3C VCs. AuthZEN's COAZ-MCP binding follows the identity side's pattern, separating the human subject from the acting agent, though without a delegation proof it stops at attribution. Likewise the richest policy expression in the survey (Cedar) sits in the one spec with no delegation mechanics at all, while the specs with delegation mechanics reserve policy extension points and leave them empty.

No spec is complete. Every spec has at least one No or Partial in the required rows 1 through 7. The capability family misses semantics (rows 2 and 4); the server-mediated family misses holder-side and offline mechanics (rows 1, 5, 7 in various ways); the policy decision family misses the delegation artifact entirely (row 5) and is identity-based (row 3). AuthZEN has the sparsest set of verdicts in the scorecard, and its chapter is explicit about why. The spec addresses the decision interface layer, so most of its No verdicts mark a different layer rather than a failed attempt at delegation. The confused deputy No is the exception: a decision drawn from standing permissions is ambient authority at whatever layer it sits. This is the observation that motivates both the delegation-stack section and the gaps section below.

### Server-Mediated vs. Certificate Delegation

AAuth and the certificate capability family aim at the same use case (autonomous agents acting across organizational boundaries under delegated, revocable, auditable authority) and give opposite answers to the central question of where authority lives. The contrast is the sharpest one available in this survey, precisely because so much else about them agrees: both reject bearer tokens for signed requests, both identify agents by key material, both record who delegated what to whom.

* Portability. A certificate capability is self-contained: the chain travels with the invocation and any verifier can check it. AAuth's delegation history lives in the Person Server's Mission Log; verification of standing depends on authenticated interaction with the server that holds the state.  
* Offline delegation. The capability family delegates by signing, with no round trip. AAuth has no offline delegation; obtaining and extending authority are online protocol exchanges. This is the efficiency and partition-tolerance line, laid out stage by stage under Offline Use Considerations in the Introduction.  
* Attenuation. The capability family enforces algebraic, monotonic attenuation at every link: a child provably holds a subset. AAuth deliberately rejects the subset rule for call chaining; the Person Server evaluates each hop against mission context, which its specification argues is more flexible than algebraic rules. This is a genuine philosophical split (constraint by algebra versus constraint by governance), not an omission on either side.  
* Revocation locus. In the capability family, anyone in the chain can revoke, and the enforcement point learns of it. In AAuth, the servers holding authority revoke; the holder of a credential cannot act on the credential itself.  
* Key lifecycle. AAuth delegations reference stable agent identifiers, with the cnf claim binding tokens to the current key; identity survives key rotation, at the cost of lifecycle questions about delegations that straddle a rotation. The capability family binds delegations to keys directly, and typically sidesteps rotation with short-lived, delegation-specific keys.

For agentic delegation, the trade reduces to this: the certificate model gives stronger guarantees (offline, locally verifiable, algebraically attenuated, holder-revocable) at the price of thin semantics, while the server-mediated model gives richer semantics and governance (missions, consent, authenticated roles) at the price of a live, trusted server on the path of authority.

### The Policy Decision Family as Complement

Cedar is, almost row by row, the complement of the capability family: strongest exactly where zCaps and UCANs are weakest (policy expression, agent-vs-operator modeling, fine-grained revocation, composition of authority) and weakest exactly where they are strongest (no chain, no portable grant, no holder-side anything). Read as a competitor it fails the survey; read as a component it fills the survey's emptiest cell.

The specific fit: zCaps reserve a caveat slot with no defined syntax, and UCAN specifies a policy syntax in detail but ships no vocabulary of delegation-control terms. Cedar is a mature, analyzable policy language with no carrier. A capability chain whose caveats were expressed in a Cedar-class language would give the family what its chapters identify as their highest-value missing piece, with one property nothing else in the survey offers: because Cedar is non-Turing-complete and formally analyzable, a delegator can prove at delegation time that a child's caveats are a subset of its own. That check is an optimization, so that no party carries a delegation that will later fail. Only the resource's verifier is trusted to decide an invocation. The reverse composition also holds: Cedar deployments that need portable, cross-organizational grants need exactly the carrier the capability family provides.

AuthZEN extends the same complement to the protocol layer, and makes the composition concrete. Cedar has no standard request/decision interface; AuthZEN supplies exactly that. In a composed stack, a capability verifier can act as a PEP: it checks the chain and proof of possession locally, then consults its own PDP over AuthZEN for the caveat evaluation the capability family leaves unspecified. The PDP in this arrangement belongs to the resource, so no ambient authority is introduced; it is an implementation detail of the verifier. AuthZEN also fits at the other end of the chain, as the interface a resource owner's policy engine exposes for deciding which capabilities to hand out to which authenticated principal. COAZ points the same direction from the other side, mapping live protocol operations (including MCP tool calls, via COAZ-MCP) into decision requests that reflect the actual invocation parameters. None of this requires the capability chain to change shape; it fills the empty caveat slot with an evaluation pipeline that already exists.

This suggests the shape of a composed delegation stack, assembled from surveyed parts:

1. Entry point (identity layer): an identity system (OIDC, Verifiable Credentials) makes the human or organizational judgement call and binds operator to agent; the zCap specification itself recommends this split, and KYA-OS's Entity Card and Responsible Party are one worked example of it.  
2. Authority flow (carrier layer): a certificate capability chain carries the delegated, attenuated authority across hops and organizations, offline and locally verifiable.  
3. Restrictions (policy layer): caveats within the chain expressed in a standardized, analyzable policy language. The resource's verifier evaluates them at invocation. A delegator may also check them at each hop, as an optimization to avoid handing out a delegation that will fail. Where a verifier hands evaluation to its own PDP, AuthZEN is the standard interface for asking.  
4. Presentation (possession layer): signed invocation of the transport in use (HTTP Message Signatures, Data Integrity proofs, DPoP-style binding), with keys held outside the agent per the Introduction's requirements.

No surveyed spec provides more than two adjacent layers of this stack, and no pair of them currently interoperates by standard. That observation leads directly to the gaps.

### Gaps

Reading down the merged scorecard, the recurring Partial and No verdicts cluster into five gaps that no surveyed specification fills. These are candidates for working group attention, ordered roughly by how directly they block agentic delegation.

1. A standard policy and caveat vocabulary for delegation control. Both certificate capability specs have the extension point and neither ships vocabulary; AAuth places policy outside the protocol; OAuth scopes are opaque strings; AuthZEN standardizes the decision request while leaving policy semantics to the engine. The concrete missing terms are the checklist's own examples: usage restrictions such as time windows, re-delegation constraints, and depth limits. Cedar demonstrates that such a vocabulary can be expressive and formally analyzable at once. This is the gap where existing pieces come closest to composing, and arguably the highest-value item. Two alternatives to a global standard exist. A resource can treat its policy language as part of its API, so that a delegator learns the caveat vocabulary from the resource it is delegating access to. Some practitioners also report that natural-language caveats interpreted by an LLM work surprisingly well, trading determinism for flexibility.  
2. An advisory tier. No surveyed system can express a request whose violation is not refused ("please do not re-delegate outside the org"): every evaluation model in the survey is binary enforce/reject. Advisory semantics matter for agentic delegation because agents cross governance boundaries where hard enforcement is impossible but recorded intent still has value (for audit, for liability, for downstream policy).  
3. Agent-vs-operator semantics and identity binding. The capability family cannot say which principal is the accountable legal entity; the identity-centric systems can, but only inside their own trust domain. What is missing is a standardized companion mechanism (the VC-based binding the capability chapter gestures at) so that accountability to an operator survives cross-organizational, multi-hop delegation without collapsing back into identity-based authorization. KYA-OS is the first surveyed specification to supply one, by naming and recomputing a Responsible Party at the root of a zCap chain; whether its particular carrier (the capability wrapped in a Verifiable Credential, with the chain resolved from a registry) is the right shape for a cross-organizational standard is the open question the VC modeling chapter raises. AuthZEN's open Issue \#416, which requests a `delegation_chain` structure in the Subject for AI-agent scenarios, is this same gap filed independently by an adjacent working group. Hop-by-hop responsibility tracking is an alternative that needs no standard binding: each delegator vouches for the party it delegated to, and an auditor walks the chain one edge at a time. This works across domains that do not federate.  
4. Offline and decentralized revocation. Every surveyed revocation mechanism is state at an enforcement point that must be reached: an RS endpoint, a server's grant, a policy store. For the local-machine and LAN agentic use cases the checklist highlights, revocation semantics under partition are unspecified everywhere. UCAN's open dissemination model (gossip, DHT) is the closest starting point, but its guarantees are undefined. The dSD-JWT draft states the problem from the delegator's side: an individual holder has no channel to distribute revocation to verifiers, so it falls back to short expiry.  
5. Cross-organizational policy exchange. Even where evaluation is local, policy semantics are unilateral: Cedar's store belongs to one verifier, an AS's scope definitions to one provider, a PS's mission policy to one server. There is no standard by which one organization hands another a policy fragment with agreed meaning. AuthZEN's Policy Store draft is a partial answer: it standardizes the packaging and exchange of policy artifacts while explicitly leaving their semantics engine-specific, sharpening this gap without closing it. This is the cross-organizational criterion's unmet half, and it compounds gap 1: a shared caveat vocabulary is also the natural interchange format. Cross-organizational policy exchange is a hard problem in general. The pragmatic alternative, as under gap 1, is to make the policy language part of each application's API rather than a shared standard.

One further case was raised in review and is recorded here rather than scored. A delegation may be made to a target not yet known, for example by publishing it on a message bus for whichever agent picks up the task. UCANs and zCaps could carry the metadata such a pattern needs, but responsibility tracking becomes delicate when the delegator does not know who will act. Whether this deserves a criterion of its own is left for a later revision.

## Conclusion

The survey's answer to "which specification should agentic delegation use" is that the question cannot have a general answer. The certificate capability family supplies the only viable backbone for the authority flow itself as a first-class data element and throughline: it alone meets the chain criteria, offline and holder-side, and it alone is immune to the confused deputy by construction. But it is a backbone, not a solution. Its own chapters identify the missing semantics, and those semantics are exactly what the identity-centric and policy systems in this survey already know how to express. The near-term work is therefore not a seventh competing spec but composition.

Less of that composition needs global agreement than the gaps list might suggest. Identity binding at the chain's root can be handled hop by hop, with each delegator accountable for the party it delegated to. The decision interface and the revocation mechanism are each the concern of the resource's own verifier, which deals with its own certificates and is the only party that needs to learn of a revocation. What does benefit from shared agreement is the policy and caveat vocabulary that rides in the chain, since a delegator must be able to write a caveat that a verifier it has never met will interpret as intended. Even there, treating the policy language as part of each resource's API is a workable alternative. Revocation under partition remains an open design problem for every verifier regardless. The gaps list above is, in effect, the outline for that work.

## Appendix: Other Systems Considered

This appendix covers specifications that were examined during the survey but that did not warrant a full chapter, either because they address only one layer of the stack described in the Introduction or because they are too early in their standards lifecycle to evaluate on equal footing with the main candidates. Each entry is scored against the same checklist so that it can be placed alongside the merged scorecard in the Synthesis chapter.

### Delegated SD-JWT (dSD-JWT)

#### Overview

[Delegated SD-JWT](https://datatracker.ietf.org/doc/draft-gco-oauth-delegate-sd-jwt/) ("dSD-JWT", an individual IETF draft at revision 00) is an extension to [Selective Disclosure JWT (SD-JWT, RFC 9901)](https://datatracker.ietf.org/doc/rfc9901/) that lets the holder of an issued credential hand a bounded, verifiable authorization to another party without involving the issuer. Rather than defining a new credential type, it appends a delegation hop to an existing SD-JWT as an additional disclosure. Each hop is a JSON payload chosen by the delegator, wrapped in a Key Binding JWT signed by the key the previous hop named in its `cnf` claim, and hash-bound to the prior state of the credential. That is, the Key Binding JWT carries a hash of the credential as it stood before the hop (`sd_hash` or `issuer_jwt_hash`), so a hop cannot be detached, reordered, or replayed against a different chain. A hop may name a further `cnf` key, in which case the chain can continue, or omit it, in which case the chain terminates at that delegatee.

The draft is deliberately claim-agnostic. It defines the chaining and key-binding mechanics and says nothing about what the delegated claims mean. Vocabulary for agents, resources, actions, or limits is left to a higher-level profile. The one implementation examined for this appendix (the [equs-credentials-sdk](https://github.com/equs-ai/equs-credentials-sdk), where the feature is marked experimental and disabled by default) names the [Agent Payments Protocol (AP2)](https://ap2-protocol.org/) as the first such consumer, but that profile is not part of the draft and its dSD-JWT binding was not available for review.

This makes dSD-JWT a chainable proof method plus a possession mechanism, in the terminology of the Introduction, rather than a full authorization data model. It is closest in spirit to the certificate capability family, but its root of authority is a credential issuer rather than a resource owner, and it lacks the two properties that define that family in this report: an enforced attenuation rule and a resource and action model. For those reasons it is placed here rather than in a chapter of its own.

The draft covers issuance, delegation, and presentation to a verifier, the last by Key Binding JWT with the audience and nonce binding inherited from RFC 9901 and OpenID4VP. It defines no invocation protocol against a resource. How a verifier turns an accepted presentation into an action is outside the draft.

Reference points: the draft text (sections 4 through 8) and the SDK's design note, source, and tests. The SDK delegates chain verification to a separate crate that was not reviewed directly, so verdicts below rest on the current internet-draft (-00), the SDK call sites, and the SDK's test suite.

#### Scorecard

Each entry gives the requirement, the verdict, and the reasoning.

**1. Attenuated.** Verdict: No.

Neither the draft nor the implementation enforces narrowing. The delegate payload is arbitrary JSON, and the only validation rejects reserved keys. Attenuation is explicitly deferred to a higher-level profile.

**2. Accountable (agent vs. principal/operator).** Verdict: Partial.

Every hop is signed by the key the previous hop named, so each edge is attributable. But delegatees are identified by raw JWKs in `cnf`, not by DIDs or any stable identifier, and there is no agent or operator field. Weaker than zCaps and UCANs, whose principals are at least DIDs.

**3. Resistant to confused deputy.** Verdict: Partial.

Authority travels in the signed artifact and the verifier evaluates the chain rather than the presenter's standing. However, the artifact carries no designated resource or action, so whether designation and authorization are bound together depends entirely on the profile that defines the payload vocabulary.

**4. Represent authorization policies.** Verdict: No.

No caveat or policy slot. The only delegation control is structural: a hop that omits `cnf` ends the chain. Draft section 8.2 says a delegator "MAY" add constraints as visible claims, with no defined vocabulary (unclear if profiles define it, or if it is per-verifier/usecase). No advisory tier.

**5. Chainable.** Verdict: Yes.

Multi-hop, holder-side, each hop hash-bound to the prior state and rooted in the original issuer signature. Covers operator to agent and agent to sub-agent.

**6. Cross-organizational / locally verifiable.** Verdict: Yes.

Verification is local and needs no authorization server. The root issuer key is obtained through DID resolution or issuer metadata; hop keys are embedded JWKs. The format does not answer whether a given hop key is trusted for a given resource.

**7. Self-revocable.** Verdict: No.

Only Token Status List on the base credential, which revokes the whole tree at once. Draft section 8.3 acknowledges that an individual holder "can not easily distribute revocation information to Verifiers" and suggests a short `exp` as mitigation. No per-hop revocation and no revoker role.

**8. Authentication / Proof of Possession (nice-to-have).** Verdict: Yes.

Key Binding JWT at every hop, with `aud` and `nonce` binding at presentation. Not a bearer format.

#### Placement relative to the survey

Against the merged scorecard, dSD-JWT scores well on exactly the rows where the certificate capability family scores well (chainability, proof of possession), shares its no-round-trip delegation, and scores poorly on the rows that make that family a capability system (attenuation, resource and action model, policy vocabulary, revocation below the root). Two features are novel relative to the surveyed specs:

* Selective disclosure in the chain. A delegatee can hold several pre-signed alternatives and disclose only the one it uses. This is the closest any surveyed artifact comes to hiding chain interiors from a verifier, although it hides alternative payloads rather than intermediate parties (see Delegation Chain Privacy Considerations in the Introduction).  
* An explicit statement of the holder-side revocation problem (Gap 4). The draft's security considerations state directly that a holder, unlike an issuer, has no channel to distribute revocation, and fall back to short expiry. This is the offline-revocation gap named from the delegator's side.

If a profile for agentic delegation were layered on dSD-JWT, it would need to supply a resource and action vocabulary, an attenuation rule checked by the verifier at each hop, and a per-hop revocation mechanism. Those are the same items the Gaps list in the Synthesis chapter assigns to the certificate capability family, which suggests dSD-JWT is better understood as an alternative envelope for that family's semantics than as a competing model. Its main attractions over the DID-native envelopes are its fit with existing SD-JWT VC wallets and its inherited selective disclosure. The chapter Modeling Chained Delegation with W3C VCs sets out what such a profile would have to add, and compares dSD-JWT with KYA-OS, the other surveyed design that carries delegation in a credential.

### AuthZed/SpiceDB

#### Overview

AuthZed is an open-source permissions system that enforces access decisions at request time across an organization's data stores, backed by a "ReBAC" (Relationship-based Access Control) data store called SpiceDB, best understood as a directed relationship graph fit for complex graph-aware queries and wrapping simpler access control lists.
SpiceDB is, in turn, a kind of community-fork or open-source alternative to Google's internal Zanzibar system, powering the authorization of Google's many diverse integrated systems (GMail, Gdrive, Google Cloud IAM, and Youtube, for example), based on the original Zanzibar whitepaper.
It replaces simple access-control lists with a just-in-time and relationship-aware access-control graph.

While SpiceDB can still be categorized as an "access control list", its design goals were to be a superset of many different simpler access lists, and capable of expressing complex relationships and queries (like inheritance, tombstoning, etc) _across_ those multiple data models.
This "meta" relationship is crucial to the enterprise sales pitch, in that by being a flexible layer on top of heterogenous authorization and identity systems, it can abstract and reason over multiple of them in an enterprise context where data governance and relationships comes (inflexibly) along with the data of customers and partners.
It combines many separate access control lists into a single configurable permissions engine, so that graph queries can be run efficiently over one central store.

It is particularly noteworthy as an alternative to conventional ACLs for use in agentic systems because of [OpenAI publicly using and championing it](https://authzed.com/customers/openai), and for AuthZed's [RAG use case](https://authzed.com/use-cases/ai-retrieval-augmented-generation), whereby data governance inherited from training sets can dynamically be applied in training pipelines and (ideally) even preserved in their outputs.

#### References

* Gates, Carrie, [Access Control Requirements for Web 2.0 Security and Privacy](https://www.researchgate.net/profile/Carrie-Gates-2/publication/240787391_Access_Control_Requirements_for_Web_20_Security_and_Privacy/links/540e6f670cf2d8daaacd4adf/Access-Control-Requirements-for-Web-20-Security-and-Privacy.pdf), 2006.
* [Explainer for SpiceDB relationship syntax](https://authzed.com/docs/spicedb/concepts/relationships#relationship-syntax)
* [Explainer for SpiceDB relationship-caveat syntax](https://authzed.com/docs/spicedb/concepts/caveats#writing-relationships-with-caveats-and-context)

#### Scorecard

Each entry gives the requirement, the verdict, and the reasoning.

**1. Attenuated.** Verdict: Partial.

Caveats expressable in the AuthZed DSL can be added each time permissions get assigned/"delegated", but these can also be removed (i.e. the total permissions escalated) by later hops as well.

**2. Accountable (agent vs. principal/operator).** Verdict: Partial.

Agent<>principal relationships CAN be encoded in SpiceDB, but this is more of a userspace/configuration option than an explicit, first-order distinction

**3. Resistant to confused deputy.** Verdict: No.

Arguably, a richer vocabulary for authentication-based decisions and a centralized ACL just incentivizes impersonation or credential-sharing further, and makes them harder to detect from the (cross-service) logs and behaviors observed.

**4. Represent authorization policies.** Verdict: Yes.

Caveats can be imposed on permissions, or on the underlying relationships themselves, in an AuthZed-specific DSL. No advisory tier for these.

**5. Chainable.** Verdict: Partial.

No direct or explicit chaining of permissions, although permissions can be delegated to groups whose members change (and new members can add new members in turn) so something like chainable permissions can be effected implicitly.

**6. Cross-organizational / locally verifiable.** Verdict: No.

Delegation and invocation both require live access to the authoritative SpiceDB server.

**7. Self-revocable.** Verdict: No.

User can be permissioned to remove themselves from groups, but this has no effect on, e.g., other members they added to those groups.

**8. Authentication / Proof of Possession (nice-to-have).** Verdict: Partial.

How actors are authenticated to their corresponding entries in the SpiceDB directory is out-of-scope of the specification, can be possesion-based or not.

#### Placement relative to the survey

AuthZed/Zanzibar-based approached are mentioned here not because they _are_ an authority-chaining system, but because it could easily be mistaken for one (or sold as one) using a definition of delegation other than that used in this paper.
Since all authorization decisions are made relative to a relationship graph, adding relationships or adding members to collections could be called a "delegation" of authority (from the collection to the new member, or from one end of the new relationship to the other).
That said, while capability-certificate systems make very **explicit** (and atomic) the delegation of authority, "delegation" by updating a relationship graph is exactly the opposite: implicit and easily incurring side-effects for OTHER authorities or permission-sets, far from atomic except where heavily caveated.
As one of the guiding design goals of this report has been minimizing "ambient authority" (and thus confused-deputy potential), it is worth noting that centralizing all authorization decisions to a _single_ relationship graph governed independently of all the permissions and resources scoped more narrowly than that graph might well be the penultimate "ambient authority" situation!

While the _identities_ against which authorization decisions are made (and against which actors are authenticated) are strictly pegged to the SpiceDB layer, the delegations and permissions themselves (encoded in the AuthZed layer) _do_ allow rich and expressive caveats.
The fairly expressive [domain-specific language for caveats](https://authzed.com/docs/spicedb/concepts/caveats#defining-caveats) provided in AuthZed for attenuating permissions (_or for attentuating [the relationships themselves](https://authzed.com/docs/spicedb/concepts/caveats#allowing-caveats-on-relations)!_) puts the attenuation logic closer to the delegation than many of the other protocols we've evaluated here, and allows these attenuations to travel across the systems unified by the overarching ReBAC abstraction.

AuthZed was not added to the merged scorecard in the Synthesis chapter because that scorecard already covers so many extensions to the ACL+OAuth model, but this section was added for the sake of completeness.

### Relationship to Stiegler's Seven Aspects of Sharing

The evaluation checklist in the Introduction largely tracks Stiegler's seven aspects of sharing (see the [Delegated Authority Problem-space report](https://identity.foundation/delegated-authority-report)), with the following adaptations for spec evaluation:

* Composability is not listed as a separate criterion, since any capability system satisfies it trivially, so it does not discriminate among the candidates under evaluation.
* Functional resistance to confused deputy attacks (item 3) replaces the more formal composability, as it proved the more useful rubric.
* An additional requirement that authorization *policies* be representable (at least by reference) in the specification (item 4) was added, as a practical necessity in enterprise and large-scale deployments.
* Stiegler's "dynamic" aspect (delegation without undue delay) is not scored as a criterion. It is covered by the per-stage reachability analysis under Offline Use Considerations in the Introduction.
