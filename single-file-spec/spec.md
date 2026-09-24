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

Why agents specifically? Conventional access control was already strained by ordinary human sharing, but LLM-based agents completely break its assumptions. Agents are often short-lived, sometimes existing for only seconds, therefore provisioning each agent with accounts or ACL entries doesn't scale, and an agent's own identifier can never be the responsible party. Responsibility has to trace back to the principal operator who gave it the task. Agents can't be trusted to keep secrets (a clever prompt may extract a credential), and, unlike humans, they can't be deterred by post-facto punishments or personal consequences. In cases where conventional software has a defined API, an agent's behavior is effectively unbounded: the permissions it will need can't be fully known in advance, making "least privilege" a dynamic problem rather than a provisioning-time one. These properties push the authors toward forms of authorization that are delegated, attenuated, and verifiable at request time, rather than derived  from an identity. Agentic AI’s fundamental risks suggest that any proof-of-possession keys must be held outside the agent itself (by a more deterministic component trusted to use them on the agent's behalf).

To address the problem of Agentic authorization and delegation holistically, we assessed specifications addressing various authorization layers. Their strengths and weaknesses are addressed primarily against their scope and goals, with an eye to their various possible recombinations and configurations. A given spec may address one or several of the following:

* Data model: how the various mandatory and optional data elements of an authorization get expressed.  
* Protocol: how tokens are requested, presented, and (optionally) revoked.  
* Possession mechanism: how tokens are secured, e.g. through some proof of possession (and specifically protocol for agentic management of keys and token, which are sometimes profiled externally to the specification itself).  
* Chainable proof methods: how a multi-hop chain of delegations is expressed and verified.


Delegation is central to the problem of agentic authorization because the only alternative to delegation is impersonation. Delegation can be done without capabilities, using identity-based (authentication-centric) systems, but that entails some degree of  ambient authority, which opens the door to “confused deputy” attacks, whereby too much faith is placed in fallible (or second-hand) authentication. Cedar and similar systems illustrate the point: they support more flexible delegation while remaining identity-based, so they remain subject to the confused deputy problem, and they also place policy evaluation on the critical path of every request, incentivizing attacks on the identity layer rather than the policy layer. The same applies one layer down, at the decision protocol. AuthZEN standardizes how an enforcement point asks a policy decision point for a verdict, but the authority consulted still lives at that server and its trust model is still based on  identity.

A quick note on the [confused deputy](https://en.wikipedia.org/wiki/Confused_deputy_problem) problem, which was central to our motivation and inquiry here. Typically, people refer to the slippage between logged identities and actual-executing, actually-liable parties as a confused-deputy situation, but we took a more existential approach: we think of impersonation and credential-sharing and exceptions to rigid logging or access policies as invitations, which make authentication and credential data the weakest links. Agents are nothing if not expert weakest-link finders\! To foreclose the possibility of confused deputies, we maintain, designation and authorization must be combined, explicitly stating which actions are allowed to be performed on which resource, verb, or API endpoint, rather than leaving that to be solved indirect via authentication and policy. This combination is definitive of a family of approaches generally grouped under the category of "capability systems", explicitly formalizing capabilities, as opposed to systems relying on ambient authority (e.g. assigning authority by identity and/or role).

Capability systems can be categorized by which parts of the system do the enforcement. The main types are:

* hardware capabilities (the oldest form)  
* object references enforced by the platform (which require a memory-safe language to be securable at scale)  
* opaque tokens  
* certificates  
* cryptographic-only (encryption-based) systems.

In certificate systems, metadata is bundled *with* the permission and presented together with it; in opaque token, platform, and hardware systems, the metadata is stored somewhere else and merely referenced. Certificate systems themselves come in two types: bearer tokens and proof of key possession.

Multi-hop (chainable) delegation is important. Without it, after a "single hop" delegation (traditional OAuth2), the system lends itself to impersonation (for example, copy-and-pasting an OAuth2 token into an agent’s prompt or context window for it to act as you in the system). In this increasingly common scenario, the audit trail is degraded, and it’s not possible to revoke the agent’s privilege without revoking the human’s privilege as well. At scale, this is an untenable workaround.

Attenuation, or the ability to delegate a smaller scope of capabilities, is an essential part of the picture. Delegation should enforce the **principle of least privilege,** giving an actor working on your behalf as little privilege (and incurring as little risk) as necessary. When enlisting help from an agent, the user should give it only the minimum permissions necessary to perform the task. Attenuation enforced by the capability system itself prevents escalation attacks, which were less frequent and existential a risk when enterprise-internal systems were only being used by human employees.

Traditional capability systems are intentionally not identity-based, and in many cases can be configured to be almost identity-blind. For agentic use cases, however, it’s important to  constrain identifier choices (and maintain any maps or indirections necessary) so as to maximize accountability and logging, *particularly* because agentic use-cases involve so many liable parties, with non-determinism and risk distributed across so many components. Complicating identifier choices and classical best practices are the inherent challenges of cross-domain, cross-organizational identifiers, since agents often perform tasks across different web domains, trust zones, and execution contexts, which may in turn require partial confidentiality or bring their own logging and authentication requirements. Given all of the above, we take the strong stance that bearer tokens are dangerous to use at any layer of such a system, due to the ease of replay attacks and their tendency to appear in logs; luckily, this position is becoming [less controversial](https://www.ietf.org/archive/id/draft-klrc-aiagent-auth-03.html#section-7) over time across the security and identity communities, as more and more research and security incidents indicate the near- or total impossibility of trusting frontier models with such systems.

### Specification Evaluation Checklist

In this report we evaluate major authorization and/or access-control specifications that address any of the design questions listed in the Introduction: primarily, we study how delegation is supported by the data models, invocation protocols, and chainable proof methods, but also other aspects of protocol and logging are addressed as well. Validity checks must be done at the time of request (by a component trusted by the resource provider), and there is much room for configuration and profiling across all these systems, but we have focused on the core affordances of these specifications at a high-level.

The assessment criteria largely track Stiegler's seven aspects of sharing (see the [Delegated Authority Problem-space report](https://identity.foundation/delegated-authority-report)), with a few adaptations for spec evaluation. The adaptations are:

* Composability is not listed as a separate criterion, since any capability system satisfies it trivially, so it does not discriminate among the candidates under evaluation;   
* Functional resistance to confused deputy attacks (item 2\) replaces the more formal composability, as it was proved the more useful rubric, \]  
* An additional requirement that authorization *policies* be representable (at least by reference) in the specification (item 3), which is a practical necessity in enterprise and large-scale deployments, and  
* Stiegler's "dynamic" aspect (delegation without undue delay) is relaxed from a requirement to the offline-capable nice-to-have below.

For this report, the final list of assessment criteria is:

1. **Accountability:** Specification must allow stable, in-band differentiation between the agent and its principal operator (e.g., natural or legal person)  
2. **Confused-deputy Resistance:** relative infeasibility of impersonation and authentication attacks,  mostly from reducing “ambient authority.”  
3. R**epresentability of authorization policies** (either by embedding or linking to them)   
   1. This includes **functional policies** like "do not delegate further" or “this permission is non-delegatable”  
   2. This includes **re-delegation policies** like “only delegate within X boundary or role”  
   3. "**Please**": A request containing desired rules for redelegating but not refusing to honor violations.   
   4. "**Disclosure** **governance**": Confidentiality of tokens and logs is not just configurable but governable.  
4. **Chainability**: Authorization must be able to be delegated: At root, authorization is delegated from principal/operator to agent. But also, agents must be able to delegate to further subsystems or other agents.  
5. **Cross-organizational execution** (validation checks are independently verifiable, locally verifiable). At minimum, different organizations/groups should be able to express authorization policies without having accounts on each other's IAM systems.  
6. **Attenuated delegation**: delegation of a subset of permissions as a first-class data element.  
7. **Self-revocability:** The capability holder, somebody in the delegation chain, or a holder of an explicit revocation capability, must be able to revoke a delegated permission, modulo network partitions.

**Not required but nice to have:**

1. **Authentication grounded in first-class, in-band Proof of Possession**   
2. **Privacy of Delegation chain:** Can the system hide some or all the delegation data and metadata in the proof chain from agent / token carrier, and/or from intermediate counterparties?   
3.  **Offline-capable:** Create tokens offline, delegate offline, use/present offline. Agentic Use cases: most of them online, but some are offline-capable to (local machine, or local LAN only agentic use cases). Also, offline-capable parts of the spec can offer efficiency advantages. For example, if it’s possible to  delegate without having to contact the resource server, there’s a significant speed advantage.

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
* "revocation"—how a permission is revoked (mostly protocol, though revocation has a data model aspect as well, such as pointing to where revocation status is checked).  
* "invocation" (separate protocol and data model)—how to invoke a permission for a given resource.  
* cryptographic suites (if applicable)—how to prove cryptographic key possession for delegation and invocation.


A table mapping each spec's concrete field names onto this common shape appears in the Synthesis chapter, under Data Model Correspondence.

### A Note on Terminology

To discuss delegation, we need to at least have names for the parties involved. We use the following terms:

* Delegat**or**:The party who is *doing* the delegating. The source of the authorization. This one is easy, most texts agree on this one.  
* Delegat**ee**:The party who *is being delegated to*. The target of the delegation. This term is harder. Some texts also refer to this party as a “delegate” (noun). This  makes sense, but in written form, it's easy to confuse with 'delegate' (verb). Some texts even use “delegee” for this party. For ease of use, this report standardizes on 'delegat**ee**'.


Of course, these roles are transitive, and a delegatee becomes in turn the delegator the moment they sub-delegate whatever permission or authorization they have to yet another party in the chain.

## OAuth Family Evaluation

_Hardening, Consolidation, and the Push Toward Autonomous Authorization_

This chapter covers what is actually deployed today—OAuth 2.0 (RFC 6749/6750), as hardened by RFC 9700 (2025 Security Best Current Practice)—then traces the evolution toward OAuth 2.1 and, separately, the independent AAuth draft.

Each section below opens with an italicized tag, one of:

* Deployed standard


* Adjacent RFC/profile


* OAuth 2.1—consolidation draft


* AAuth—independent draft


These standards clarify the most common solution in use for  Google, Okta, Auth0, GitHub, or a typical enterprise IdP today, versus what's still in development and limited/niche deployments.

### Overview

OAuth (Open Authorization) is an open standard for **access delegation** anchored in HTTP calls, session state, and redirects. It allows users to grant third-party applications secure, scoped access to their data hosted on another service without ever handing over their passwords. Much like a "valet key" for a car, which lets a valet drive and park the vehicle but prevents them from unlocking the glovebox or trunk, OAuth uses limited-scope, session-bound **Access Tokens** rather than shared passwords or other static, long-lived credentials to protect user data.

The standard running in production today—what every major identity provider implements for most clients at scale— is OAuth 2.0: RFC 6749 (the core authorization framework) and RFC 6750 (bearer token usage), both from 2012, with additional security on higher layers. OAuth 2.0 is very much a framework, not a fixed protocol: it deliberately leaves many security-relevant decisions to implementers, architects, and configuration. This flexibility is precisely why a decade of real-world attacks produced a long list of follow-on RFCs tightening specific gaps (PKCE, JWT access tokens, token exchange, sender-constraining, DPoP, and so on).

In January 2025, the IETF published RFC 9700, "Best Current Practice for OAuth 2.0 Security," which consolidates that decade of lessons into concrete guidance: don't use the Implicit grant, don't use Resource Owner Password Credentials, always use PKCE where possible, match redirect URIs exactly, and prefer sender-constrained tokens. RFC 9700 is guidance on top of RFC 6749/6750. It doesn't supersede or deprecate them.

OAuth 2.1 (draft-ietf-oauth-v2-1) is the next step, versioning the protocol rather than just profiling it. It is an attempt to fold RFC 9700's hardening directly into a single normative document that deprecates RFC 6749/6750. As of this writing, it is still an Internet-Draft in its [seventh year of iteration](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/). "OAuth 2.1" is best understood as the direction the ecosystem is consolidating toward, with most major 2.0 deployments largely compliant with minor changes, not a separately deployed protocol version.

Separately, AAuth (draft-hardt-oauth-aauth-protocol) is an independent, early-stage draft built specifically for agent-to-resource access, designed to coexist with OAuth and OIDC, not to extend either.

### What's Actually Deployed: OAuth 2.0 plus RFC 9700

#### The Core Framework

*Deployed standard.* RFC 6749 defines the four classic grant types (Authorization Code, Implicit, Resource Owner Password Credentials, Client Credentials) plus Refresh Tokens. RFC 6750 defines how bearer tokens are presented to a resource server. Some form of Client Registration is [assumed but out of scope](https://datatracker.ietf.org/doc/html/rfc6749#section-2) of RFC 6749; many later RFCs extend alternate mechanism. This is the substrate nearly every production OAuth deployment is built on.

#### RFC 9700: The Hardening Layer in Force Today

*Deployed standard / BCP.* RFC 9700 is the document that defines  a secure OAuth 2.0 deployment.

RFC 9700 mandates the following:

* Don't use the Implicit grant (response\_type=token). Tokens returned in a URL fragment are exposed to browser history, referrer leakage, and any script on the page.


* Don't use Resource Owner Password Credentials. Password-based delegation trains users to type credentials into third-party UIs and gives the client raw password access.


* Use PKCE (RFC 7636\) on the Authorization Code flow for all clients, not just public ones. PKCE it closes authorization-code-interception attacks that affect confidential clients too.


* Match redirect URIs exactly; no substring or wildcard matching.


* Prefer sender-constrained tokens (DPoP or mutual TLS) over plain bearer tokens where feasible.


* Use state for CSRF protection and PKCE's code\_verifier together. Each protect against different attacks.


None of this requires specific use of "OAuth 2.1" as a separate thing to implement. OAuth 2.1's contribution is mostly editorial consolidation: combining these BCP recommendations and making the safe behaviors the only correct specified behaviors.

#### Client Authentication: Required vs. Common Practice

*Deployed standard / Adjacent RFC.* RFC 6749 only requires that confidential clients authenticate to the token endpoint somehow. client\_secret\_basic and client\_secret\_post are the baseline methods, widely used today.

Asymmetric client authentication has become common best practice for higher-assurance deployments:

* private\_key\_jwt (RFC 7523). The client signs an assertion with its own private key instead of sending a shared secret.


* Mutual TLS client authentication (RFC 8705). The client authenticates via its TLS certificate.


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

*Deployed standard / Adjacent RFC.* RFC 7009 allows a client or resource owner request explicit revocation. Short-lived access tokens provide a passive backstop. OAuth 2.0 has no native notion of delegation-chain revocation, because it has no native delegation chain in the first place. Delegation semantics are difficult to expose to the user, but delegations can be tracked across the logs of the Authorization Server.

### OAuth 2.1 : Where the Consolidation Is Heading

*OAuth 2.1 consolidation draft.* OAuth 2.1 (draft-ietf-oauth-v2-1) takes everything in Part 1's hardening layer and includes it in the core spec.

Key changes in OAuth 2.1 vs. today's hardened OAuth 2.0:

* Implicit and ROPC grants are removed from the spec text, not just discouraged.


* PKCE is required, not recommended, for every client on the Authorization Code flow.


* Redirect URI exact-matching is specified, not left as guidance.


* Sender-constrained tokens (DPoP/mTLS) remain a SHOULD, not a MUST. OAuth 2.1 does not mandate proof-of-possession tokens.


* Client secrets are still supported. The draft explicitly requires AS support for clients sending a plain client\_secret. Asymmetric client authentication is not the OAuth 2.1 baseline.


A well-hardened, RFC 9700-compliant OAuth 2.0 deployment today is very close to "OAuth 2.1 compliant". OAuth 2.1's main value is editorial: a single document new implementers can build against without needing to piece together a  decade’s worth of separate BCP RFCs.

### AAuth: A Separate, Independent Draft for Agent-to-Resource Access

*AAuth*—*independent draft.* Caveat: this is a single-author Internet-Draft, not an IETF working-group product. It has already been renamed and restructured more than once. Treat the below as a snapshot of a moving target.

#### Why It Exists

AAuth (draft-hardt-oauth-aauth-protocol) was created by an OAuth 2.0/2.1 co-author who concluded that OAuth, in any version, is a poor fit for agent-to-resource access because:

* Agents need authorization decisions made mid-task against resources they've never interacted with before.


* Agents have no durable client\_id that travels across organizations the way OAuth assumes.


* Copying long-lived API keys into autonomous workloads recreates the shared-secret leak surface OAuth 2.0 and 2.1 are trying to move away from.


AAuth is explicitly designed to coexist with OAuth 2.0 and OpenID Connect, not extend either one.

#### Architecture

Roles (See [Terminology](https://www.ietf.org/archive/id/draft-hardt-oauth-aauth-protocol-10.html#section-3), in newest draft):

* Person/Principal  
* Agent Provider (AP): The Agent Provider is a centralized entity which controls which manages agent identity and issues agent tokens, providing centralized governance over a distributed agent fleet.  
* Agent  
* Person Server (PS): server that represents the principal to rest of the protocol. The person chooses their PS; it is not imposed by any other party. The PS manages missions, handles consent, asserts user identity, and brokers authorization on behalf of agents.  
* Access Server (AS): A policy engine that evaluates token requests, applies resource policy, and issues auth tokens on behalf of a resource.  
* Resource


Token types:

* `agent_token` establishes the agent's identity


* `resource_token` describes the access a resource needs


* `auth_token` grants access to a specific resource, where applicable


Core mechanism: Signed HTTP requests using HTTP Message Signatures, not bearer tokens trusted on presentation. The simplest mode requires no token issuance flow at all.

#### Four Resource Access Modes

AAuth defines four modes, increasing in complexity:

* Identity-based access: The agent signs requests with its agent\_token. The resource verifies the signature and applies its own access policy. No authorization flow, no additional tokens. This replaces API keys with cryptographic identity.


* Resource-managed access (two-party): The resource handles authorization itself, optionally returning an opaque token for subsequent calls.


* PS-asserted access (three-party): A Person Server vouches for claims about the person on whose behalf the agent acts, with a consent step and an auth\_token issued on approval.


* Federated access (four-party): Extends the three-party model across an additional trust boundary.


Key discovery: Agents and resources discover signing keys via well-known metadata and JWKS-style endpoints, similar in spirit to OAuth's JWKS discovery, but used for HTTP-signature key lookup, not bearer-token verification.

### Part 4: Comparative Scorecard

*Note: A comparative scorecard placing OAuth 2.0 and AAuth alongside the capability systems previously appeared here. It has been superseded by the merged scorecard in the Synthesis chapter, which derives its verdicts from each specification's own chapter.*

### Closing Note

In production today, almost all systems are using OAuth 2.0, hardened to RFC 9700's recommendations, with client secrets still alive and well at most token endpoints and bearer tokens still the default token type. OAuth 2.1 is where that hardening is headed as a single normative document. However, the changes are fairly minor. It doesn’t mandate PoP nor does it eliminate client secrets.

AAuth, meanwhile, isn't a future version of OAuth at all. It's a parallel, signature-based protocol built because its author judged some core assumptions of the OAuth model itself a poor match for autonomous agents. AAuth, however, pushes complexity and risk into a fiduciary agent. This super-agent manages delegations, token-swaps, and other intermediary security and trust decisions. While AAuth ceremonies are different than those in OAuth, most of the token formats are lifted directly from OAuth, making it more of a logical variant than a complete overhaul of OAuth. UCAN/zCAP-LD remain the only one of the three with native, offline, multi-hop delegation \-- the clearest architectural line separating capability systems from the many HTTP Servers defining both the OAuth lineage and its AAuth variant (which adds a third Server).

## AAuth Evaluation

### Overview

AAuth is an emerging authorization architecture designed for authenticated interactions among people, AI agents, resources, Person Servers (PS), and Agent Providers (AP). Rather than treating software agents as extensions of end users, AAuth introduces authenticated agents as first-class participants capable of invoking protected resources, receiving asynchronous events, and operating across organizational boundaries. Crucially, the “container” for these cross-organizational distributed transactions is called a “mission” (dovetailing with [other work](https://notes.karlmcguinness.com/mission-handbook/) by OAuth Working Group insiders). While the previous chapter touched on AAuth in terms of its common assumptions and mechanisms, assessing AAuth as a bespoke protocol for managing and auditing agentic distributed transactions is easier treating it in isolation.

Unlike capability-based delegation systems such as UCAN and zCaps, AAuth does not represent delegated authority as a self-contained, cryptographically attenuated artifact carried entirely within a delegated credential. Instead, delegation is server-mediated. Authority is anchored within the Person Server, with delegation history represented through the act claim, Mission References, and the Mission Log maintained by the Person Server. AAuth therefore defines explicit delegation mechanics while intentionally separating delegation state from portable capability objects.

AAuth is evaluated here as a server-mediated delegation architecture. It authenticates participants, establishes trusted execution between them, records delegation history, and supports authenticated event-driven execution. Questions concerning delegation semantics, attenuation policy, authority evolution, and the policy logic behind execution-time decisions remain outside the protocol itself and are expected to be supplied by complementary architectural layers.

Three reference points are used:

*  [draft-hardt-oauth-aauth-protocol-10](https://datatracker.ietf.org/doc/html/draft-hardt-oauth-aauth-protocol-10) (6 August 2026), together with [draft-hardt-httpbis-signature-key-08](https://datatracker.ietf.org/doc/html/draft-hardt-httpbis-signature-key-08), the companion specification to which AAuth defers key conveyance, discovery, and algorithm determination;


* The AAuth [Events draft](https://dickhardt.github.io/AAuth/draft-hardt-aauth-events.html), which extends the architecture with authenticated asynchronous event delivery between resources and agents through Agent Providers; and


* public implementation discussions describing the intended architecture and deployment model.


Where protocol specification and implementation practices differ, this distinction is noted throughout the evaluation.

A note on method: This chapter was drafted with AI research collaborators under a gated review process, with final editorial authority held by the author. Load-bearing claims were verified directly against the AAuth specification repository rather than model recall — including the token-exchange question, the act claim's provenance, credential key-binding, and the Appendix B.3.7 attenuation rationale.

### Scorecard

#### Accountable (agent vs. principal/operator)

Verdict: Yes

AAuth distinguishes authenticated people, agents, Person Servers, Agent Providers, and resources. Delegation history records acting parties through the act claim.

#### Resistant to confused deputy (no ambient authority)

Verdict: Partial

The resource-managed, PS-asserted, and federated modes bind an authorization to a designated resource and scope through the resource token, combining designation with authorization. The **optional** account parameter added in draft-10 narrows that binding further, naming which account at the resource the authorization covers and propagating through the resource token into the auth token. Identity-based access does not do this: the resource authorizes on the agent's identity alone and applies its own access control, which is ambient authority.

#### Represent authorization policies

Verdict: Partial

Authorization context and Mission References are represented, but expressive delegation policy—for example, undelegatable authority, advisory policy, or attenuation constraints—is intentionally out of scope of the protocol, or inside the logic and authority of the Person Server.

#### Chainable

Verdict: Yes

Delegation chains are represented through the act claim and the Mission Log. Chains are server-mediated rather than portable capability artifacts. Sub-agent nesting is single-level; deeper workflows use chained top-level agents, each an independent principal holding its own grant.

#### Cross-organizational / locally verifiable

Verdict: Yes

Designed for authenticated interactions across organizational boundaries while preserving local trust relationships.

#### Attenuated

Verdict: Partial — differs by mode

Sub-agent authorization supports attenuation: every request passes through the parent, which can refuse, attenuate, or rate-limit. Call chaining does not — downstream authorization is intentionally not required to be a subset of upstream scope. Cryptographically enforced attenuation is not intrinsic to the protocol in either case.

#### Self-revocable

Verdict: Partial

Token revocation, mission revocation, and propagation through a parent's grant are all specified. Revocation is exercised by the servers holding authority rather than by a holder acting on a credential in hand, and mission state administration beyond completion is deferred to a companion specification.

#### Authentication / Proof of Possession

Verdict: Yes

Core capability of the protocol. `draft-10` requires a fully-specified alg identifier, recommends Ed25519, and prohibits none, symmetric algorithms, and the polymorphic EdDSA identifier that RFC 9864 deprecated.

#### Privacy of Delegation Chain

Verdict: Partial

Delegation history is maintained by the Person Server rather than being universally disclosed through portable credentials. The protocol records delegation continuity while allowing deployment architectures to determine how much of that history is exposed during execution.

#### Offline Capable

Verdict: Partial

Implementations MUST cache JWKS and SHOULD continue verifying against cached keys when a fetch fails, bounded by a cache lifetime of at most 24 hours, so token verification survives temporary loss of contact with an issuer. Obtaining authority remains online-only: AAuth's server-mediated model provides no offline delegation, and initial key discovery requires reachable metadata.

### Detailed Evaluation

#### Accountable (agent vs. principal/operator)

AAuth explicitly distinguishes among authenticated people, AI agents, Person Servers, Agent Providers, and protected resources. Rather than treating software agents merely as extensions of user sessions, the protocol gives each participant its own authenticated identity and architectural role.

Delegated actions are represented through the act claim, allowing delegation history to be maintained across direct delegation, sub-agent delegation, and chained execution. Mission References further associate requests with the governing mission maintained by the Person Server.

Unlike certificate capability-based systems, accountability derives from authenticated protocol exchanges combined with server-maintained mission history rather than from cryptographically self-contained delegation credentials.

The specification also permits a resource to authorize an agent based solely on its identity, without interaction or mission context; working-group discussion has flagged that identity-only authorization reintroduces confused-deputy exposure, since the execution context rather than the requester's identity is what validates an action.

#### Resistant to confused deputy (no ambient authority)

The confused deputy problem arises where a party exercises its own permissions on a resource designated by someone else. The defence is to combine designation with authorization, so that what may be done is bound to what it may be done to.

In the resource-managed, PS-asserted, and federated modes AAuth does this. The agent obtains a resource token naming the resource and scope; the auth token issued against it carries that binding, and the resource enforces it. The OPTIONAL account parameter, added in \-10, narrows the binding further: it names which account at the resource the authorization is for, drawn from the resource's own namespace, and is echoed through the resource token into the auth token, so an auth token for one account grants nothing at another.

Identity-based access is the exception, and a deliberate one. The agent signs requests with its agent token, and the resource applies its own access control based on who the agent is. There is no authorization flow and no designation carried with the request. The specification presents this as a replacement for API keys, which it is; but authority derived from identity alone is ambient, and an agent acting on a request supplied by another party has no way to signal that the request is not its own.

#### Ability to Represent Authorization Policies

AAuth represents authorization context through missions, authenticated requests, and protocol-defined execution relationships. Mission References bind requests to an existing mission while allowing authorization policy to remain under the control of the Person Server.

The protocol intentionally does not standardize a rich delegation policy language. Concepts such as undelegatable permissions, advisory delegation preferences, attenuation constraints, or policy composition remain responsibilities of external policy engines or governance frameworks.

This separation allows AAuth to remain focused on authenticated execution while supporting a wide variety of higher-level authorization models.

#### Chainable

AAuth provides explicit delegation mechanics through the act claim and the Mission Log. These mechanisms record delegation history across direct delegation, chained delegation, and sub-agent authorization.

Unlike certificate capability-based systems, delegation history is maintained by the Person Server rather than embedded entirely within a portable delegation credential. Verification therefore depends upon authenticated interaction with the server maintaining mission state rather than validating an independently portable capability chain.

Accordingly, AAuth supports delegation chains, but those chains are server-mediated rather than self-contained cryptographic artifacts.

Sub-agent nesting is limited to a single level: a sub-agent must not have sub-agents of its own, and the specification enforces this from both directions. Deeper workflows are carried instead by call chaining, where each hop is an independent top-level agent holding its own grant rather than a recursive sub-agent — which is also why upstream-subset rules do not apply to it. What the specification defers to a companion document is mission state administration beyond completion: revocation state transitions, delegation-tree queries, and administrative interfaces. Delegation-chain lifecycle is not deferred; sub-agent revocation propagates through the parent's grant.

#### Cross-Organizational / Locally Verifiable

Supporting authenticated interactions across organizational boundaries is one of AAuth's principal architectural objectives.

Person Servers, Agent Providers, agents, and protected resources may participate across independent trust domains while preserving authenticated relationships between participants. Authentication evidence can therefore span organizations without requiring centralized identity management.

Trust remains anchored in authenticated protocol interactions and configured trust relationships rather than portable authorization credentials.

From a capability-based perspective, the need for Access Server federation may indicate that authority remains server-mediated rather than fully represented by the delegation credential itself. If the delegation credential carried sufficient authority for cross-domain verification, no ongoing relationship between authorization servers would be required.

#### Attenuated

Attenuation in AAuth differs by delegation mode. Sub-agent authorization supports it directly: every sub-agent request passes through the parent, which can refuse, attenuate, or rate-limit. Call chaining, by design, does not support delegation.  Downstream authorization is intentionally not required to be a subset of upstream scope, and downstream scope is constrained by the downstream resource's own policy together with Person Server evaluation against mission context.

Unlike UCAN or zCaps, attenuation is not represented as a cryptographically enforced property of a portable delegation artifact. Instead, attenuation depends upon server-managed mission state and policy evaluation.

The protocol therefore leaves attenuation semantics outside its definition, supplying contextual evaluation in place of algebraic constraint.

This is deliberate design rather than omission. AAuth does not adopt the RFC 8693 token-exchange model for attenuation (it borrows only that RFC's act claim structure for representing delegation chains), and Appendix B.3.7 states the rationale directly: downstream scope is intentionally not required to be a subset of upstream scope, because "the PS evaluates each hop against the mission context, providing governance-based constraints... more flexible than algebraic attenuation rules." The one place attenuation exists within an agent's own purview is sub-agent authorization, where the parent "can refuse, attenuate, or rate-limit" the sub-agent's authority.

#### Self-Revocable

AAuth includes mechanisms for token revocation and lifecycle management.

\-10 specifies named revocation scenarios: a Person Server revoking an auth token it issued or provided, an Access Server revoking one it issued, an agent provider revoking an agent token, and a Person Server revoking a mission, after which subsequent token requests referencing that mission are denied. Revocation also propagates structurally. Revoking a parent's grant causes the next sub-agent authorization to fail, and tokens already issued expire within the hour.

The specification is candid about the limits. Verifying an auth token does not consult its issuer, so nothing in the verification path reports that a token has been revoked; a party that no revocation request reaches is bounded only by token lifetime. What remains unspecified is mission state administration beyond completion, deferred to a companion specification. Revocation is therefore exercised by the servers holding authority rather than by a holder acting on a credential in hand.

### Summary

AAuth represents a distinct approach to delegated authorization for AI systems. Rather than expressing delegated authority as portable cryptographic capabilities, it anchors authority within the Person Server and maintains delegation continuity through authenticated protocol exchanges, mission state, and recorded delegation history. In the taxonomy of capability models, AAuth authorizations are neither bearer capabilities nor certificate capabilities: they are server-mediated proof-of-possession credentials, and the specification assigns multi-hop safety to mission-context evaluation rather than to algebraic attenuation, expecting complementary layers to supply the latter where required. Delegation in AAuth references stable agent identifiers rather than individual keys. The cnf claim binds each authorization token to the agent's current signing key while the identifier persists across key rotation. This reduces coupling between delegation history and key lifetime, but introduces lifecycle questions concerning delegation validity across key rotations—whether a delegation was issued before or after a rotation, and how invocations arriving after should be treated. Capability systems using delegation-specific or one-time keys largely avoid this issue, since delegated credentials typically expire before key rotation becomes relevant.

Its strengths lie in authenticated interoperability, cross-organizational execution, event-driven interaction, and maintaining coherent delegation history throughout mission execution. Unlike capability-based systems, authority remains associated with server-managed mission state rather than being embodied entirely within portable delegation artifacts.

This architectural choice intentionally separates authenticated execution from higher-level governance concerns. Authentication and delegation establish who is requesting execution and under what delegated authority. The Permission Endpoint now provides a normative execution-time decision surface: an agent may ask whether a specific action is permitted and receive granted or denied, recordable in the mission log. What remains outside the protocol's scope is the policy logic that produces that answer, together with attenuation policy, authority evolution, mission governance, and dynamic authority state.

Accordingly, AAuth is best understood as a server-mediated delegation and execution framework. It provides authenticated execution, delegation continuity, and mission coordination while allowing complementary policy and governance architectures to determine whether delegated authority remains valid as mission context, organizational policy, and authority state evolve.

## Certificate Capabilities Evaluation (zCaps and UCANs)

### Overview

Two of the specifications in this survey are implementations of the same underlying model, and this chapter evaluates them together. Authorization Capabilities ("zCaps", standardized in draft as [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/)) and the [User-Controlled Authorization Network](https://ucan.xyz/) ("UCANs") are both certificate capability systems: the authorization is a structured, cryptographically signed artifact that a delegator hands to a delegatee, carrying within itself everything a verifier needs to check it. Both descend from the [object-capability](https://en.wikipedia.org/wiki/Object-capability_model) tradition (UCAN cites [SPKI](https://en.wikipedia.org/wiki/Simple_public-key_infrastructure) as its direct ancestor), and both make the same core design choice: access is decided by possession ("do you hold a valid capability, and can you prove control of its key?") rather than by identity ("who are you, and what does my policy store say about you?").

The shared model works as follows. The root of authority is the resource owner; there is no central authorization server. A capability answers which agent can do what, with which resource, under which restrictions. Delegation is performed offline, by signing a new capability with a key the parent authorized; no round trip to the resource server or any other party is needed to delegate. Each delegation step must monotonically attenuate: a delegatee can pass on no more than they hold, along whatever axes the spec defines (actions, time, resource scope, policy). At invocation time, the delegatee presents the capability together with its full delegation chain and a proof of key possession, and the verifier checks the chain locally and cryptographically.

Because the two specs score nearly identically against the evaluation checklist, the scorecard below covers both in one table, with the shared reasoning stated once and per-spec nuances called out. The second half of the chapter covers where the specs genuinely diverge: encoding, resource scoping, revocation mechanics, chain representation, and a few features present in only one of the two.

Reference points used in this evaluation:

* the [ZCAP v0.3 specification](https://w3c-ccg.github.io/zcap-spec/) (the data model), and the [zCap Developer's Guide](https://github.com/interop-alliance/zcap-developer-guide), which documents the deployed state of the art; where deployment has diverged from the spec text (most notably: typed caveat objects are largely unused in production, replaced by URL-suffix attenuation, allowedAction, and expires), this is called out explicitly;  
* the [UCAN specification set](https://ucan.xyz/specification/): the overview plus the Delegation, Invocation, and Revocation sub-specs. (The Promise sub-spec is skipped as still marked draft.)

### Scorecard

| \# | Requirement | zCaps | UCAN | Notes |
| ----- | ----- | ----- | ----- | ----- |
| 1 | Accountable (agent vs. principal/operator) | Partial | Partial | Same verdict for the same reason: every delegation edge is signed and thus attributable to a responsible delegator, but neither data model distinguishes "agent" from "legal-entity operator". B; both are opaque DID-identified principals. Identity binding is delegated to a companion layer (VCs and issuer registries for zCaps; DID methods and higher-level identity systems for UCAN). |
| 2 | Resistant to confused deputy | Yes | Yes | By construction, as capability systems: designation and authorization travel together in the signed artifact (invocationTarget plus allowedAction; subject times command times policy), so a delegatee never exercises its own ambient authority on a resource designated by another party. |
| 3 | Represent authorization policies ("undelegatable", re-delegation rules, advisory "please") | Partial | Partial | Both provide an extensible policy slot and neither ships a standard vocabulary. zCaps reserve a generic caveat property, unused in deployment. UCAN defines a policy syntax (the pol property) and reserves the /ucan command namespace, but standardizes no caveats for delegation control. Neither has an advisory (non-enforced) tier; both are binary enforce/reject. |
| 4 | Chainable | Yes | Yes | The headline feature of the family. zCaps: parentCapability plus capabilityChain proof chains. UCAN: hash-linked self-certifying certificates rooted in the resource owner. Both cover operator to agent and agent to sub-agent, with links creatable offline. |
| 5 | Cross-organizational / locally verifiable | Yes | Yes | Verification is cryptographic and local; principals are DIDs, so no party needs an account on another's IAM system. zCaps mandate the full chain be carried in the invocation, with no verifier network access required. UCAN resolves proofs by CID from a local store, DHT, gossip, or REST. DID resolution and revocation freshness may still touch the network in both. |
| 6 | Attenuated | Yes | Yes | Both enforce monotonic attenuation, on different axes. zCaps: allowedAction subsets, no-later expires, URL path/query suffix narrowing of invocationTarget. UCAN: capability sets must not widen, nbf/exp validity intersection, command-lattice narrowing, policy statements. |
| 7 | Self-revocable (holder or anyone in chain) | Yes (at RS) | Yes (per Revocation spec) | Same guarantee, opposite mechanics. zCaps: any controller in the chain POSTs the zCap to an RS revocation endpoint; out-of-band, RS-enforced, online-only. UCAN: revocation is a first-class command (/ucan/revoke) with a Revoker role, expressed inside the same capability model; dissemination is left open (DHT, gossip, etc.). |
| \+ | Authentication / Proof of Possession | Yes | Yes | zCaps: HTTP Message Signatures (Cavage, migrating to RFC 9421\) or a Data Integrity capabilityInvocation proof; the request itself is signed. UCAN: the Varsig envelope is signed by the issuer key, with a required nonce, time bounds, and signed invocations. Both are DPoP-analogous: an intercepted invocation cannot be replayed without the key. |
| \+ | Privacy of delegation chain | No | No | Both treat the chain as a provenance log, visible to the carrying agent and submitted whole to the verifier. No selective disclosure, encryption, or blinding in either. (zCaps: urn:uuid ids reduce correlation; ephemeral one-off keys approximate blinding. UCAN: explicitly offers no confinement.) |
| \+ | Offline-capable | Yes for delegation and verification | Yes for delegation and verification | The family's signature efficiency win: delegating requires no contact with the resource server. Chain verification is local. Invocation depends on reaching the resource; revocation depends on reaching the RS (zCaps) or on propagation (UCAN). |

### Detailed Evaluation

#### Accountable: agent vs. principal/operator

Both specs satisfy the accountability half of this criterion structurally and the differentiation half only implicitly.

The actor is named directly: the controller field of a zCap, or the issuer/audience/subject DIDs of a UCAN. Accountability across the chain is a structural guarantee in both. Every zCap delegation carries a capabilityDelegation proof signed by a controller of its parent, and every UCAN must be signed by the issuer DID's key, with the chain of tokens forming a provenance log of who delegated what to whom. For every edge in the delegation graph there is a cryptographically attributable, non-repudiable responsible party, and walking the chain back to the root recovers the complete provenance of any authority.

What neither spec provides is a semantic distinction between "this DID is an autonomous agent" and "this DID is the legal entity that stands behind it." A principal is an opaque DID; the root controller is the operator only by convention. Both specs push the binding of key identifiers to real-world legal entities explicitly out of scope, and for similar reasons: it is not always desirable or possible, and forcing it would be a privacy violation in many settings. Where the mapping is needed, both point at the same companion pattern: use an identity layer (Verifiable Credentials, issuer registries, DID methods) for the human-judgement call at the entry point, and use the capability chain for the flow of authority thereafter. An auditor can always identify which key delegated which authority to which other key; note that this auditability can be fulfilled with local pairwise identifiers only, without global identities.

#### Resistant to confused deputy

Resistant by construction, and this shared property is the reason the family exists. In both specs, designation and authorization are inseparable parts of the signed artifact: a zCap binds invocationTarget to allowedAction, and a UCAN capability is defined as subject times command times policy. The verifier evaluates the presented chain, not the invoker's own ambient standing, so there are no deputy permissions to confuse. A delegatee asked to act on a resource designated by someone else either holds a capability whose designation covers that resource, or the invocation fails.

#### Ability to represent authorization policies

The least emphasized area for both specs as currently deployed, and the clearest shared gap.

Both have the extension point. The zCap spec defines a generic caveat property: typed restriction objects (the spec's examples are ValidWhileTrue and DriveNoMoreThan) inherited down the chain, whose meaning "must be understood between the entity adding a caveat and the target evaluating it." UCAN goes further on machinery: policy is a dedicated top-level property (pol) with a [defined syntax](https://ucan.xyz/delegation/#policy) of statements and operators, alongside arguments (args) that constrain the command, and a reserved /ucan command namespace for community-blessed commands.

Neither ships vocabulary. In zCap deployments no caveat notation is currently used at all; attenuation is expressed implicitly through allowedAction, expires, and URL-suffix narrowing. UCAN's policy syntax expresses conditions on invocation arguments (the spec's examples are of the form "from this email address"), but standardizes nothing for delegation control.

Against the checklist's specific policy examples, the two specs fail identically:

* "Do not delegate further" / "this permission is undelegatable": not natively expressible in either. zCaps have no depth-limit or non-delegation flag; the only depth control is a verifier-global chain-length cap (the RS's blanket rule, not a per-capability instruction). UCAN likewise defines no such caveat; delegation is constrained only by the attenuation rules. (In the zCap design philosophy this is deliberate: forbidding delegation is considered an anti-pattern, since a delegatee can always proxy or share the controlling key.)  
* Re-delegation policies: not standardized in either; both would need custom caveats or resource-specific policy semantics.  
* Advisory "please" rules (desired re-delegation rules whose violation is not refused): no notion of advisory, non-enforced policy in either spec. Caveats and policies are evaluated at invocation and a failure invalidates the invocation. The model is binary enforce/reject, with no honor-if-possible tier.  
* "Who gets to know what is a governance topic": consistent with the object-capability philosophy, both specs deliberately encode authority, not knowledge or identity policy; disclosure governance is out of scope.


So both data models can carry policy through an extensible slot, but neither ships a shared vocabulary for the policies the checklist cares about, and current deployments do not use the slots. This is arguably the highest-value area of future work for the family, if it is to serve agentic delegation.

#### Chainable

The headline strength of both specs and the reason the model exists.

In zCaps, delegation is expressed by parentCapability (pointing up at the parent or the root) plus a capabilityDelegation proof whose capabilityChain enumerates the ancestry. The root of authority is the resource itself: a root zCap controlled by the operator, from which authority flows root to agent to sub-agent. In UCAN, each token is a self-certifying certificate attesting a subset of its delegator's authority, hash-linked to its proofs and rooted in a resource owner; the Subject (sub) field marks whose authority the chain is about, and the chain is by definition a provenance log.

Both directly satisfy the two senses the checklist asks for: principal/operator to agent (root to first delegation), and agent to further subsystems or other agents (the zCap spec's valet example, car to Alyssa to Ben to Lem, is precisely a multi-hop agent-to-agent chain). In both, each link is created offline, simply by signing a new child artifact with a key the parent authorized; no involvement of the resource server is required to delegate.

#### Cross-organizational / locally verifiable

Strong in both, and again by design. Verification is purely cryptographic and local to the verifier: it traverses the chain from the root down, checking each delegation proof against the key authorized by the step above, and checking that each step only narrows authority. Principals are DID-based key identifiers, so two organizations can delegate to each other purely by exchanging signed capabilities, with no accounts on each other's IAM systems.

The specs differ in how the chain reaches the verifier. zCaps fix this normatively: a delegated zCap must carry its entire chain in the invocation, and a verifier must not be required to make network requests or database queries to dereference it; the root of trust is the RS, which dereferences the root zCap's controller from its own store. UCAN relies on content addressing: tokens are identified by CID, and proofs are resolvable from wherever the deployment keeps them (local store, DHT, gossip, REST), which makes chain transport a policy choice rather than a normative embedding rule.

Verification involves several steps, some of which (like DID resolution) may need to access the network, and some (like checking a local revocation registry) can be done offline. To put it another way, chain structure verification can be performed offline; key revocation checks and other verification steps may need network access.

#### Attenuated

Both specs enforce monotonic attenuation, checked by the verifier at every link: a delegator can hand out no more than they hold. The axes differ.

zCaps attenuate along three deployed axes: allowedAction in a child must not be less restrictive than the parent's; expires must not be later than the parent's (and every delegated zCap must have an expiry); and invocationTarget may be narrowed by appending a URL path or query suffix (/bars/123 to /bars/123/bazzes/456?day=tuesday). The caveat property is reserved as a fourth axis but unused in practice.

UCAN attenuates abstractly: each direct delegation must restate or diminish its capabilities, the validity interval is the intersection across the chain (latest nbf to earliest exp), and commands narrow along a lattice, where shorter command paths prove longer ones in the same namespace (/crud can prove /crud/read but not /stack/pop). Policy statements and arguments provide further attenuation axes.

The limitations are mirror images. zCap attenuation of the resource is restricted to URL-suffix shaping, fine for hierarchical REST resources but awkward for non-hierarchical scoping, and a single zCap carries exactly one invocationTarget (multiple target/action combinations are listed as future work). UCAN's abstraction is more flexible but less prescriptive: resource scoping is left to how cmd and args are interpreted, with no normative URL rules in the core spec.

#### Self-revocable

Both specs meet the checklist requirement that the holder, or someone in the delegation chain, be able to revoke (modulo network partitions), and this is the criterion where their mechanics diverge most sharply.

zCaps treat revocation as a resource-server state change. Any controller appearing in a delegated zCap's chain may revoke it (revoking everything downstream) by POSTing the zCap to the RS's revocation endpoint; the spec sketches a /zcaps/revocations subpath whose root zCap is controlled by all controllers in the chain, precisely so that any ancestor can revoke. Mandatory expires on every delegated zCap bounds how long a revocation must be remembered and bounds damage from a lost-but-unrevoked capability. The mechanism is out-of-band and RS-enforced: online-only, with the RS as the single enforcement point.

UCAN treats revocation as a first-class operation inside the same capability model. The Revocation sub-spec defines a Revoker role (an issuer listed in a proof chain who revokes a UCAN) and revocation is expressed as a command in the reserved /ucan namespace; executors must validate revocation state at execution time. How revocation notices propagate is left to the deployment (DHT, gossip, and similar patterns are all permitted), so there is no mandated always-online central point, at the cost of unspecified dissemination guarantees.

The shared characteristic worth stating plainly for agentic use cases: in neither spec is revocation embedded cryptographically in the token itself. It is state that the enforcement point (the RS, or the executor's revocation store) must learn about, so offline revocation semantics are unavailable in both.

### Nice-to-haves

#### Authentication / Proof of Possession

Possession alone is never sufficient in either spec; invocation requires proving control of the authorized key.

zCaps define two binding mechanisms: an HTTP Message Signature (deployments use Cavage Draft 12 today, migrating to RFC 9421\) over a Capability-Invocation header, plus Digest/Content-Digest for bodies; or a Data Integrity capabilityInvocation proof attached to a request document. Either binding mechanism requires a digital signature. Note that as with all proof-of-possession signature mechanisms, additional replay attack mechanisms are required (and used by zCaps), such as short expiration periods and unique ids tracked by the resource server/verifier. 

UCAN builds proof of possession into its envelope: every token is signed by the issuer DID's key over the canonical DAG-CBOR payload, and replay protection is layered (a required nonce, nbf/exp time bounds, and signed invocations at the transport layer). The intent is the same as zCaps' HTTP signatures; the expression lives in the UCAN envelope rather than in a separate signing convention.

#### Privacy of the delegation chain

No, in both. The full chain must reach the verifier on invocation and is carried by (and therefore visible to) the invoking agent itself; the principal, the verifier, and the token carrier all see the whole chain. Neither spec defines selective disclosure, encryption, or blinding of intermediate delegatees. UCAN is explicit that chains are provenance logs and that UCANs do not offer confinement; delegatees can sub-delegate without alerting their delegator. zCaps use urn:uuid identifiers to reduce correlation, and most of the desired effect can be approximated with ephemeral one-off keys, but neither is a chain-privacy mechanism. In both systems, additional information associated with the actors in the chain, if any, is stored usually out-of-bound as non-key DID document contents, verifiable credentials, or other identifier-anchored metadata.

#### Offline-capable

The offline story is the same for both, split by lifecycle stage, and it is the family's signature efficiency advantage:

* Delegation: fully offline. Delegating is signing a new child artifact with an authorized key; no contact with the resource server or any authorization server.  
* Chain verification: offline-capable. The verifier checks the supplied or resolved chain locally (subject to the DID-method and revocation-freshness caveats noted under \#5).  
* Invocation: depends on reaching the resource. If the RS or executor is reachable offline (local machine, LAN), invocation can be offline too.  
* Revocation: depends on reaching the RS (zCaps) or on the deployment's propagation pattern (UCAN), per \#7.

### Where the Two Specs Diverge

The evaluation above shows the two specs scoring as a family. This section highlights features that exist in one spec but not the other. (The revocation divergence, endpoint versus command, is covered under criterion 7 above.)

#### Multiple actions per capability

A delegated zCap may carry an allowedAction that is a single string or an array of strings (for example "allowedAction": \["write", "read"\]), so one capability can directly encode "read and write but not delete", with verifiers enforcing that a child's array is no less restrictive than its parent's. A UCAN payload has exactly one cmd string; the spec defines no list form. To grant multiple methods, UCAN relies on either a more general command (such as /crud) whose policy and arguments restrict which operations are allowed, or multiple UCANs, each with its own cmd.

Asymmetry: zCaps have built-in multi-action support; UCAN pushes multi-method expressiveness into policy or multiple tokens.

#### Resource scoping: URL attenuation vs. command lattice

zCaps make URL structure part of the attenuation rule: a child's invocationTarget must equal the parent's or extend it with a path or query suffix, with normative rules for the suffix forms. UCAN has no invocationTarget equivalent; resources are governed by the Owner/Subject relationship and command semantics, and narrowing happens along the command lattice (segment structure matters: shorter commands prove longer paths in the same namespace) plus arguments, with no normative URL rules.

Asymmetry: zCaps hard-code hierarchical URL narrowing in the data model; UCAN scopes resources through its command and argument structure.

#### Capability composition and authority union

UCAN defines a spec-level algebra for combining authority: the set of capabilities delegated by a UCAN is its "authority", merging capability authorities must follow set semantics (the result includes all capabilities from the inputs), and authorities can be combined in any order. This supports compositionality across resources, with no distinction between co-located and distributed resources. zCaps define only single-root, single-target chains: one parentCapability per delegated zCap, one invocationTarget per capability, and no defined algebra for merging multiple chains into a combined authority.

Asymmetry: UCAN specifies authority union; zCaps do not. (As discussed in the evaluation checklist, composability is trivially satisfied by any capability system at the model level; the asymmetry here is about spec-level machinery, not achievability.)

#### Container and signature format

zCaps are JSON-LD documents (with a zcap context plus a security suite context) signed with Data Integrity proofs; there is no content-addressing requirement. UCANs are DAG-CBOR payloads inside a Varsig envelope, identified by CIDv1: native IPLD objects, with the signature scheme described by the Varsig header. This is the divergence that most shapes each spec's natural habitat: zCaps sit on the web-standards stack (JSON-LD, Data Integrity, HTTP signatures), UCAN on the content-addressed, local-first stack (IPLD, CIDs, replicated stores).

#### Chain structure and storage

zCaps fix the wire representation of the chain: the capabilityChain must include the root zCap's id and every ancestor by id, except the immediate parent, which must be fully embedded; verifiers must not be required to perform network requests to dereference the chain. UCAN defines chains logically and relies on content-addressed storage and transport: proofs may be carried inline or referenced by CID and resolved from a delegation store, DHT, gossip, or REST endpoint, with no prescribed embedding pattern.

Asymmetry: zCaps trade transport size for guaranteed verifier self-sufficiency; UCAN trades a resolution step for flexible, deduplicated proof storage.

#### Invocation mechanism

zCaps define invocation conventions for two carriers: an HTTP capability-invocation header (with the zCap attached compressed in a header parameter) signed with an HTTP signature, or a Data Integrity capabilityInvocation proof on an arbitrary JSON-LD document. UCAN reuses its own envelope: the Invocation sub-spec describes an Invoker sending UCANs plus args to an Executor, which validates signatures, chains, nonce, time bounds, and revocation; transport is left open, and Data Integrity is not used.

#### Powerline (UCAN only)

UCAN's delegation work defines Powerline, a pattern for automatically delegating all future delegations to another agent regardless of Subject. Once established, it acts as a standing delegation conduit: whenever any issuer delegates along that line, authority transparently flows to the designated sink agent without being explicitly named in each delegation. It is built from normal UCAN pieces (principals, subjects, delegation rules) rather than a new primitive, and it is useful for centralizing audit, enforcement, or mediation in a specific agent, for example a storage or orchestration service that should automatically receive all capabilities granted in a namespace.

Asymmetry: zCaps have no analog to the Powerline pattern.

### Summary

zCaps and UCAN are two realizations of the same certificate-capability model, and against the evaluation checklist they score as a family: strong on the mechanics of delegated authority (chaining, monotonic attenuation, locally and cross-organizationally verifiable proof chains, proof of possession, offline delegation), and weak in the same two places. Neither distinguishes agent from operator in its data model (both delegate identity binding to a companion layer such as VCs), and neither ships a standard policy vocabulary for delegation control ("undelegatable", re-delegation rules, advisory semantics), despite both reserving the extension point for one. Chain privacy is absent from both, and revocation in both is state the enforcement point must learn, not a property of the token.

Those shared gaps are family-level gaps, which makes them natural working-group targets: a standardized caveat/policy vocabulary and a companion identity-binding mechanism would lift both specs at once.

The differences between the two are real but sit a level down, in encoding and ecosystem rather than capability model: JSON-LD and Data Integrity on the web-standards stack versus DAG-CBOR and CIDs on the content-addressed local-first stack; normative chain embedding versus resolvable proof stores; multi-action arrays versus a command lattice; an RS revocation endpoint versus a revocation command. For delegated authorization for AI agents, choosing between them is largely a deployment-ecosystem decision; the properties that matter for the agentic checklist, and the work still needed, are common to both.

## Cedar/Janssen Evaluation

### Overview

[Cedar](https://www.cedarpolicy.com/en) is an open-source authorization policy [language](https://docs.jans.io/stable/cedar-intro/?h=cedar) and evaluation engine created by IAM specialists at Amazon with formal methods training (used in AWS Verified Permissions; the language syntax itself is now a CNCF [candidate project](https://www.cncf.io/projects/cedar/)). A Cedar policy answers "**may** this principal **perform** this action **on** this resource **under** these conditions (context)" using `permit` and `forbid` statements validated against a typed schema. Evaluation is deterministic, default-deny, and `forbid` always overrides `permit`.  Over time, the core Cedar [language](https://docs.jans.io/stable/cedar-intro/?h=cedar) and its canonical reference implementation in Rust have been generalized from Amazon’s production implementation in Keycloak/AWS contexts to be hardened and governed at CNCF within the Linux Foundation (as mentioned above); a cloud-agnostic IAM framework generalizing the Keycloak/AWS tooling is governed in a dedicated Linux Foundation Project called [Janssen](https://docs.jans.io/stable/getting-started/), which also governs and develops the younger “Cedarling,” a lightweight version of Cedar’s enforcement engine that can be run “practically anywhere” as a WASM component inside of not just servers but even clients and agents/harnesses.


**Cedar has no native, enforced delegation primitive.** It is a policy expression language plus a Policy Decision Point (PDP), with no native concept of a credential, a holder, a signature, or a delegation chain. It is *identity-based* in the classic sense: the verifier decides access by evaluating who the principal is (and their attributes and group memberships, as made known to the verifier by tokens from server-side tooling like Janssen) against policies the verifier holds, in contrast to possession-based models like zCaps, where presenting a valid signed capability constitutes the “just-in-time” proof of authority. This identity-vs-possession distinction drives nearly every row of the scorecard.

Cedar is therefore evaluated here **as a component in a delegation stack**: Cedar supplies policy expression and local evaluation in cross-organization use cases where policy must be made intelligible to callers and counterparties across policy boundaries; the delegation mechanics (chain construction, portable proof, holder-side revocation) must be supplied by a carrier layer around it, but the policy anchoring the semantics of those delegations and error messages can be more meaningfully communicated by the combination of the layers.

Three reference points are used:

* the [Cedar language and documentation](https://docs.cedarpolicy.com/) (the "spec": syntax, schema, validation, templates, evaluation semantics),  
* the [Janssen Project Cedarling](https://docs.jans.io/stable/cedarling/), the most relevant deployed embedding: a lightweight, embeddable PDP (Rust core with WASM, mobile, Python and Java bindings) that implements Token-Based Access Control (TBAC), deriving Cedar principal entities from JWTs issued by trusted issuers and evaluating policies locally at the edge, and  
* [Clawdrey Hepburn](https://clawdrey.com/), a first-person agentic case study: an autonomous AI agent whose operator-defined action boundaries (which accounts it may follow, what it may do with its own credentials and payment card) are enforced by an embedded Cedar engine checked before every action, with a typed schema modeling the agent, its operator, and its action space.


Where Cedar (the language) and runtime-evaluation deployments like Cedarling differ in what they provide, this is called out, since several matrix rows are satisfiable only by the deployment layer.

### Scorecard

| \# | Requirement | Verdict | Notes |
| ----- | ----- | ----- | ----- |
| 1 | **Accountable** (agent vs. principal/operator) | Yes (at the modeling level) | The schema can type the distinction directly (e.g. `entity Agent in [User]`); Cedarling derives separate User and Workload principals from the `id_token` and `access_token` and can require both to be authorized. But the distinction is data the verifier trusts an Authorization Server and token exchange to provide, not a cryptographically attributable delegation edge. |
| 2 | **Resistant to confused deputy** | No | Identity-based by design: the deputy exercises its own standing permissions against a caller-designated resource, and the model does not combine designation with authorization. Policies can condition on request context to mitigate specific deputy scenarios, but that is mitigation, not structural immunity. |
| 3 | **Represent authorization policies** (embed/link; "undelegatable", re-delegation rules, advisory "please") | Yes, except advisory | This is Cedar's core competence. "Do not delegate further" and re-delegation rules are expressible if delegation itself is modeled as an action. No advisory tier: decisions are binary permit/forbid, with no obligations or native "honor-if-possible" semantics. |
| 4 | **Chainable** | No (stack-dependent) | No delegation primitive, no chain artifact. Delegation can be modeled as verifier-side **state** (entities or instantiated policy templates), but chains are not portable, signed, or first-class. In a delegation stack, the chain must live in the carrier or elsewhere in the Authorization stack; Cedar evaluates each hop atomically. |
| 5 | **Cross-organizational / locally verifiable** | Partial | Evaluation is fully local (Cedarling runs in browsers, mobile, gateways; multi-issuer JWTs supported without shared IAM accounts). But the policy store is unilateral, that is \-- the verifier's. There is no standard or guidance for exchanging ~~or federating~~ policy semantics across organizations in an adhoc way, and trusted issuers must be pre-configured~~.~~, BUT core contributors do work to keep Janssen [in lockstep with AuthZEN](https://github.com/JanssenProject/jans/pull/13077) for federation and runtime issuer-list capabilities |
| 6 | **Attenuated** | Partial | Forbid-overrides-permit gives monotonic restriction: layering additional forbid policies (or a stricter verifier-side policy set) can only shrink authority, never expand it. But Cedar has no runtime check that a delegatee's grant is a subset of the delegator's; subset/equivalence can be proven offline with Cedar's symbolic analysis tooling. Attenuation enforcement is a stack responsibility. |
| 7 | **Self-revocable** (holder or somebody in chain) | Partial | Revocation is a policy store change (remove a permit or add a forbid), immediate at the next evaluation and very fine-grained. But it is performed by whoever administers the policy store, not natively by a delegator or delegatee in the chain; mapping chain participants to revocation rights is a stack design task. Cascade to downstream grants is not native either: it holds only if delegation is modeled as linked entities whose policy re-checks the ancestry. |
| \+ | Authentication / proof of possession | No (out of scope) | Cedar does not authenticate. Cedarling validates JWT signatures (proof of issuance by a trusted issuer), which is not holder proof of possession; PoP binding is left to the token layer. |
| \+ | Privacy of delegation chain | Possible (by architecture) | There is no chain in the data model, so nothing is carried in the request: a delegatee presents only their own token/identity. Intermediate delegatees need not learn the rest of the chain. The flip side: the verifier's policy store and entity data must hold whatever graph exists, so the verifier sees everything. |
| \+ | Offline-capable | Partial | Evaluation is fully offline once the policy store is loaded (this is Cedarling's headline feature). But *granting* a delegation means writing to a policy store, which is an online, administrative act, the opposite of zCaps' offline delegation-by-signing. |

### Cedar Evaluation

#### Accountable, agent vs. principal/operator

Differentiation is a schema design choice, made trivially. Cedar entities are typed and hierarchical, so the agent/operator relationship can be modeled directly, for example `entity Agent in [User] { model: String, runtime: String, trust_level: String }`, which is exactly the pattern the Clawdrey case study uses: the agent is a typed principal nested under its human operator, and policies can condition on either or both. Cedarling operationalizes the same split out of the box: it constructs a **User** principal from the OIDC id\_token and a separate **Workload** principal from the OAuth access token (the software acting on behalf of the person, or autonomously), and its default decision logic can require that *both* the person and the workload be authorized for the request to proceed. That is precisely the agent-vs-operator distinction the matrix asks for, expressed natively in the data model rather than by convention.

What Cedar does not provide is cryptographic accountability for the *delegation* itself. There is no signed delegation edge: the "fact" that operator O stands behind agent A is an entity attribute or a token claim that the verifier chose to trust. Attribution quality is therefore exactly the attribution quality of the surrounding identity layer (the JWT issuer, in Cedarling's case). An auditor can read the decision logs (Cedarling logs every decision with diagnostics) and reconstruct *which policy permitted which principal*, but cannot derive a non-repudiable "this delegator authorized this delegatee" proof from the Cedar artifacts alone. The accountable party for any grant is, structurally, the policy store administrator.

#### Resistant to confused deputy

Evaluated as-is: no, and the exposure follows directly from the identity-based model. The verifier decides access by evaluating who the principal is against verifier-held policies, so a deputy service acting on a caller-designated resource exercises its own standing permissions; nothing in the model binds the caller's designation to the authority being used. Policies conditioned on request context can mitigate specific deputy scenarios, but that is per-policy mitigation, not the structural designation-plus-authorization coupling that capability systems provide. In a delegation stack, immunity would have to come from the carrier layer, with Cedar evaluating the rules at each hop.

#### Ability to represent authorization policies

Policies are first-class, schema-validated, and arbitrarily conditional on principal, action, resource and request context: attribute comparisons, group membership (`in`), set operations, string and IP functions, and explicit `forbid` statements that override any permit. Policies can be embedded with the verifier or distributed to it (Cedarling fetches signed policy stores over HTTPS or from a Lock Server), satisfying the embed-or-link condition.

Against the matrix's specific examples:

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

**Cross-organizational: partial.** Cedarling's TBAC model is genuinely multi-party at the *evidence* level: `authorize_multi_issuer` accepts JWTs from several different trusted issuers in one request and maps their claims to Cedar entities, so organization A's verifier can consume identity evidence issued by organization B without B having an account in A's IAM. But the *policy* level is unilateral: the policy store (schema, policies, trusted issuer list) belongs to the verifier alone. There is no standard for one organization to hand another a policy fragment with agreed semantics, no shared schema federation, and the trusted issuer list is pre-configuration, not dynamic trust establishment. Two organizations can interoperate, but only after bilateral setup; the matrix's "express authorization policies without accounts on each other's IAM systems" is met for authentication evidence and unmet for policy exchange.

#### Attenuated

Cedar has one structural property that maps beautifully onto attenuation, and one missing property that prevents a full Yes.

The structural property is **monotonic restriction by composition**: `forbid` overrides `permit`, and evaluation is over the union of all policies. Adding policies to a set can therefore only ever *shrink* what a forbid touches and a verifier (or any later layer) can stack stricter policy on top of a base grant with a guarantee that the stack cannot widen authority. This is a natural fit for the delegation-stack pattern in which each hop attaches further restrictions: express the delegator's grant as the base policy set and each subsequent restriction as additional forbids or narrower permits evaluated together.

What is missing is **enforced subset semantics between grants**. If a delegation is represented as "delegatee gets policy set B, derived from delegator's policy set A", nothing in the Cedar runtime checks B ⊆ A; an over-broad B is just another valid policy set. The check is possible, and this is where Cedar is unique: its analyzability means "B permits nothing A does not" is a decidable question answerable by symbolic analysis tooling at delegation time or audit time. But it is an offline/toolchain guarantee the stack must invoke, not a property the evaluator enforces per request the way a zCaps verifier enforces allowedAction narrowing.

Expressiveness of attenuation, once enforced, is far richer than URL-suffix narrowing: any condition the language can state (attribute ranges, context conditions, time windows, resource groups) can be the attenuation axis.

#### Self-revocable

Revocation in Cedar terms is straightforward and immediate: delete the permit (or the template instantiation, or the delegation entity) or add a forbid, and the very next evaluation reflects it. Forbid-overrides-permit makes "revoke" particularly clean: a targeted forbid kills a grant without having to find and remove every permit that contributed to it. Granularity is excellent: one delegatee, one action, one resource, one condition.

The matrix question, however, is *who* can revoke. Natively: whoever can write to the policy store. That is an administrative role, not a chain position. A delegator can revoke what they delegated only if the stack maps "delegator of grant X" to "may modify/forbid grant X in the store", which Cedar can itself express as policy (delegation management actions authorized by Cedar policies, the recursive pattern) but does not provide out of the box. Propagation is also a deployment property: Cedarling instances cache their policy store and pick up changes on refresh or via Lock Server push, so revocation latency equals policy distribution latency. Within a single trust domain this is near-immediate; across many edge PDPs it is eventually consistent.

On **cascade** (does revoking a grant revoke everything delegated below it): Cedar has no built-in notion of a chain, so there is no built-in cascade either. Whether revocation propagates downstream depends on how delegation is modeled. If each grant is an independent policy, revoking Alice-to-Bob leaves Bob-to-Carol untouched, and Carol keeps her access until that grant is separately removed. Cascade is achievable by modeling delegation as data with parent links plus a permit policy that succeeds only when the whole ancestry is still valid: each delegation is an entity carrying a parent reference, and Carol's evaluation walks up the chain, so revoking Alice-to-Bob cascades to Bob-to-Carol because the walk hits the now-missing or now-forbidden link and fails closed. This is a contrast with zCaps, where cascade is structural (a delegated capability embeds its parent, so rejecting any link fails everything below it by construction); in Cedar cascade is a property you build into the policy and entity model, not one the engine provides.

Net: revocation is a strength of the centralized-policy model (no token to chase, no expiry to wait out, unlike credential-based systems where an issued artifact is "out there"), but holder/chain self-revocation is a stack construct, not a Cedar feature.

#### Composability (spec-level note)

A clear yes. Cedar evaluates every request against the union of all loaded policies; an Allow needs at least one satisfied permit and no satisfied forbid, and the satisfied permit(s) can originate from any source loaded into the store. Principals can belong to arbitrarily many groups and roles simultaneously (the matrix's RBAC single-role counter-example does not apply: Cedar's `in` hierarchy is a DAG, not a single assignment). Cedarling extends composition to the evidence layer: `authorize_multi_issuer` lets one request carry tokens from multiple independent issuers, each mapped to entities, so authority deriving from several principals or organizations is combined in a single decision.

The one caveat mirrors row 5: all the composed sources must already be present in (or mapped into) the verifier's policy store and trusted issuer configuration by stable identifiers. Composition is native at decision time, but onboarding a new source of authority is configuration.

### Nice-to-haves

#### Authentication / proof of possession

Out of scope for Cedar by design: the engine assumes the principal has already been authenticated and evaluates the request it is handed. Cedarling adds issuance-side verification: JWT signature validation against pre-configured trusted issuers, aud/sub cross-checks between tokens, optional status checks, and exp/nbf checks. This proves the tokens were issued by a trusted party and are current, not that the presenter is the rightful holder. Sender-constrained tokens (DPoP/mTLS-bound) can be layered in by the token infrastructure, but Cedar/Cedarling does not itself verify possession.

#### Privacy of the delegation chain

An interesting inversion of the credential-based technologies. Because nothing is carried in the request, the matrix's desired property (the intermediate delegatee does not learn the rest of the chain) falls out of the architecture almost for free: each delegatee authenticates as themselves and presents their own tokens; whatever delegation graph exists is reconstructed verifier-side from the policy store and entity data. The principal (as policy author) and the verifier can know the full story while the token carrier knows only their own grant.

The cost is the mirror image: the verifier's policy store must contain the graph, so the verifier (and the policy store infrastructure) sees everything, and chain privacy *from the verifier* is impossible. Whether this counts as satisfying the row depends on which party the subgroup is trying to blind; for the stated formulation (hide the chain from the intermediate agent, principal and verifier know all) Cedar-style architectures actually do well.

#### Offline-capable

* **Evaluation/use: fully offline.** This is Cedarling's reason for existing: the PDP is embedded (browser WASM, mobile, gateway, local process) and decides locally with no network round trip. Local-machine and LAN-only agentic use cases are well served; the Clawdrey deployment is exactly this shape (an agent on a single machine consulting its local Cedar engine before each action).  
* **Delegation/granting: online and administrative.** Creating a new grant means writing to a policy store and distributing it. There is no offline delegate-by-signing; the efficiency advantage the matrix highlights (delegate without contacting anyone) is absent.  
* **Revocation: online to the store, then eventually consistent** to edge PDPs on refresh.

### Summary

Cedar is, almost row by row, the complement of the capability-based technologies in this survey. It is strongest exactly where zCaps are weakest: rich, analyzable, schema-validated policy expression (including "undelegatable" and re-delegation rules), native agent-vs-operator modeling, composition of authority from multiple sources in one decision, fine-grained immediate revocation, and fully local evaluation at the edge. It is weakest exactly where zCaps are strongest: there is no delegation chain, no portable signed grant, no holder-side revocation, no offline delegation, and no cryptographic attribution of who authorized whom; all of these are verifier-side state administered through a policy store.

For delegated authorization for AI agents, the implication is that Cedar is not a candidate *delegation data model* but a strong candidate *policy layer inside one*: a carrier technology supplies the chain, proof and portability, while Cedar supplies the rulebook each hop is checked against, with the unusual bonus that those rules can be formally analyzed (delegation attenuation provable offline). The Cedarling deployment shows the integration pattern already in production form (policies travel to the edge, identity evidence travels as multi-issuer JWTs, decisions stay local), and the Clawdrey case study shows the agentic pattern (operator-authored Cedar policies as the hard boundary on an autonomous agent's action space). What no current Cedar deployment shows is the missing piece the matrix cares most about: a standardized way for a delegator to hand a delegatee a portable, attenuated, chain-verifiable grant.

### References

#### Cedar language and engine

* [Cedar project site](https://www.cedarpolicy.com/en) — positioning, project overview, CNCF status.  
* [Cedar documentation](https://docs.cedarpolicy.com/) — primary language reference.  
  * [Basic Cedar syntax](https://docs.cedarpolicy.com/policies/syntax-policy.html) — permit/forbid effects, scope, when/unless conditions, and the forbid-overrides-permit / default-deny combining rules.  
  * [Terms & concepts](https://docs.cedarpolicy.com/overview/terminology.html) — entities, typed UIDs, the entity hierarchy, static vs. template-linked policies, RBAC/ABAC/ReBAC framing.  
  * [Policy templates](https://docs.cedarpolicy.com/policies/templates.html) and [Roles with policy templates](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles-templates.html) — `?principal` / `?resource` slots, template-linked policies as a runtime grant mechanism, and their coupling to user life-cycle management (the "delegation as template instantiation" argument).  
* [Cedar: A New Language for Expressive, Fast, Safe, and Analyzable Authorization (extended version, arXiv 2403.04651)](https://arxiv.org/pdf/2403.04651) — the design paper; basis for the non-Turing-complete / formally-verified / analyzable claims and the two-slot template restriction.  
* [cedar-policy/cedar (GitHub)](https://github.com/cedar-policy/cedar) and [cedar\_policy crate (docs.rs)](https://docs.rs/cedar-policy) — reference implementation; entity store model, local in-process evaluation, validator/typechecking.


#### Janssen Project Cedarling (the deployed embedding)

* [Cedarling overview / getting started](https://docs.jans.io/stable/cedarling/) — embeddable PDP on the Rust Cedar engine, WASM/mobile/server bindings, Token-Based Access Control, local edge evaluation.  
* [Authorization using Cedarling](https://docs.jans.io/v1.15.0/cedarling/reference/cedarling-authz/) — derivation of separate User (id\_token) and Workload (access\_token) principals, the "person AND workload must be authorized" decision logic, JWT validation steps.  
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

The term Core Specification refers to the [AuthZEN Authorization API 1.0](https://openid.net/specs/authorization-api-1_0.html) (hereafter referred to as "Core"), which defines the normative interoperability model between a Policy Enforcement Point (PEP) and a Policy Decision Point (PDP). As the primary standards-track specification and the foundation for runtime authorization interoperability, it serves as the baseline for evaluating AuthZEN against each criterion.

The broader AuthZEN ecosystem currently includes three complementary specifications:

It is important to note that these specifications are not all at the same level of maturity. AuthZEN Authorization API 1.0 is a Final Specification and serves as the normative foundation for this assessment. By contrast, COAZ and COAZ-MCP are currently Draft 1 specifications, while the AuthZEN Policy Store Format remains a working draft ([GitHub Repo](https://github.com/nynymike/AuthZen_Policy_Store/tree/main)). Accordingly, any assessment of the broader AuthZEN ecosystem should be understood as including current draft specifications in addition to the finalized Core specification.

* [AuthZEN Policy Store Format](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html) (hereafter referred to as "Policy Store"), which defines a portable and PDP-neutral packaging format for policy artifacts, schemas, trusted issuers, metadata, and related configuration.  
* [COAZ: A Framework for Mapping Information Models to AuthZEN Authorization Requests](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html) (hereafter referred to as "COAZ"), which defines a protocol-neutral framework for translating external protocol interactions into AuthZEN Subject-Action-Resource-Context (SARC) authorization requests.  
* [COAZ-MCP: COAZ Binding for the Model Context Protocol](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html) (hereafter referred to as "COAZ-MCP"), which applies the COAZ model specifically to the Model Context Protocol (MCP), enabling parameter-aware authorization decisions for AI agents, MCP gateways, and MCP servers.

It is important to note that these specifications are not all at the same level of maturity. AuthZEN Authorization API 1.0 is a Final Specification and serves as the normative foundation for this assessment. By contrast, COAZ and COAZ-MCP are currently Draft 1 specifications, while the AuthZEN Policy Store Format remains a working draft ([GitHub Repo](https://github.com/nynymike/AuthZen_Policy_Store/tree/main)). Accordingly, any assessment of the broader AuthZEN ecosystem should be understood as including current draft specifications in addition to the finalized Core specification.

The evaluation therefore follows a layered approach. Each criterion is first assessed against the capabilities provided by the Core specification alone. Where relevant, the analysis then identifies how Policy Store, COAZ, or COAZ-MCP specifications materially strengthen, clarify, extend, or operationalize those capabilities. This distinction prevents ecosystem features from being incorrectly attributed to the core protocol while still allowing AuthZEN to be evaluated as a practical authorization framework for agentic systems.

Its central data model is a runtime authorization request expressed in terms of Subject, Action, Resource, and Context (SARC). The core specification defines [single Access Evaluation](https://openid.net/specs/authorization-api-1_0.html#access-evaluation-api) and [multiple Access Evaluations](https://openid.net/specs/authorization-api-1_0.html#access-evaluations-api) endpoints together with [Search](https://openid.net/specs/authorization-api-1_0.html#name-search-apis) operations for discovering permissible resources, subjects, or actions. At the same time, it explicitly excludes policy language, architecture, policy state management aspects of a PDP and API authentication from its scope. Accordingly, this chapter distinguishes functionalities that are intrinsic to the AuthZEN protocol from functionalities supplied by the PDP, policy engine, identity system, or another surrounding layer.

The Policy Store addresses the policy-management side of the architecture by defining a PDP-neutral package, the bundle of policy files in a directory structure, for co-locating policies, schema, default entities, trusted token issuers, and supporting metadata. Also, this specification defines Policy Store API to manage the package. Its purpose is portability, versioning, auditability, and interoperability of policy artifacts across PDPs and tooling. Importantly, it standardizes the packaging and management format rather than the underlying semantics of a particular policy language or engine, thereby preserving implementation flexibility within PDPs while enabling interoperable policy distribution and lifecycle management.

COAZ addresses the boundary between external protocols and AuthZEN. It defines a protocol-neutral framework for declaratively mapping the inputs of an incoming operation into a SARC authorization request. Values may be fixed or computed from operation inputs, normally using Common Expression Language (CEL), allowing authorization decisions to reflect the actual parameters and context of an operation rather than only a caller identity. COAZ itself is protocol-agnostic, while protocol-specific bindings such as COAZ-MCP define how a particular protocol is projected into the SARC model.

Finally, COAZ-MCP applies that framework specifically to the Model Context Protocol. It maps MCP JSON-RPC operations into AuthZEN evaluations, providing default mappings for MCP methods and allowing tool-specific mappings where greater precision is required. This is particularly relevant to AI-agent authorization because it enables fine-grained, parameter-level policy evaluation at MCP gateways and servers. In the same way, future bindings could apply COAZ to other protocols, such as HTTP APIs, OpenAPI-described routes, gRPC services, or additional agent frameworks.

Many of the entries in the scorecard are marked as No, but this does not imply that AuthZEN is poorly suited to agentic authorization. Rather, they reflect the fact that AuthZEN addresses a different layer of the problem. Its primary role is to standardize runtime access decisions and PEP/PDP interoperability. Delegation chains, attenuation, revocation, and related capabilities remain the responsibility of capability systems or delegation model provided elsewhere in the architecture. As explained in the Overview chapter, an agent's required permissions cannot be fully known in advance, making least privilege dynamic. AuthZEN's access evaluation and search model is directly relevant to that problem because decisions can be made against the concrete action, resource, and context at execution time. Therefore, rather than representing a limitation, this can be considered a distinctive strength of AuthZEN, enabling it to be used without violating the requirements of agentic systems.

### Scorecard

| \# | Requirement | Verdict | Notes |
| :---- | :---- | :---- | :---- |
| 1 | Accountable (agent vs. principal/operator) | Partial | AuthZEN core can identify a subject but does not standardize relationship between the responsible-principal and the agent. COAZ-MCP explicitly separates human subject and agent context, which improves agent accountability. However, it does not establish a delegation proof or legal-operator binding. |
| 2 | Resistant to confused deputy | Partial (deployment-dependent) | AuthZEN evaluates explicit Subject-Action-Resource tuples and supports resource-specific authorization decisions. COAZ and COAZ-MCP further bind authorization to protocol inputs and trust-anchored identities. However, the ecosystem lacks delegation artifacts and end-to-end cryptographic delegation chains, leaving confused-deputy resistance dependent on correct PEP behavior and deployment architecture. |
| 3 | Represent authorization policies (embed/link; "undelegatable", re-delegation rules, advisory "please") | No for core (out of scope); Partial with ecosystem | Policy language, architecture, and state management of the PDP are outside the scope of AuthZEN. The ecosystem improves policy-artifact portability and request-binding interoperability through Policy Store, COAZ, and COAZ-MCP. However, authorization-policy semantics remain policy-engine-specific and are not inherently interoperable across policy engines or organization. |
| 4 | Chainable | No | Verdict is "No" because it is not a delegation framework. Not because it is incompatible with delegation frameworks. Neither the core data model nor COAZ-MCP contains a normative parent grant, delegator, delegatee, delegation edge, parent capability, or proof-chain primitive. COAZ-MCP improves subject/agent attribution, but not multi-hop delegation. [Access Evaluations](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluations-api) support request batching (boxcarring) but not chaining. Each evaluation is processed independently and cannot depend on the outcome of another evaluation in the same request. |
| 5 | Cross-organizational / locally verifiable | Partial (deployment-dependent) | Enable cross-vendor permission request/decision interoperability. Cross-organizational use is possible through a trusted remote PDP, but AuthZEN does not standardize policy exchange or provide locally verifiable proof of delegated authority. The receiving organization relies on the remote PDP's decision. |
| 6 | Attenuated | No | The core API has no native delegation or attenuation model. The broader ecosystem also does not establish that downstream authority is a non-widening subset of upstream authority. Any attenuation must therefore be implemented by the underlying policy engine and data model, not by AuthZEN itself. |
| 7 | Self-revocable (holder or anyone in chain) | No | No native self-revocation of delegated authority. Revocation is a responsibility of surrounding policy/token/delegation systems. Policy changes can revoke access, but this is different from delegation-chain self-revocation by the delegator or holder. |
| \+ | Authentication / Proof of Possession | No (out of scope) | AuthZEN relies on external authentication mechanisms and does not define holder proof-of-possession for delegated authority. |
| \+ | Privacy of delegation chain | No (undefined) | There is no standardized delegation chain to disclose or hide. An implementation could store one entirely PDP-side and thereby hide it from the agent, but that is an architectural consequence, not an AuthZEN feature. |
| \+ | Offline-capable | No | AuthZEN is transport-agnostic and a PDP can be deployed locally. However, "Offline" in this context means the ability to create, carry, present, and verify delegated authority without contacting a PDP. Because AuthZEN defines neither a native delegation artifact nor an offline-verifiable delegation primitive, its Offline-capable verdict should be No. |

### Detailed Evaluation

#### Accountable (agent vs. principal/operator)

The Core can identify a [Subject](https://openid.net/specs/authorization-api-1_0.html#name-subject), defined as a "user or machine principal," through mandatory type and id fields and optional properties. An [Access Evaluation API](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluation-api) contains one and only one Subject. While that Subject may represent either a user or a machine principal, the specification provides no standard way to model additional Subjects, distinguish an acting agent from an accountable principal/operator, or express a standardized relationship between them. [Context](https://openid.net/specs/authorization-api-1_0.html#name-context) could carry operator, delegator, or chain information, but their semantics would be implementation-specific rather than interoperable. Therefore, an auditor cannot reliably reconstruct the accountable principal or hop-by-hop delegation solely from a conformant AuthZEN request. The specification also excludes policy architecture, state management, and API authentication from its scope.

The result is more favorable when considering the broader ecosystem rather than the Core specification alone.

With COAZ-MCP, the binding explicitly separates the human subject from the acting agent by allowing the human identity to populate subject.id while representing the agent in [context.agent](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim). As a result, auditors can determine not only that an agent acted, but also on whose behalf the authorization decision was evaluated. However, the specification does not define delegation relationships or delegation chains.

COAZ strengthens the trustworthiness of this model through mapping controls and [trust-anchored](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields) fields. Bindings may require critical fields, such as subject identity, to be derived from independently verifiable inputs rather than caller-supplied data. This improves identity provenance and reduces the risk of agent-supplied identity assertions. Nevertheless, COAZ remains an identity-to-authorization mapping framework and does not define delegation relationships or delegation chains.

Thus, the ecosystem improves agent-versus-principal attribution and identity provenance, but it still falls short of providing interoperable delegation provenance and multi-hop accountability.

#### Resistant to confused deputy

AuthZEN's [data model](https://openid.net/specs/authorization-api-1_0.html#name-information-model) provides a structural foundation for confused-deputy resistance because every Access Evaluation request identifies a required Subject, Action, and Resource, with optional Context. [Resource](https://openid.net/specs/authorization-api-1_0.html#name-resource) represents the target, while [Action](https://openid.net/specs/authorization-api-1_0.html#name-action) represents the intended operation and may contain operation parameters. Consequently, AuthZEN can ask "may this subject perform this specific action on this specific resource?" rather than merely checking whether an identity has a generic role. However, the API does not define how the PEP derives these values from the actual invocation. Subject, Resource, and Action attributes are supplied by the PEP, API authentication is out of scope, and enforcement remains the PEP's responsibility. Therefore, AuthZEN alone neither proves that the Resource/Action matches what the upstream requester designated nor ensures that the authority used for a downstream operation is the authority intended by the requester rather than the deputy's own ambient authority.

COAZ and COAZ-MCP provide procedural binding between an invocation, the constructed authorization request, and the resulting authorization decision, reducing the likelihood that designation is lost and authorization decisions become based on the deputy's ambient authority. COAZ requires [bindings](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-information-model-and-input) to specify the source and structure of input variables, treats mapping inputs as potentially untrusted, provides a mechanism for [trust-anchored fields](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields), and requires mapping inconsistencies or evaluation failures to be treated as [mapping errors](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-denials-and-errors). COAZ-MCP binds MCP [tool names, arguments, resource URIs, token claims](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-information-model), and [server audience information](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-default-mappings) into authorization requests. Its trust-anchored [subject.id](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim) may be verified against the designated subject-identity claim, while [declared mappings](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-declared-mappings) allow authorization policies to incorporate tool parameters directly.

Taken together, these ecosystem specifications strengthen the preservation of requester intent and reduce the risk that access decisions become detached from the actual operation being performed. However, they still lack a cryptographically transferable designation-and-authority artifact and end-to-end attenuating delegation chain. Consequently, confused-deputy resistance remains partial and deployment-dependent.

### Represent authorization policies

Core specification does not standardize an authorization-policy representation. The Access Evaluation model provides SARC, but it does not define a standardized mechanism for embedding or referencing authorization policies themselves. Furthermore, the PDP's policy language, architecture, and state management are explicitly outside the scope of the specification. The specification illustrates advice, obligations, reasons, and step-up instructions as possible uses of [Decision Context](https://openid.net/specs/authorization-api-1_0.html#name-decision-context), but explicitly leaves their semantics and format outside the standard. Consequently, AuthZEN does not provide interoperable semantics for advisory rules, re-delegation constraints, or disclosure-governance policies.

Policy Store standardizes policy packaging and interchange, not policy semantics. Policies remain policy-engine-specific. Consequently, policy artifacts can be exchanged and discovered through a common packaging mechanism, but the semantics of delegation, re-delegation rules, undelegatable authority, and other authorization policies are not inherently interoperable across policy engines or organizations, since their meaning and enforcement are still defined by the underlying policy engine and its policy model.

COAZ and COAZ-MCP standardize how operations are mapped to AuthZEN authorization requests ([COAZ Mapping](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-the-mapping), [COAZ-MCP Declaring a Mapping](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-declaring-a-mapping)), not the semantics of authorization policies. COAZ provides declarative mappings from protocol inputs into AuthZEN SARC requests, while COAZ-MCP allows such mappings to be embedded in MCP tool definitions and enforced by a PEP before execution. Consequently, operations can be bound to authorization requests in a portable and interoperable manner, but the meaning of the policies being evaluated remains defined by the underlying PDP and policy engine rather than by COAZ or AuthZEN.

None of these specifications defines a common authorization-policy language or semantic model. They improve interoperability at the authorization interface layer (requests, responses, and policy artifacts), but not at the policy semantic layer. Consequently, different PDPs can exchange authorization requests and decisions through a common protocol, and policy artifacts may be shared through common packaging mechanisms, but the meaning and enforcement semantics of those policies remain implementation specific.

### Chainable

If being chainable is interpreted as preserving delegated authority across multiple hops, AuthZEN and its current ecosystem do not provide a standardized delegation-chain model. Delegated authority remains external to the AuthZEN interoperability model. The same observation largely applies to provenance reconstruction. However, AuthZEN itself should not be interpreted as an impersonation-oriented model. Although it does not solve multi-hop delegation, neither does it force workflows into impersonation. Instead, AuthZEN can consume and evaluate externally provided delegation-related context as part of a delegated-authority workflow.

The [Access Evaluations API](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluations-api) should likewise not be interpreted as delegation chaining. It supports batching (boxcarring) of authorization requests, but each evaluation remains an independent authorization decision. Multiple evaluations do not create, transfer, attenuate, or propagate authority between actors. Consequently, request batching is fundamentally different from a genuine delegation chain and should not receive chainability credit.

The ecosystem specifications improve interoperability but do not materially change this conclusion. Policy Store, COAZ, and COAZ-MCP improve portability, interoperability, and attribution, but none of them define delegated authority as a first-class, verifiable artifact across multiple hops. Nevertheless, like AuthZEN itself, they do not inherently force delegated-authority workflows into impersonation.

[AuthZEN Issue \#416 "\[Feature Request\] Add 'delegation\_chain' to Subject for AI Agent and Multi-Actor Scenarios"](https://github.com/openid/authzen/issues/416) further illustrates this point, which proposes introducing a delegation\_chain structure for AI-agent scenarios. The proposal explicitly argues that the current subject model cannot adequately represent multi-hop delegation chains involving users, agents, and downstream services. This should not be interpreted as existing support. Instead, it demonstrates that delegation provenance and multi-hop authority flow are recognized gaps in the current model.

Accordingly, if chainability is viewed as a property of delegated authority itself rather than authorization interoperability, AuthZEN should still receive No. Nevertheless, the reason is not that AuthZEN prevents delegation chains or requires impersonation. AuthZEN deliberately operates at a different architectural layer. It can evaluate and enforce authorization decisions informed by delegated-authority context, but the delegation chain itself, including attenuation, provenance, chain verification, and revocation semantics, must be supplied by another system. AuthZEN is therefore best characterized as a consumer of delegation chains rather than a provider of delegation chains.

### Cross-organizational / locally verifiable

The Core enables authorization interoperability across organizational boundaries and the ecosystem improves it, but participating organizations must separately establish trust in the relevant token issuers, [identity claims](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim), and PDPs. ([COAZ Trust-Anchored fields](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields) improve input trustworthiness but do not establish cross-organizational trust.) The Core does not define how such cross-organizational trust is established, and policy semantics remain external to the AuthZEN ecosystem.

The [Access Evaluation API](https://openid.net/specs/authorization-api-1_0.html#name-access-evaluation-api) returns an access decision, but not the underlying proof needed for another organization to independently validate why that decision was made. Nor do Policy Store, COAZ, or COAZ-MCP define such an authorization-proof format. AuthZEN ultimately relies on an access decision made by a PDP.

Consequently, an organization can operate its own PDP locally, and organizations can integrate their authorization systems across organizational boundaries, but neither outcome constitutes local verification of authorization evidence. The receiving organization still relies on the PDP that made the access decision and on the separately configured identity and token trust infrastructure, rather than independently verifying a transferable delegation or authorization artifact. The criterion is therefore rated Partial.

Cross-organizational deployments can be strengthened by combining AuthZEN with external trust frameworks for issuer trust, identity federation, key discovery, delegation credentials, and shared policy semantics, while continuing to use AuthZEN as the interoperable authorization layer connecting otherwise independent trust and policy domains.

### Attenuated

The Core alone can express narrowly scoped, context-sensitive access questions through SARC, making it well suited to runtime least-privilege enforcement. However, as with the previous sections, given the scope of the specification, neither monotonic attenuation nor the prevention of authority widening across delegation hops is defined by AuthZEN itself.

Adding the three ecosystem specifications does not improve support for attenuation. As discussed in the previous sections, the Policy Store does not establish parent-child delegation semantics or a monotonic subset invariant. COAZ strengthens the trustworthy construction of fine-grained authorization inputs, but it explicitly preserves AuthZEN request semantics rather than defining delegation semantics. COAZ-MCP extends this mapping framework to MCP, and its controls can enforce constraints supplied by policy. However, none of these specifications establishes that a child delegation cannot exceed the authority of its parent.

Thus, AuthZEN provides strong runtime enforcement for attenuated authority defined and validated elsewhere, but it does not provide native delegation attenuation.

### Self-revocable

The core specification provides no protocol mechanism by which a holder or an ancestor in a delegation chain can revoke a specific delegated authority, nor does it define any requirement for revocation to cascade to descendants. While changes to PDP policy may cause subsequent evaluations to return false, the specification does not define who is authorized to make such changes or whether that authority derives from participation in a delegation chain.

The specification also defines no revocation-state object, revocation-propagation protocol, cache-invalidation mechanism, network-partition behavior, or maximum revocation latency. Although expiration timestamps or other short-lived authorization evidence may be supplied as [contextual input](https://openid.net/specs/authorization-api-1_0.html#name-context) by an implementation, this represents expiration rather than explicit revocation and is not standardized by the API.

The ecosystem does not add delegation-chain self-revocation capabilities. The Policy Store specification allows administrators to upload [immutable, versioned Policy Store](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html#name-policy-store-api) packages through a [POST-only Policy Store API](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html#name-api-request), while existing policy stores cannot be updated or deleted. This supports policy deployment, versioning, and rollback, but not holder- or ancestor-initiated revocation. COAZ defines how protocol-specific information models are mapped into AuthZEN authorization requests, but it introduces neither delegation lineage nor revocation operations. Similarly, COAZ-MCP does not define holder- or ancestor-driven revocation, child-selective revocation, cascading revocation semantics, or guarantees regarding cache consistency, revocation propagation, or network partitions.

### Authentication / Proof of Possession

Authentication of the Authorization API is explicitly out of scope. While OAuth 2.0 support is [RECOMMENDED](https://openid.net/specs/authorization-api-1_0.html#name-model), OAuth 2.0 deployments are typically based on bearer tokens and therefore do not inherently provide holder proof-of-possession. Achieving proof-of-possession generally requires additional mechanisms such as DPoP, mTLS, sender-constrained tokens, or comparable cryptographic binding techniques. AuthZEN neither mandates nor standardizes any of these mechanisms. Consequently, while AuthZEN can operate in deployments that provide proof-of-possession through external identity and transport layers, the specification itself does not define holder-bound credentials or cryptographic proof-of-possession for delegated authority.

The ecosystem improves the integrity and trustworthiness of identity-related inputs, but it still does not satisfy the proof-of-possession criterion. COAZ can designate [trust-anchored fields](https://openid.github.io/authzen/authzen-coaz-framework-1_0.html#name-trust-anchored-fields) that must be derived from trusted inputs or verified by the PEP. COAZ-MCP uses [decoded JWT OAuth access-token](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-information-model) claims and [recommends anchoring subject.id to the designated subject-identity claim](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-declared-mappings). It also keeps agent identity separately in [context.agent](https://openid.github.io/authzen/authzen-coaz-mcp-binding-1_0.html#name-the-subject-identity-claim). These mechanisms reduce the risk of identity-claim substitution, but they do not require sender-constrained tokens, a signed invocation, or proof that the agent possesses a private key bound to delegated authority. The Policy Store packages [trusted-issuer](https://htmlpreview.github.io/?https://github.com/nynymike/AuthZen_Policy_Store/blob/main/draft-schwartz-authzen-policy-store.html#name-trusted-issuers) configuration and policy artifacts, but it likewise does not establish holder proof-of-possession for delegated authority.

### Privacy of delegation chain

AuthZEN API 1.0 does not satisfy this criterion because the specification does not define a delegation-chain representation and therefore provides no standardized mechanism for concealing, selectively disclosing, or verifying delegation chains. While an implementation could include delegation-related information in custom properties or context fields, the semantics and privacy characteristics of such information remain implementation-specific rather than standardized by AuthZEN.

Including the ecosystem specifications does not change this conclusion. Neither COAZ nor COAZ-MCP defines a multi-hop delegation chain or any mechanism for hiding selected hops from the MCP client or intermediary. Any delegation-chain privacy that may be achieved through a particular deployment architecture is implementation-specific rather than standardized by the AuthZEN ecosystem.

Accordingly, AuthZEN does not satisfy this criterion. More precisely, the property is undefined because no delegation-chain representation exists within the specification.

### Offline-capable

The Core does not satisfy this criterion because the specification does not define a self-contained delegated-authority artifact that can be created, delegated, presented, and independently verified without PDP involvement. The normal AuthZEN model is the evaluation of an authorization request by a PDP at runtime. Although the specification is transport-agnostic and permits locally deployed or embedded PDPs, local communication is not equivalent to offline delegation. The ability to evaluate authorization decisions without an external network connection should not be confused with the ability to create, transfer, and verify delegated authority offline.

The same conclusion applies when the ecosystem specifications are considered. Neither Policy Store, COAZ, nor COAZ-MCP defines a self-contained delegated-authority artifact that can be presented and independently verified outside the PDP evaluation model. Consequently, the ecosystem provides no standardized basis for offline delegation. While some deployments may reduce network dependencies through locally available policy artifacts or locally deployed PDPs, runtime authorization remains dependent on a PDP or policy engine rather than a self-verifying delegation artifact.

More precisely, locally deployed or disconnected PDPs may support offline policy evaluation, but they do not provide offline delegation.

## AuthZEN-specific Considerations

### Treat Policy Store portability carefully

AuthZEN standardizes the container and deployment unit for policies, schemas, entities, and issuer configuration, enabling these artifacts to be packaged, exchanged, and reused across implementations. However, Policy Store portability should not be confused with delegation interoperability. While AuthZEN standardizes the packaging and exchange of policy artifacts, the semantics of delegation, attenuation, and revocation remain specific to the underlying capability system implementation and architecture. Consequently, a policy store may be portable between systems, while the resulting delegation behavior may differ across authorization engines and deployments.

### Search results should not be interpreted as proof of capability or delegation

AuthZEN Search is designed for discovery rather than for conveying authority. The specification states that "Search APIs provide lists of resources, subjects or actions that would be allowed access." Search results indicate what may be permitted under the PDP's current state, but they do not by themselves constitute proof of delegated authority, capability possession, or authorization grants.

### Search can create an information-disclosure risk

Reverse-query APIs inherently expose information derived from the current state of the authorization system. If made available to insufficiently trusted callers, repeated or carefully crafted searches may reveal the existence of resources, access relationships, or characteristics of effective authorization policies. While Search can improve usability and discovery, it can also increase the risk of information disclosure. Appropriate authentication, query scoping, rate limiting, response minimization, and careful handling of diagnostic information should therefore be considered.

### Policy Store administration

Policy Store specification does not define a normative authorization model for administration of the Policy Store itself. Section 13, "Security Considerations," indicates that security, authorization, and operational controls of Policy Store itself are implementation responsibilities rather than standardized Policy Store behavior. Specifically, the specification does not prescribe who may create, modify, approve, publish, or delete policy bundles, nor does it define administrative roles, delegation models, or approval workflows for policy governance. These responsibilities are left to the implementing environment and its operational controls. As a result, organizations may adopt different approaches, ranging from repository-based access controls and CI/CD approval processes to dedicated PAP solutions. This separation preserves implementation flexibility, but it also means that governance, segregation of duties, and administrative access control for the Policy Store remain implementation-specific rather than standardized by AuthZEN.

### Decision Interoperability vs. Policy Interoperability

The Authorization API standardizes how a PEP requests and receives access decisions from a PDP, but it does not standardize policy languages, policy semantics, or policy execution models. A conformant PDP may use Cedar, Rego, Zanzibar-style relationships, or other policy systems, while exposing the same AuthZEN interface. AuthZEN achieves interoperability through interoperable authorization requests and access decisions, rather than through interoperability of policy semantics. One organization can consume access decisions produced by another organization's PDP, but AuthZEN does not require PDPs to share, exchange, interpret, or enforce one another's policies.

### Summary

AuthZEN can serve as a strong runtime authorization and policy-decision layer within an agentic authorization stack. Its primary value lies in standardizing authorization decisions and interoperability between authorization components, while supporting emerging AI-agent and tool-invocation scenarios through initiatives such as COAZ and COAZ-MCP.

However, AuthZEN is not a delegated-authority system. Delegation semantics, including delegation proof, attenuation, and revocation, remain outside the scope of Authorization API 1.0 and are provided by other components of the overall architecture. Likewise, policy representation is intentionally left to the underlying PDP implementation.

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

The surveyed specs are not six competitors for one job. They fall into three families, distinguished by where authority lives:

* Certificate capabilities (zCaps, UCANs). Authority lives in a portable, signed artifact. The delegatee holds it, can attenuate and re-delegate it offline, and presents it with proof of key possession. The verifier checks the chain locally; no authorization server exists.  
* Server-mediated authorization (the OAuth 2.0/2.1 lineage, AAuth). Authority lives in a server: the OAuth Authorization Server, or AAuth's Person Server with its mission state. Tokens reference that authority rather than embodying it, and delegation, attenuation, and revocation are acts performed at or by the server.  
* Policy decision systems (Cedar, AuthZEN). Authority lives in the verifier's policy store. There is no token of authority at all; the request presents identity evidence and the verifier evaluates rules it already holds. Cedar is the family's policy language. AuthZEN is its wire protocol, standardizing how an enforcement point (PEP) asks a decision point (PDP) for a verdict. The two cover complementary layers of the same architecture, much as zCaps and UCANs do for the certificate family.

The families also differ in which of the specification types from the Introduction (data model, protocol, possession mechanism, chainable proof methods) each spec actually covers. Much of the apparent unevenness between chapters reflects this: the chapters describe different layers of a potential stack, not six answers to the same question.

| Spec | Data model | Protocol | Possession mechanism | Chainable proofs | Policy language |
| :---- | :---- | :---- | :---- | :---- | :---- |
| OAuth 2.0 / 2.1 | Partial (scopes, JWT claims) | Yes (core focus) | Optional (DPoP, mTLS) | No (Token Exchange emulates, online) | No (opaque scope strings) |
| AAuth | Partial (act claim, tokens) | Yes (core focus) | Yes (signed requests, core) | Server-mediated (Mission Log) | No (deferred to PS policy) |
| zCaps | Yes (core focus) | Out of scope (invocation and revocation conventions only) | Yes (HTTP signatures, Data Integrity) | Yes (core focus) | Extension point (caveat, reserved) |
| UCANs | Yes (core focus) | Partial (invocation and revocation sub-specs) | Yes (signed envelope) | Yes (core focus) | Extension point (pol syntax, no vocabulary) |
| Cedar | No (no authorization artifact) | No | No | No | Yes (core focus) |
| AuthZEN | Partial (SARC request shape; no authorization artifact) | Yes (core focus: PEP/PDP evaluation and search) | No (out of scope) | No | No (explicitly out of scope; engine-specific) |

### Data Model Correspondence

The family taxonomy is easiest to see at the level of concrete fields. The table below maps each spec's terminology onto the common data model from the Overview: despite the differences in encoding, the same shape recurs wherever a spec defines an authorization artifact at all. (Cedar is absent because it defines no such artifact; its equivalents are schema entities in the verifier's store. AuthZEN's column reflects the request shape a PEP submits, not an authority artifact, per its chapter. GNAP is included for field comparison, although it is not otherwise surveyed in this paper. dSD-JWT is included on the same basis; see the Appendix.)

| Common field | OAuth 2.1 | AuthZEN | GNAP | zCaps | UCANs | AAuth | dSD-JWT |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| Agent | client\_id | context.agent (COAZ-MCP); no agent concept in core | client (rich object) | controller (DID) | Delegation: aud, Invocation: iss | agent identifier (agent\_token subject, stable across key rotation) | cnf key (JWK) named by the previous hop; no identifier |
| Principal | sub | subject | subject | implicit (in delegation chain) | Delegation: iss, Invocation: sub | person, vouched by the Person Server | holder of the base SD-JWT VC, bound by its cnf |
| Invocation / key binding | Optional, via DPoP | no | Yes (DPoP, HTTPSig, mTLS) | Yes: HTTP signatures plus Capability-Invocation header, or Data Integrity proof | Yes: its own signed-envelope mechanism; keys resolved from DIDs | Yes: HTTP Message Signatures on every request; cnf binds tokens to the current key | Yes: Key Binding JWT per hop, with aud and nonce at presentation |
| Resource | scope or RAR (structured scope); see also htm/htu claims | resource and context | locations and datatypes | invocationTarget (URL or object) | specified in args | resource\_token (names resource and scope; optional account parameter) | none defined; left to the delegate payload (profile-defined) |
| Actions | scope (also htm) | actions | actions | allowedAction | cmd | scope (in resource and auth tokens) | none defined; left to the delegate payload (profile-defined) |
| Caveats / additional policy | n/a | context | ? | caveat (reserved, unused in deployment) | pol (policy statements) plus args | none in the token; mission context evaluated server-side at the PS | none defined; draft permits constraints as visible claims, no vocabulary |
| Delegation chain | only via Token Exchange | no | no | Yes: capabilityChain proof chains | Yes: proof chains (CID-linked) | act claim plus the server-held Mission Log | Yes: hops appended as disclosures, hash-bound via sd_hash or issuer_jwt_hash |
| Protocol (how is authorization requested?) | OAuth 2.1 itself | AuthZEN | GNAP | out of scope (uses GNAP, VCALM, and similar) | out of scope (transport left open) | AAuth itself | OpenID4VP transaction_data item |
| Revocation | RFC 7009 | n/a (Policy Store rollback is versioning, not revocation) | n/a | Yes: out of band, at the RS | Yes: Revocation sub-spec; also a revocation capability | specified scenarios, exercised by the issuing servers; tokens expire within the hour | Token Status List on the base credential only; no per-hop mechanism |

Two readings of this table reinforce the scorecard that follows. Empty and "n/a" cells are findings, not omissions: OAuth simply has no place to put a caveat or a chain, which predicts its rows 3 and 4 below. And the two cells where AAuth diverges from the certificate pattern (chain and caveats both live at the server, not in the token) are precisely what produces its server-mediated column throughout.

### The Merged Scorecard

Verdicts below are taken from the per-spec chapters, which carry the detailed reasoning; the notes here are compressed to one line. zCaps and UCANs are shown separately for completeness, but as their chapter establishes, they score as a family. Cedar and AuthZEN likewise share a family, though they cover different layers of it and their verdicts diverge accordingly. dSD-JWT is included from the Appendix for comparison.

| \# | Requirement | OAuth 2.0/2.1 | AAuth | zCaps | UCANs | Cedar | AuthZEN | dSD-JWT |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 1 | Accountable (agent vs. principal/operator) | Yes with asymmetric client auth; weaker with shared client\_secret | Yes: distinct authenticated roles, act claim | Partial: signed edges, but principals are opaque DIDs | Partial: same as zCaps | Yes at the modeling level; not a cryptographic delegation edge | Partial: COAZ-MCP separates subject.id from context.agent; no delegation proof or operator binding | Partial: signed hops, but delegatees are bare JWKs, not identifiers |
| 2 | Resistant to confused deputy | No: scopes attenuate but do not designate | Partial: token modes bind designation; identity-based mode is ambient | Yes, by construction | Yes, by construction | No: identity-based, mitigation only | Partial: SARC carries designation per request; no delegation artifact, PEP-dependent | Partial: chain evaluated, but no resource or action in the artifact |
| 3 | Represent authorization policies | Partial: scopes and RAR; extension-dependent | Partial: policy intentionally outside the protocol | Partial: caveat slot reserved, unused, no vocabulary | Partial: pol syntax defined, no vocabulary | Yes, except the advisory tier; core competence | Partial: Policy Store packages artifacts; semantics stay engine-specific (core: out of scope) | No: no policy slot; terminal hop is the only control |
| 4 | Chainable | No: single-hop; Token Exchange emulates multi-hop online | Yes, server-mediated: act claim plus Mission Log; sub-agent nesting single-level | Yes: parentCapability plus capabilityChain | Yes: hash-linked proof chains | No: no chain artifact; stack-dependent | No: consumer of delegation chains, not a provider | Yes: hash-bound hops rooted in the issuer signature |
| 5 | Cross-organizational / locally verifiable | Conditional: JWT access tokens yes, opaque tokens no | Yes: cross-domain by design; trust via configured relationships | Yes: local cryptographic verification, DID principals | Yes: same, via CID resolution | Partial: evaluation local, policy store unilateral | Partial: cross-vendor decisions interoperate; relying party trusts the remote PDP | Yes: local verification, no authorization server |
| 6 | Attenuated | Via the AS only (scope narrowing, Token Exchange); not by the holder | Partial, differs by mode: sub-agent yes, call chaining intentionally no | Yes: monotonic, verifier-enforced | Yes: monotonic, verifier-enforced | Partial: forbid-composition shrinks authority, but no subset check between grants | No: attenuation left to the underlying policy engine | No: payload is arbitrary; narrowing deferred to a profile |
| 7 | Self-revocable (holder or chain) | Partial: RFC 7009 plus short lifetimes; no chain concept | Partial: revocation by the servers holding authority, not the holder | Yes, at the RS revocation endpoint | Yes, via the Revocation sub-spec | Partial: policy store change by the administrator | No: policy change is administrative, not holder revocation | No: status list on the base credential only |
| \+ | Authentication / Proof of Possession | Optional (DPoP, mTLS); bearer remains the deployed default | Yes: every request signed, core mechanism | Yes: HTTP signatures or Data Integrity proofs | Yes: signed envelope, nonce, time bounds | No: out of scope; evidence-issuance checks only | No: out of scope; bearer OAuth is the recommended default | Yes: Key Binding JWT per hop, aud and nonce |
| \+ | Privacy of delegation chain | High by absence: no chain exists to disclose | Partial: history held at the PS, disclosure deployment-defined | No: full chain visible to carrier and verifier | No: same | Possible by architecture: carrier sees nothing, verifier sees everything | No (undefined): no chain representation exists | Partial: alternative payloads selectively disclosed; hops visible |
| \+ | Offline-capable | Partial: issuance online; JWT verification offline | Partial: verification survives on cached keys; obtaining authority online | Yes for delegation and verification | Yes for delegation and verification | Partial: evaluation offline; granting online and administrative | No: local PDP evaluation is not offline delegation | Delegation yes; verification if the root key resolves offline |

### Reading the matrix

Four patterns are worth drawing out, because they organize everything else in this chapter.

The capability family owns the chain rows. On the criteria that concern the mechanics of delegated authority (confused deputy, chainable, cross-organizational, attenuated, self-revocable: rows 2 and 4 through 7), zCaps and UCANs are the only column pair that is Yes across the board, and they achieve it offline and holder-side. Every other column pays for those rows with a server dependency, a partial verdict, or both. This is the clearest architectural line in the survey, and it is a family property, not a property of either spec individually.

The identity side owns the semantic rows. On accountability (row 1\) and policy expression (row 3), the verdicts invert. The systems that know who everyone is (AAuth's authenticated roles, Cedar's typed principals, OAuth clients with asymmetric auth) can distinguish agent from operator natively; the capability family deliberately keeps principals opaque and delegates that distinction to a companion identity layer. AuthZEN's COAZ-MCP binding follows the identity side's pattern, separating the human subject from the acting agent, though without a delegation proof it stops at attribution. Likewise the richest policy expression in the survey (Cedar) sits in the one spec with no delegation mechanics at all, while the specs with delegation mechanics reserve policy extension points and leave them empty.

Chain privacy inverts too, and for a structural reason. The capability family's defining artifact, the portable chain, is exactly what its privacy row suffers from: the carrier and the verifier both see the whole provenance log. The systems with no portable chain get carrier-side privacy for free, at the cost of an all-seeing server or policy store. Neither side offers the checklist's actual wish (principal and verifier know all, intermediate carrier does not) as a mechanism; the closest approximations are workarounds (ephemeral keys) or deployment choices (PS disclosure policy).

No column is complete. Every spec has at least one No or Partial in the required rows 1 through 7 The capability family misses semantics (rows 1 and 3); the server-mediated family misses holder-side and offline mechanics (rows 4, 6, 7 in various ways); the policy decision family misses the delegation artifact entirely (row 4, and for Cedar row 2 as well; AuthZEN's SARC request carries designation with each decision, which earns it a Partial on row 2 that pure identity evaluation does not get). The AuthZEN column is the sparsest in the matrix, and its chapter is explicit about why: the spec addresses the decision interface layer, so its No verdicts mark a different layer rather than a failed attempt at delegation. This is the observation that motivates both the delegation-stack section and the gaps section below.

### Server-Mediated vs. Certificate Delegation

AAuth and the certificate capability family aim at the same use case (autonomous agents acting across organizational boundaries under delegated, revocable, auditable authority) and give opposite answers to the central question of where authority lives. The contrast is the sharpest one available in this survey, precisely because so much else about them agrees: both reject bearer tokens for signed requests, both identify agents by key material, both record who delegated what to whom.

* Portability. A certificate capability is self-contained: the chain travels with the invocation and any verifier can check it. AAuth's delegation history lives in the Person Server's Mission Log; verification of standing depends on authenticated interaction with the server that holds the state.  
* Offline delegation. The capability family delegates by signing, with no round trip. AAuth has no offline delegation; obtaining and extending authority are online protocol exchanges. This is the efficiency and partition-tolerance line.  
* Attenuation. The capability family enforces algebraic, monotonic attenuation at every link: a child provably holds a subset. AAuth deliberately rejects the subset rule for call chaining; the Person Server evaluates each hop against mission context, which its specification argues is more flexible than algebraic rules. This is a genuine philosophical split (constraint by algebra versus constraint by governance), not an omission on either side.  
* Revocation locus. In the capability family, anyone in the chain can revoke, and the enforcement point learns of it. In AAuth, the servers holding authority revoke; the holder of a credential cannot act on the credential itself.  
* Key lifecycle. AAuth delegations reference stable agent identifiers, with the cnf claim binding tokens to the current key; identity survives key rotation, at the cost of lifecycle questions about delegations that straddle a rotation. The capability family binds delegations to keys directly, and typically sidesteps rotation with short-lived, delegation-specific keys.

For agentic delegation, the trade reduces to this: the certificate model gives stronger guarantees (offline, locally verifiable, algebraically attenuated, holder-revocable) at the price of thin semantics, while the server-mediated model gives richer semantics and governance (missions, consent, authenticated roles) at the price of a live, trusted server on the path of authority.

### The Policy Decision Family as Complement

Cedar is, almost row by row, the complement of the capability family: strongest exactly where zCaps and UCANs are weakest (policy expression, agent-vs-operator modeling, fine-grained revocation, composition of authority) and weakest exactly where they are strongest (no chain, no portable grant, no holder-side anything). Read as a competitor it fails the survey; read as a component it fills the survey's emptiest cell.

The specific fit: both zCaps and UCANs reserve a policy extension point (caveat; pol) and ship no vocabulary for it. Cedar is a mature, analyzable policy language with no carrier. A capability chain whose caveats were expressed in a Cedar-class language would give the family what its chapters identify as their highest-value missing piece, with one property nothing else in the survey offers: because Cedar is non-Turing-complete and formally analyzable, delegation policies could be audited and subset relationships proven offline, at delegation time rather than invocation time. The reverse composition also holds: Cedar deployments that need portable, cross-organizational grants need exactly the carrier the capability family provides.

AuthZEN extends the same complement to the protocol layer, and makes the composition concrete. Cedar has no standard request/decision interface; AuthZEN supplies exactly that. In a composed stack, a capability verifier can act as a PEP: it checks the chain and proof of possession locally, then consults a PDP over AuthZEN for the caveat evaluation the capability family leaves unspecified. COAZ points the same direction from the other side, mapping live protocol operations (including MCP tool calls, via COAZ-MCP) into decision requests that reflect the actual invocation parameters. None of this requires the capability chain to change shape; it fills the empty caveat slot with an evaluation pipeline that already exists.

This suggests the shape of a composed delegation stack, assembled from surveyed parts:

1. Entry point (identity layer): an identity system (OIDC, Verifiable Credentials) makes the human or organizational judgement call and binds operator to agent; the zCap specification itself recommends this split.  
2. Authority flow (carrier layer): a certificate capability chain carries the delegated, attenuated authority across hops and organizations, offline and locally verifiable.  
3. Restrictions (policy layer): caveats within the chain expressed in a standardized, analyzable policy language, evaluated at each hop and provable between hops; where evaluation is handed to a PDP, AuthZEN is the standard interface for asking.  
4. Presentation (possession layer): signed invocation of the transport in use (HTTP Message Signatures, Data Integrity proofs, DPoP-style binding), with keys held outside the agent per the Introduction's requirements.

No surveyed spec provides more than two adjacent layers of this stack, and no pair of them currently interoperates by standard. That observation leads directly to the gaps.

### Gaps

Reading down the merged matrix, the recurring Partial and No cells cluster into six gaps that no surveyed specification fills. These are candidates for working group attention, ordered roughly by how directly they block agentic delegation.

1. A standard policy and caveat vocabulary for delegation control. Both certificate capability specs have the extension point and neither ships vocabulary; AAuth places policy outside the protocol; OAuth scopes are opaque strings; AuthZEN standardizes the decision request while leaving policy semantics to the engine. The concrete missing terms are the checklist's own examples: undelegatable, re-delegation constraints, delegation depth. Cedar demonstrates that such a vocabulary can be expressive and formally analyzable at once. This is the gap where existing pieces come closest to composing, and arguably the highest-value item.  
2. An advisory tier. No surveyed system can express a request whose violation is not refused ("please do not re-delegate outside the org"): every evaluation model in the survey is binary enforce/reject. Advisory semantics matter for agentic delegation because agents cross governance boundaries where hard enforcement is impossible but recorded intent still has value (for audit, for liability, for downstream policy).  
3. Agent-vs-operator semantics and identity binding. The capability family cannot say which principal is the accountable legal entity; the identity-centric systems can, but only inside their own trust domain. What is missing is a standardized companion mechanism (the VC-based binding both capability chapters gesture at) so that accountability to an operator survives cross-organizational, multi-hop delegation without collapsing back into identity-based authorization. AuthZEN's open Issue \#416, which requests a delegation\_chain structure in the Subject for AI-agent scenarios, is this same gap filed independently by an adjacent working group.  
4. Delegation chain privacy. The checklist's formulation (principal and verifier know the full story, the intermediate carrier does not) is offered by no spec as a mechanism. The capability family discloses everything to everyone on the path; the server-mediated and policy families relocate the problem to an all-seeing server. Selective disclosure or blinding of chain interiors is an open design problem. The closest existing mechanism is dSD-JWT (see the Appendix), which inherits selective disclosure from SD-JWT and lets a delegatee reveal one of several pre-signed alternative payloads, although the hops themselves remain visible.  
5. Offline and decentralized revocation. Every surveyed revocation mechanism is state at an enforcement point that must be reached: an RS endpoint, a server's grant, a policy store. For the local-machine and LAN agentic use cases the checklist highlights, revocation semantics under partition are unspecified everywhere. UCAN's open dissemination model (gossip, DHT) is the closest starting point, but its guarantees are undefined. The dSD-JWT draft states the problem from the delegator's side: an individual holder has no channel to distribute revocation to verifiers, so it falls back to short expiry.  
6. Cross-organizational policy exchange. Even where evaluation is local, policy semantics are unilateral: Cedar's store belongs to one verifier, an AS's scope definitions to one provider, a PS's mission policy to one server. There is no standard by which one organization hands another a policy fragment with agreed meaning. AuthZEN's Policy Store draft is a partial answer: it standardizes the packaging and exchange of policy artifacts while explicitly leaving their semantics engine-specific, sharpening this gap without closing it. This is the cross-organizational criterion's unmet half, and it compounds gap 1: a shared caveat vocabulary is also the natural interchange format.

## Conclusion

The survey's answer to "which specification should agentic delegation use" is that the question cannot have a general answer. The certificate capability family supplies the only viable backbone for the authority flow itself as first-class data element and throughline: it alone meets the chain criteria, offline and holder-side, and it alone is immune to the confused deputy by construction. But it is a backbone, not a solution; its own chapters identify the missing semantics, and those semantics are exactly what the identity-centric and policy systems in this survey already know how to express. The near-term work is therefore not a seventh competing spec but composition: a standardized policy vocabulary that can ride in a capability chain, a standardized identity binding at the chain's root (whether this be centralized in a “Mission Log” or not), a standard decision interface (which AuthZEN already supplies) where caveat evaluation is handed to a policy engine, and revocation and privacy mechanisms designed for the partitioned, multi-organizational environments agents actually operate in. The gaps list above is, in effect, the specification outline for that work.

## Appendix: Other Systems Considered

This appendix covers specifications that were examined during the survey but that did not warrant a full chapter, either because they address only one layer of the stack described in the Introduction or because they are too early in their standards lifecycle to evaluate on equal footing with the main candidates. Each entry is scored against the same checklist so that it can be placed alongside the merged scorecard in the Synthesis chapter.

### Delegated SD-JWT (dSD-JWT)

#### Overview

[Delegated SD-JWT](https://datatracker.ietf.org/doc/draft-gco-oauth-delegate-sd-jwt/) ("dSD-JWT", an individual IETF draft at revision 00) is an extension to [Selective Disclosure JWT (SD-JWT, RFC 9901)](https://datatracker.ietf.org/doc/rfc9901/) that lets the holder of an issued credential hand a bounded, verifiable authorization to another party without involving the issuer. Rather than defining a new credential type, it appends a delegation hop to an existing SD-JWT as an additional disclosure. Each hop is a JSON payload chosen by the delegator, wrapped in a Key Binding JWT signed by the key the previous hop named in its `cnf` claim, and hash-bound to the prior state of the credential (via `sd_hash` or `issuer_jwt_hash`). A hop may name a further `cnf` key, in which case the chain can continue, or omit it, in which case the chain terminates at that delegatee.

The draft is deliberately claim-agnostic. It defines the chaining and key-binding mechanics and says nothing about what the delegated claims mean. Vocabulary for agents, resources, actions, or limits is left to a higher-level profile. The one implementation examined for this appendix (the [equs-credentials-sdk](https://github.com/equs-ai/equs-credentials-sdk), where the feature is marked experimental and disabled by default) names the [Agent Payments Protocol (AP2)](https://ap2-protocol.org/) as the first such consumer, but that profile is not part of the draft and its dSD-JWT binding was not available for review.

This makes dSD-JWT a chainable proof method plus a possession mechanism, in the terminology of the Introduction, rather than a full authorization data model. It is closest in spirit to the certificate capability family, but its root of authority is a credential issuer rather than a resource owner, and it lacks the two properties that define that family in this report: an enforced attenuation rule and a resource and action model. For those reasons it is placed here rather than in a chapter of its own.

Reference points: the draft text (sections 4 through 8) and the SDK's design note, source, and tests. The SDK delegates chain verification to a separate crate that was not reviewed directly, so verdicts below rest on the current internet-draft (-00), the SDK call sites, and the SDK's test suite.

#### Scorecard

| \# | Requirement | dSD-JWT | Notes |
| ----- | ----- | ----- | ----- |
| 1 | Accountable (agent vs. principal/operator) | Partial | Every hop is signed by the key the previous hop named, so each edge is attributable. But delegatees are identified by raw JWKs in `cnf`, not by DIDs or any stable identifier, and there is no agent or operator field. Weaker than zCaps and UCANs, whose principals are at least DIDs. |
| 2 | Resistant to confused deputy | Partial | Authority travels in the signed artifact and the verifier evaluates the chain rather than the presenter's standing. However, the artifact carries no designated resource or action, so whether designation and authorization are bound together depends entirely on the profile that defines the payload vocabulary. |
| 3 | Represent authorization policies | No | No caveat or policy slot. The only delegation control is structural: a hop that omits `cnf` ends the chain. Draft section 8.2 says a delegator "MAY" add constraints as visible claims, with no defined vocabulary. No advisory tier. |
| 4 | Chainable | Yes | Multi-hop, holder-side, each hop hash-bound to the prior state and rooted in the original issuer signature. Covers operator to agent and agent to sub-agent. |
| 5 | Cross-organizational / locally verifiable | Yes | Verification is local and needs no authorization server. The root issuer key is obtained through DID resolution or issuer metadata; hop keys are embedded JWKs. The format does not answer whether a given hop key is trusted for a given resource. |
| 6 | Attenuated | No | Neither the draft nor the implementation enforces narrowing. The delegate payload is arbitrary JSON, and the only validation rejects reserved keys. Attenuation is explicitly deferred to a higher-level profile. |
| 7 | Self-revocable | No | Only Token Status List on the base credential, which revokes the whole tree at once. Draft section 8.3 acknowledges that an individual holder "can not easily distribute revocation information to Verifiers" and suggests a short `exp` as mitigation. No per-hop revocation and no revoker role. |
| \+ | Authentication / Proof of Possession | Yes | Key Binding JWT at every hop, with `aud` and `nonce` binding at presentation. Not a bearer format. |
| \+ | Privacy of delegation chain | Partial | The only artifact in this survey that inherits selective disclosure. A delegator can pre-authorize several alternative payloads in one signature and the delegatee reveals one. However, the hops themselves are always visible to the verifier, and selective disclosure within a single delegate payload is not supported. |
| \+ | Offline-capable | Delegation yes; verification depends on key resolution | Delegation is a local signing act with no callback to the issuer. Verification is offline only when the root key resolves without network access (for example did:key or a cached resolution). |

#### Placement relative to the survey

Against the merged scorecard, dSD-JWT scores well on exactly the rows where the certificate capability family scores well (chainability, proof of possession, offline delegation) and poorly on the rows that make that family a capability system (attenuation, resource and action model, policy vocabulary, revocation below the root). Two features are novel relative to the surveyed specs and bear on the Gaps list in the Synthesis chapter:

* Selective disclosure in the chain (Gap 4). A delegatee can hold several pre-signed alternatives and disclose only the one it uses. This is the closest any surveyed artifact comes to hiding chain interiors from a verifier, although it hides alternative payloads rather than intermediate parties.  
* An explicit statement of the holder-side revocation problem (Gap 5). The draft's security considerations state directly that a holder, unlike an issuer, has no channel to distribute revocation, and fall back to short expiry. This is the offline-revocation gap named from the delegator's side.

If a profile for agentic delegation were layered on dSD-JWT, it would need to supply a resource and action vocabulary, an attenuation rule checked by the verifier at each hop, and a per-hop revocation mechanism. Those are the same items the Gaps list assigns to the certificate capability family, which suggests dSD-JWT is better understood as an alternative envelope for that family's semantics than as a competing model. Its main attractions over the DID-native envelopes are its fit with existing SD-JWT VC wallets and its inherited selective disclosure.
