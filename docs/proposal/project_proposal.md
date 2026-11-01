# AgentPassport: A Know-Your-Agent Trust Layer for Autonomous Agents on Google AX

**Project Proposal** · Hacettepe University, Department of Computer Engineering
Team: Ömer Burak Taşkın, Alper Melkeli, Yiğit Mete Tuygun, Fikret Kerem Dinçer · Advisor: [advisor name]

> Proposal form answers, based on the [design document](../AgentPassport_Design_Document.pdf) (Draft v0.3).
> Aligned with Google AX release v0.3.0 and `main` (`ax.io/v1alpha1`), checked October 2026.
> Figures are taken from the design document; image files live in [`../figures/`](../figures/).

## Contents

1. [Project Summary](#1-project-summary)
2. [Problem Definition and Literature Review](#2-problem-definition-and-literature-review)
3. [Solution Plan](#3-solution-plan)
4. [Methodology](#4-methodology)
5. [Outcome and Impact](#5-outcome-and-impact)

---

## 1. Project Summary

**Motivation.** Autonomous LLM agents are becoming a new kind of internet user: they browse, call APIs, place orders and move money on behalf of people. Google AX, an open-source orchestrator built on Agent Substrate, already runs such agents in isolated sandboxes and limits which hosts they may reach. This protects the cluster from the agent, but not the services on the receiving end. When an online shop receives an order from an agent, it cannot tell:

- which agent sent it, and which code and model drive it;
- which human is accountable for it;
- whether that human authorised this purchase;
- whether the agent was hijacked minutes ago by a prompt injection hidden in a web page.

Banks solved a similar problem for people with Know Your Customer (KYC). The emerging Know Your Agent (KYA) idea applies it to software agents, but no open implementation tied to a real agent platform exists.

![Figure 1: The four pillars of Know Your Agent, the everyday analogy for each, and how AgentPassport implements it.](../figures/kya_pillars.png)

*Figure 1: The four pillars of Know Your Agent, the everyday analogy for each, and how AgentPassport implements it.*

**Solution plan.** We will design, implement and evaluate AgentPassport, a KYA layer that runs next to AX without forking it, using only AX's documented extension points:

- An **admission proxy** in front of the AX API authenticates the launching person (university SSO) and checks every task against policy.
- A **passport authority** issues each task a short-lived, revocable passport (a signed JWT). The passport names the agent, its accountable principal and a "visa" of allowed hosts, actions and spending limits.
- Using AX's own Gateway resource, each sandbox may reach only one destination: our **KYA signing proxy**. The proxy enforces the visa and signs every outbound request (RFC 9421) with a per-task key that never enters the sandbox.
- A **verifier SDK** for Spring Boot and Go lets third-party services check these requests in a few lines of code and ask for human approval on risky actions.
- A **trust observer** combines rule penalties with an isolation-forest anomaly score into a per-task trust score. When it drops, the task is suspended through AX and its passport revoked.
- Delegation chains, a tamper-evident audit ledger and a web console complete the system.

**Expected outcome and impact.** Deliverables are open-source components, a passport and signature specification, verifier SDKs, and a reproducible suite of 30+ attack scenarios. Our targets are:

- block at least 95% of in-scope attacks;
- add under 10 ms median latency per request;
- suspend a hijacked agent within 5 seconds.

In our demo scenarios, a prompt-injected payment is rejected twice (by the signing proxy and the verifier) and the agent is suspended; a looping agent is stopped before it exhausts its budget; and a stolen passport is useless, because only the signing proxy holds the signing key. Parts of the design follow AX's roadmap and can be contributed upstream.

---

## 2. Problem Definition and Literature Review

### 2.1 Problem statement

Let a principal P (an authenticated human) launch an AX task T that runs an LLM-driven agent inside a sandbox. During its lifetime, T emits a stream of outbound requests r₁, r₂, … to external services called relying parties (RPs). For every request rᵢ, an RP must compute a decision

```
decide(rᵢ) ∈ { allow, deny, step-up (ask the principal) }
```

cryptographically, without trusting the agent or the network. The decision must answer the four questions of Know Your Agent (KYA) [4–6]:

1. **Identity:** which agent (task, container image digest, model) sent rᵢ?
2. **Principal binding:** which authenticated human is accountable for it?
3. **Authorization:** is rᵢ inside an explicit, human-approved scope (hosts, actions, amounts, time)?
4. **Behaviour:** is T still behaving normally, or has it been hijacked or stuck in a loop?

In addition, the platform must bound damage. If the trust score s(T) of a task (a value in [0, 1] computed from its runtime behaviour, see S5) falls below a threshold θ, T must be suspended within a bounded time Δ (target Δ < 5 s).

### 2.2 Context

Google AX [1] is a declarative orchestrator for agent workloads built on Agent Substrate [2]. Users describe four kinds of resources in YAML (`ax.io/v1alpha1`):

- **Tasks:** image, command, resources, workspaces, gateway reference;
- **Workspaces:** Git repositories, MCP servers, skills, optional natural-language setup goal;
- **Gateways:** listeners and a host/port egress allowlist;
- **Models:** provider, model ID, secret reference.

The control plane has three services:

- **ax-server:** a stateless gRPC API that validates manifests, persists state in Redis and publishes events;
- **ax-controller:** a reconciler that consumes Redis Streams, provisions sandboxed actors on Agent Substrate and applies Gateway egress policy;
- **ax-task-runner:** PID 1 inside every sandbox; it prepares workspaces, serves a metadata API and runs the agent command.

Idle tasks can be suspended and resumed in under a second. Unlike a local coding assistant whose sub-agents are watched by the developer, cluster agents run unattended and at scale. Therefore isolation, limits and accountability must come from the platform itself.

![Figure 2: Local sub-agents versus agents as managed cluster workloads.](../figures/analogy.png)

*Figure 2: Local sub-agents versus agents as managed cluster workloads. When nobody is watching each agent, isolation, limits and accountability must come from the platform.*

![Figure 3: AX architecture.](../figures/ax_architecture.png)

*Figure 3: AX architecture. ax-server persists manifests in Redis; ax-controller consumes the event stream, provisions sandboxed actors on Agent Substrate and applies each task's Gateway egress policy.*

AX is pre-1.0: its API is still v1alpha1, and its README warns that major breaking changes are likely. Four documented extension points let us add a KYA layer without modifying AX:

1. the CLI `--server` flag (or `$AX_SERVER`);
2. the public custom-runner contract (`spec.image` plus the embeddable `runner` package);
3. the `Gateway` resource, which can restrict a sandbox's egress to a single destination;
4. the `SuspendTask` RPC.

AX's roadmap lists the following as planned work [3]: SPIFFE identities for tasks and gateways; governance (access control, policy enforcement, budget guardrails, audit); task-level token/timeout budgets and approval policies; and OpenTelemetry telemetry. None of these are available in the current release.

### 2.3 Inputs

- **Principal identity:** OIDC ID token from the university SSO (subject, e-mail, groups).
- **Task manifest:** AX Task spec plus the Workspace and Gateway resources it references, the AX Model the platform uses, and an optional requested visa.
- **Policy:** an `AgentPolicy` per user group (allowed images and models, maximum visa, passport TTL, delegation depth, debug permission, trust thresholds).
- **Runtime traffic:** every outbound HTTP(S) request of the agent.
- **Runtime signals:** events from the signing proxy, runner and verifiers, plus AX lifecycle events (`WatchTask`).

### 2.4 Outputs

- **Passport:** a signed, short-lived token per task carrying identity, principal, visa and validity.
- **Signed requests:** every outbound request carries the passport and an RFC 9421 signature.
- **Verification decision** at the RP: allow, deny (401/403), or approval-required.
- **Trust score** per task, with automatic actions: normal, step-up, or suspend and revoke.
- **Audit trail:** tamper-evident and queryable per task and per principal.

### 2.5 Gap analysis (AX today vs. KYA requirements)

| Concern | AX today | Gap |
|---|---|---|
| Isolation | gVisor/microVM sandbox with CPU/memory limits | None; we rely on it |
| Who launched the task | Anyone who can reach ax-server; API authentication is not documented | No accountable human attached |
| Agent identity | Task name only; SPIFFE on roadmap | Outsiders cannot verify the caller |
| Permissions | CPU/memory limits; host/port egress allowlist via Gateway | No action names, spending limits or human approval |
| Outbound traffic | Per-task Gateway with host/port allowlist | No request signing; external services cannot tell which agent is calling |
| Third-party verification | Not in scope | Services cannot check agent requests |
| Runtime monitoring | Phase and condition changes (`WatchTask`) | Hijacked or looping agents go undetected |
| Emergency stop | `SuspendTask` exists, but must be triggered manually | No automatic reaction |
| Audit | Task state in Redis | No tamper-evident history |
| Sub-agents | Tasks can form trees | A child may get more power than its parent |

### 2.6 Literature review

- **Know Your Agent.** There is no single KYA standard yet. Identity-verification companies publish their own definitions [4–6]. Across them, four pillars recur: identity, principal binding, authorization, and behaviour/reputation. Their products mostly target finance and bind an agent to a human through document-based KYC. The Decentralized Identity Foundation now hosts KYA-OS (formerly MCP-I, donated by Vouched), an open specification that identifies agents with W3C decentralized identifiers and expresses delegation as verifiable credentials [32]; it is a protocol specification, not an integration with an agent orchestrator. None of these provides an open implementation integrated with an agent orchestrator.
- **Agent identity and accountability in research.** Chan et al. [18] analyse three measures that give visibility into deployed agents: agent identifiers, real-time monitoring and activity logging. IDs for AI systems [19] extends this to per-instance identifiers that also record parent–child relations when an agent spawns sub-agents. South et al. [20] propose extending OAuth 2.0 and OpenID Connect with agent-specific delegation credentials that tie an agent to the user who authorised it. These works are conceptual: they propose mechanisms and discuss trade-offs, but do not build and measure them on an agent platform. AgentPassport implements identifiers (passport), monitoring (trust observer), logging (audit ledger) and delegation for real AX tasks and measures them.
- **Request signing and agent identity.** HTTP Message Signatures (RFC 9421) [7] sign and verify selected components of an HTTP message, such as method, authority, path and chosen headers. The IETF Web Bot Auth working group profiles RFC 9421 for automated clients and adds a `Signature-Agent` header and a JWKS-based key directory [8]. RFC 7800 [9] defines the `cnf` claim, which binds a token to a key: a relying party that demands proof of possession rejects a stolen token used without that key. DPoP (RFC 9449) [21] applies the same idea to OAuth access tokens. SPIFFE [10] defines a URI format (`spiffe://<trust-domain>/<path>`) for workload identities; AX plans to adopt it [3]. IETF work on workload identity (WIMSE) is being extended to AI agents; one individual draft proposes credentials that bind an agent's identity to its owner's [22], the same principal binding our passport carries.
- **Human login and delegation.** The OAuth 2.0 device grant (RFC 8628) [11] lets a client without a browser, such as a command-line tool, obtain tokens after the user logs in through a browser on another device. OAuth Token Exchange (RFC 8693) [12] lets an authorization server issue a new token, optionally with a narrower scope and audience, and record delegation in the `act` claim. Macaroons [23] introduced holder-side attenuation: caveats appended to a credential can only narrow its authority. Biscuit [13] brings this to public-key tokens with Datalog-based checks, so a holder can attenuate a token without contacting the issuer. Our delegation follows the issuer-mediated token-exchange model; offline Biscuit attenuation is our research comparison.
- **Agent payments.** Google's Agent Payments Protocol (AP2) [14] records the user's intent and approval of a specific cart as cryptographically signed mandates (verifiable credentials); this inspires our approval grants bound to one exact request. Visa's Trusted Agent Protocol [24] lets merchants recognise registered shopping agents through RFC 9421 signatures aligned with Web Bot Auth, which shows that request signing for agents is already used in commerce.
- **Attacks on agents.** Indirect prompt injection hides instructions in content the agent reads [15]. OWASP ranks prompt injection first among LLM application risks [16] and catalogues agent-specific risks in a separate Top 10 for agentic applications [25]; Deng et al. [26] survey the threats to agents more broadly. AgentDojo [27] provides realistic agent tasks with prompt-injection test cases for evaluating attacks and defences. Nasr et al. [28] bypassed 12 recently published defences against jailbreaks and prompt injection with adaptive attacks, mostly at success rates above 90%, although most of those defences had reported near-zero attack success. Text-level detection is therefore not a reliable security boundary. CaMeL [29] instead enforces capabilities and data-flow policies in a system layer around the model. We take the same system-level stance at the network edge: we limit what the agent can *do*, not what it *thinks*.
- **Tamper-evident logging.** A hash chain makes changes to past entries evident, but auditing it takes time linear in the log length. Crosby and Wallach [30] propose a Merkle-tree history structure whose audit and lookup proofs are logarithmic. Our ledger uses a hash chain with signed checkpoints; tree-based proofs are a possible extension.
- **Zero trust.** NIST SP 800-207 [31] defines zero trust architecture: no implicit trust based on network location, and an access decision for each request to a resource. AgentPassport applies this model to agent egress: every outbound request is authenticated and authorised on its own, and trust in a task is re-evaluated continuously.
- **Anomaly detection.** Isolation Forest [17] is an unsupervised method suited to our setting, where labelled attack data is scarce but benign runs are easy to collect.

### 2.7 Formal constraints

```
Effective visa:        V_eff = V_policy ∩ V_requested
Manifest binding:      manifest_hash = SHA-256(canonical(Task.spec ∥ Workspaces ∥ Gateway))
Delegation:            V_child ⊆ V_parent
                       budget_child ≤ remaining_budget(parent)
                       depth ≤ d_max
                       exp_child ≤ exp_parent
Request acceptance:    accept(r) ⇔ valid_sig(r, cnf.jwk) ∧ valid_passport ∧ jti ∉ RevocationList
                                   ∧ host ∈ V.hosts ∧ action ∈ V.actions
                                   ∧ (amount ≤ limits ∨ valid approval grant bound to digest(r))
```

Here `cnf.jwk` is the public per-task key named in the passport, `jti` is the passport's unique ID, and an approval grant is a short-lived token the passport authority issues after the principal approves one exact request (see S4).

### 2.8 Limitations and assumptions

- **Non-goals.** We do not make the LLM itself safe or detect injected text. We do not build a production human identity-verification service; we rely on existing OIDC. We do not modify AX or Agent Substrate. We do not handle real money.
- **Trust assumptions.** The Kubernetes control plane, sandbox isolation, ax-server, ax-controller and our own components are trusted. The agent and everything inside the sandbox are fully untrusted.
- **Residual risks.**
  - Misuse within the granted mandate (e.g. a bad order below the limit).
  - Data exfiltration through allowed hosts.
  - RPs that do not verify signatures.
  - The signing proxy is a high-value component, since it terminates TLS and holds task keys.
  - Egress enforcement depends on AX Gateway policies; continuous Gateway reconciliation is still on AX's roadmap.
  - AX's workspace bootstrap places a model API key inside the container, which conflicts with our "no long-lived secrets in the sandbox" goal.
  - AX's API may change, since it is still alpha.

### References

1. Google, AX: declarative orchestrator for autonomous agent workloads, github.com/google/ax, v0.3.0 / main, 2026.
2. Agent Substrate project, github.com/agent-substrate/substrate.
3. Google, AX roadmap, github.com/google/ax/blob/main/docs/roadmap.md.
4. Sumsub, "From AI Agents to Know Your Agent: Why KYA Is Critical for Secure Autonomous AI", sumsub.com/blog/know-your-agent, January 2026.
5. Entrust, "What is Know Your Agent (KYA)?", entrust.com/resources/learn/know-your-agent.
6. Vouched, "Know Your Agent: Building Trust Frameworks for AI", vouched.id/learn/blog/building-a-trust-framework-for-ai-agents-why-know-your-agent-matters.
7. A. Backman, J. Richer, M. Sporny, HTTP Message Signatures, RFC 9421, February 2024.
8. T. Meunier, S. Major, HTTP Message Signatures for automated traffic, draft-ietf-webbotauth-httpsig-protocol-01, IETF Web Bot Auth working group Internet-Draft (work in progress), October 2026.
9. M. Jones, J. Bradley, H. Tschofenig, Proof-of-Possession Key Semantics for JSON Web Tokens (JWTs), RFC 7800, April 2016.
10. SPIFFE project, Secure Production Identity Framework for Everyone, spiffe.io.
11. W. Denniss, J. Bradley, M. Jones, H. Tschofenig, OAuth 2.0 Device Authorization Grant, RFC 8628, August 2019.
12. M. Jones, A. Nadalin, B. Campbell, J. Bradley, C. Mortimore, OAuth 2.0 Token Exchange, RFC 8693, January 2020.
13. Biscuit authorization tokens, biscuitsec.org.
14. Google Cloud, "Announcing Agent Payments Protocol (AP2)", Google Cloud Blog, September 2025; specification at github.com/google-agentic-commerce/AP2.
15. K. Greshake, S. Abdelnabi, S. Mishra, C. Endres, T. Holz, M. Fritz, "Not what you've signed up for: Compromising real-world LLM-integrated applications with indirect prompt injection", Proc. 16th ACM Workshop on Artificial Intelligence and Security (AISec '23), pp. 79–90, 2023.
16. OWASP GenAI Security Project, OWASP Top 10 for LLM Applications 2025, LLM01:2025 Prompt Injection, November 2024; 2026 edition, August 2026.
17. F. T. Liu, K. M. Ting, Z.-H. Zhou, "Isolation Forest", Proc. IEEE International Conference on Data Mining (ICDM), 2008.
18. A. Chan et al., "Visibility into AI Agents", Proc. ACM Conference on Fairness, Accountability, and Transparency (FAccT), 2024. doi:10.1145/3630106.3658948.
19. A. Chan, N. Kolt et al., "IDs for AI Systems", RegML Workshop at NeurIPS 2024; arXiv:2406.12137.
20. T. South, S. Marro, T. Hardjono, R. Mahari, C. D. Whitney, D. Greenwood, A. Chan, A. Pentland, "Authenticated Delegation and Authorized AI Agents", arXiv:2501.09674, 2025.
21. D. Fett, B. Campbell, J. Bradley, T. Lodderstedt, M. Jones, D. Waite, OAuth 2.0 Demonstrating Proof of Possession (DPoP), RFC 9449, September 2023.
22. Y. Ni, C. P. Liu, WIMSE applicability for AI agents, draft-ni-wimse-ai-agent-identity, IETF individual Internet-Draft (work in progress), 2026.
23. A. Birgisson, J. G. Politz, Ú. Erlingsson, A. Taly, M. Vrable, M. Lentczner, "Macaroons: Cookies with Contextual Caveats for Decentralized Authorization in the Cloud", Proc. NDSS, 2014.
24. Visa, Trusted Agent Protocol, developer.visa.com/capabilities/trusted-agent-protocol, October 2025.
25. OWASP GenAI Security Project, OWASP Top 10 for Agentic Applications for 2026, December 2025.
26. Z. Deng et al., "AI Agents Under Threat: A Survey of Key Security Challenges and Future Pathways", ACM Computing Surveys 57(7), Article 182, 2025.
27. E. Debenedetti, J. Zhang, M. Balunović, L. Beurer-Kellner, M. Fischer, F. Tramèr, "AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents", NeurIPS Datasets and Benchmarks Track, 2024.
28. M. Nasr, N. Carlini, C. Sitawarin et al., "The Attacker Moves Second: Stronger Adaptive Attacks Bypass Defenses Against LLM Jailbreaks and Prompt Injections", USENIX Security Symposium, 2026 (arXiv:2510.09023).
29. E. Debenedetti, I. Shumailov et al., "Defeating Prompt Injections by Design", arXiv:2503.18813, 2025.
30. S. A. Crosby, D. S. Wallach, "Efficient Data Structures for Tamper-Evident Logging", Proc. 18th USENIX Security Symposium, pp. 317–334, 2009.
31. S. Rose, O. Borchert, S. Mitchell, S. Connelly, Zero Trust Architecture, NIST Special Publication 800-207, August 2020. doi:10.6028/NIST.SP.800-207.
32. Decentralized Identity Foundation, "DIF and Vouched Advance Agentic Identity with KYA-OS", blog.identity.foundation/kya-os, 2026; developed in DIF's Trusted AI Agents Working Group.

---

## 3. Solution Plan

### 3.1 Potential paths and the chosen plan

- **Path A – Wrap, don't fork (chosen).** Place chokepoints around AX:
  - in front of its gRPC API (admission proxy);
  - inside its public runner contract (KYA runner);
  - on the only network path out of the sandbox (AX Gateway → KYA signing proxy).

  AX stays unchanged, and only thin adapters depend on its alpha API.
- **Path B – Fork AX and add identity inside ax-server (rejected).** AX's API is v1alpha1 and changes often. A fork would be expensive to maintain and would conflict with AX's own roadmap.
- **Path C – Agent-side SDK where the agent signs its own requests (rejected).** The agent is untrusted. Signing keys inside the sandbox would be stolen by a hijacked agent.
- **Path D – Wait for AX's native identity and governance (rejected).** These features are on AX's roadmap without a timeline. Even when they arrive, they target platform-internal control (token and time budgets, mTLS between components), not authorization that external services can verify at the level of actions and amounts.
- **Fallbacks.**
  - If AX Gateway policies prove unreliable, we fall back to proxy environment variables (cooperative mode) and report this clearly.
  - If Agent Substrate cannot be installed, we build a minimal "fake AX" exposing the same gRPC API on plain Kubernetes pods.

Design principles:

- the agent is untrusted;
- chokepoints are preferred over trusting the agent to behave;
- every action traces back to a human;
- trust is continuous (short-lived, revocable passports);
- anyone can verify with standard libraries;
- adapters to AX are kept thin, because its API is alpha.

![Figure 4: AgentPassport architecture.](../figures/system_architecture.png)

*Figure 4: AgentPassport architecture. Purple: our components; grey: AX and Agent Substrate (unchanged); green: the outside world and the AX Gateway. Numbers follow a task's three phases: launch (login, admission, passport), act (signed requests checked by verifiers) and watch (events, trust score, suspension).*

![Figure 5: Sequence of a task's life with AgentPassport.](../figures/sequence.png)

*Figure 5: Sequence of a task's life with AgentPassport. Dashed arrows are responses or asynchronous events. Step 15 only happens if the trust score falls below the threshold.*

### 3.2 Steps, inputs and outputs

| # | Step | Inputs | Outputs |
|---|---|---|---|
| S0 | Platform setup | AX v0.3.0 / pinned main commit, Agent Substrate, cloud education credits | Running AX + Substrate cluster (3–5 nodes); AX Gateway egress experiment (sandbox reaches only the signing proxy) |
| S1 | Specification | KYA literature, RFC 9421/7800/8628, AX docs and roadmap, S0 findings | Passport claim format, signature profile, AgentPolicy schema, STRIDE threat model |
| S2 | Identity core | S1 specs, university OIDC (or Keycloak/Dex), AX gRPC types (`ax.v1alpha1.AX`) | Passport authority; admission proxy handling `UpdateTask` and Gateway/Workspace/Model RPCs; `kya` CLI; first passport issued for a real AX task |
| S3 | Enforcement | Passports and task keys from S2, AX `runner` package, AX Gateway resource | KYA runner image; KYA signing proxy (visa allowlist, TLS, signing, metering); `kya-egress-only` Gateway |
| S4 | Relying-party side | Signature profile, PA public keys and revocation list | Verifier SDKs (Spring Boot, Go), step-up approvals, demo shop and payments API |
| S5 | Trust observer | Events from signing proxy, runner, verifiers and `WatchTask`; 200+ benign runs | Per-task trust score; automatic step-up, `SuspendTask` and revocation |
| S6 | Delegation and audit | Parent passports, sub-task `UpdateTask` calls, events from all components | Child passports with budget reservation; hash-chained ledger with signed checkpoints |
| S7 | Console | Ledger, PA and observer APIs | Web dashboard: agents, approvals, incidents, audit explorer |
| S8 | Evaluation | All components, 30+ attack scenarios, benchmark scripts | Measurements vs. targets, ablation study, final report, demo |

### 3.3 Weekly timeline

W1 = first week of October 2026.

| Task | Weeks | Approx. dates |
|---|---|---|
| Literature review, AX v0.3 + Substrate setup, AX Gateway egress test (S0) | W1–W4 | Oct 2026 |
| Threat model and passport specification (S1) | W3–W7 | mid-Oct – mid-Nov |
| Passport authority + admission proxy (S2) | W6–W13 | Nov – Dec |
| KYA signing proxy + AX Gateway binding (S3) | W9–W17 | Dec – Jan |
| Demo shop API + verifier v0 (S4) | W14–W17 | Jan 2027 |
| Interim report and demo | W16–W18 | late Jan |
| *Semester break* | W18–W20 | early Feb |
| Verifier SDKs, Spring Boot + Go (S4) | W20–W26 | mid-Feb – Mar |
| Trust observer: rules + anomaly model (S5) | W22–W30 | Mar – Apr |
| Delegation chains + audit ledger (S6) | W25–W31 | late Mar – Apr |
| Console: dashboard, approvals (S7) | W27–W32 | Apr – early May |
| Attack suite and evaluation (S8) | W29–W35 | late Apr – May |
| Final report, poster, demo | W34–W38 | late May – Jun |

![Figure 6: Planned timeline.](../figures/timeline.png)

*Figure 6: Planned timeline. Purple: KYA core components; green: relying-party side; red: evaluation; brown: reports; grey: setup. Diamonds mark milestones M1–M4.*

### 3.4 Milestones

- **M1 (end of Nov 2026):** AX pinned and running; AX Gateway egress restriction verified; passport spec frozen; first passport issued for a real AX task.
- **M2 (end of Jan 2027):** vertical slice. A task is launched through the admission proxy with a passport; its requests are signed by the KYA signing proxy and verified by the demo shop. Interim report.
- **M3 (end of Apr 2027):** the observer suspends tasks automatically and delegation works. All attack examples are reproducible.
- **M4 (Jun 2027):** evaluation complete; final report, poster and live demo.

---

## 4. Methodology

### S0 – Platform setup

We deploy Agent Substrate and AX at one pinned commit (v0.3.0 or a later main commit), on a managed Kubernetes cluster (GKE, 3–5 worker nodes, education credits), plus a local kind cluster for development. The first experiment verifies three things:

- a task bound to a Gateway whose egress allowlist contains only the KYA signing proxy can reach nothing else (`GatewayReady` condition);
- Gateway changes behave as expected for running tasks;
- ax-server can be restricted by a NetworkPolicy so that only the admission proxy and ax-controller reach it.

If the Gateway restriction is unreliable, the prototype falls back to HTTP_PROXY/HTTPS_PROXY (cooperative mode), and every evaluation result states which mode was used.

### S1 – Specification

- **Passport:** a JWT signed with Ed25519 (EdDSA) containing:
  - `sub`: a SPIFFE-style ID, `spiffe://kya.dev/ns/<namespace>/task/<name>`, so it can later align with AX's planned SPIFFE identities;
  - `task`: image digest, model (from the AX Model resource), the AX gateway the task is bound to (`kya.task.gateway`), and a manifest hash covering the Task spec, its Workspaces and its Gateway;
  - `principal`: subject, identity provider, login time, group;
  - `visa`: hosts, actions, per-request and total limits, approval threshold;
  - `parent`, `status` URL and `cnf.jwk`, the public task key (RFC 7800).
- **Signature profile (RFC 9421, conforming to the Web Bot Auth draft [8]):**
  - covered components: `@method`, `@authority`, `@path`, `signature-agent`, `kya-passport` and, if a body is present, `content-digest` (RFC 9530);
  - parameters: `created`, `expires` (60 s after `created`), `keyid`, a random `nonce` and `tag="web-bot-auth"`;
  - `keyid` is the JWK SHA-256 thumbprint (RFC 7638; RFC 8037 for Ed25519) of the task key named in the passport's `cnf.jwk`;
  - the `Signature-Agent` header names the passport authority's origin. The PA serves a signed key directory at `/.well-known/http-message-signatures-directory` with the public keys of all active tasks, so any Web Bot Auth verifier can check who signed a request even if it does not understand passports. KYA verifiers additionally check the key against the passport.
- **AgentPolicy:** a YAML schema per group, including whether `spec.debug` may be used.
- **Threat model:** STRIDE analysis.

![Figure 7: Anatomy of an agent passport.](../figures/passport.png)

*Figure 7: Anatomy of an agent passport. Each quadrant corresponds to one question a relying party may ask.*

### S2 – Identity core

- **Login:** `kya login` runs the OAuth 2.0 device flow (RFC 8628) against the university SSO. The CLI is pointed at the admission proxy through `--server` or `$AX_SERVER`, and a thin wrapper attaches the ID token as gRPC metadata.
- **Admission proxy (Go):** implements the `ax.v1alpha1.AX` gRPC service using AX's generated types (`pkg/apis/v1alpha1`).
  - Read-only calls (`GetTask`, `ListTasks`, `WatchTask`) are forwarded after authentication and filtered so that principals see only their own tasks.
  - AX has no separate create RPC: `UpdateTask` creates or updates a task. For each `UpdateTask` the proxy runs:
    1. validate the token;
    2. select the policy by the principal's groups;
    3. distinguish a new task from an update (only the launching principal may update a task; an update that changes the manifest hash needs a new passport, and the old one is revoked);
    4. validate the image (pinned digest only), model, resources, debug flag and requested visa, and reject with a human-readable reason if needed;
    5. compute the manifest hash;
    6. request a passport;
    7. adjust the task: set `spec.image` to the KYA runner variant, set `spec.gateway` to `kya-egress-only`, and inject `HTTPS_PROXY` with a per-task 256-bit proxy credential, `KYA_PASSPORT_ID` and `KYA_EVENTS_URL`;
    8. forward to ax-server and write audit events.
  - Because a Task references its Gateway and Workspaces by name, the proxy also governs the other write calls: `UpdateGateway`/`DeleteGateway` and `UpdateModel`/`DeleteModel` are reserved for operators (platform administrators), and `UpdateWorkspace` is rejected while a running task references the workspace.
  - A NetworkPolicy ensures that ax-server accepts connections only from the admission proxy and ax-controller, so the proxy cannot be bypassed.
- **Passport authority (Go + PostgreSQL):**
  - The issuer key is published at `/.well-known/jwks.json` and rotated monthly.
  - A fresh Ed25519 key pair is created for each task. The private half goes to the KYA signing proxy over mTLS and lives only in its memory.
  - Revocation uses an API endpoint, a per-passport status URL, and a signed revocation list that verifiers poll every 30 s or receive by server-sent events. This keeps verification local.
  - For privacy, the PA can issue pairwise pseudonymous principal IDs to services that do not need the real identity.

### S3 – Enforcement

- **KYA runner:** written in Go by embedding the public `github.com/google/ax/runner` package (`runner.Run`). It keeps the AX runner contract unchanged:
  - `/usr/local/bin/ax-task-runner` as entrypoint;
  - `/healthz` and `/readyz` on port 80;
  - `AX_TASK_YAML`, `AX_WORKSPACES_YAML` and the metadata endpoints.

  Before starting the agent it installs the proxy CA certificate, sets the proxy variables, opens a local event endpoint whose events it forwards, batched, to the observer, and serves a read-only approval-status endpoint plus three MCP tools (`kya_my_permissions`, `kya_approval_status`, `kya_wait_for_approval`). Workspace preparation (Git clones, MCP and skill registries) also goes through the signing proxy. The runner never holds keys or passports.
- **AX Gateway + KYA signing proxy:**
  - The AX Gateway decides *where* a sandbox may connect: only to the KYA signing proxy.
  - The KYA signing proxy (a Go forward proxy) decides *what* leaves the cluster and *how* it is signed. It:
    - identifies the calling task by its per-task proxy credential (later: the task's AX-issued SPIFFE SVID);
    - terminates TLS with the cluster CA and opens a new TLS connection upstream (an explicit-proxy mode is offered for libraries that pin certificates);
    - checks the visa hosts, attaches the passport and signs the request;
    - meters requests, bytes and LLM tokens as spend signals.
  - As a stretch goal, the proxy injects model API keys so that no long-lived secret is needed in the sandbox.
- **Known costs of the AX integration:**
  - AX creates one actor template per distinct image and environment, so the per-task proxy credential in `spec.env` may reduce actor density; we measure this in the evaluation (S8, see 5.4).
  - `ax ssh` needs `spec.debug`, which policy reserves for operators.
  - AX's workspace setup agents need a model API key inside the container; until AX moves setup into a separate actor, this is the one secret in the sandbox that AgentPassport does not control.

![Figure 8: The KYA signing proxy.](../figures/signing.png)

*Figure 8: The KYA signing proxy turns a plain request from the agent into a signed, attributable request. The AX Gateway makes the proxy the only reachable destination; the agent never sees the key.*

### S4 – Relying-party side

- **Verifier bindings:** a Spring Boot starter (servlet filter plus a `@KyaAction(value, amount, currency)` annotation) and a Go net/http middleware.
- **Check order (cheapest first):**
  1. `tag="web-bot-auth"` and the required covered components, including `signature-agent` and `kya-passport`;
  2. `created` within ±60 s, `expires` not passed, and nonce not seen for this key in the last 5 min;
  3. content digest;
  4. passport JWS against the JWKS, `nbf`/`exp`/`aud`;
  5. revocation list no older than 60 s;
  6. `keyid` equals the thumbprint of `cnf.jwk`, and the request signature verifies with that key;
  7. host, action and amount limits;
  8. delegation chain.
- **Gradual adoption:** requests without KYA headers pass through by default; `kya.mode=required` accepts verified agents only.
- **Step-up approvals (agent-aware):**
  1. The verifier returns an RFC 9457 `approval-required` problem document with the request digest. The relying party never contacts the human.
  2. The signing proxy opens an approval in the console and passes the 403 on to the agent, enriched with `approval_id`, a local `status_url` and `expires_at`.
  3. The agent follows the status through the KYA runner (read-only, task-scoped) or the `kya_wait_for_approval` MCP tool, and can tell its own user what it is waiting for.
  4. The console shows the principal the verified facts (host, item, amount) separately from the agent's unverified justification. Only the principal, logged in with OIDC, can approve.
  5. If approved, the PA issues an approval grant bound to the exact request digest, valid for 5 minutes. The agent repeats the identical request, and the proxy attaches the grant in a `KYA-Approval` header. If rejected, the agent receives the principal's note and can adapt.
  6. Approval requests carry no trust penalty; only spamming them does. AX's planned approval policies would be platform-internal; our grants are what the relying party can verify.
- **Demo relying parties:** a shop (Spring Boot) and a payments API (Go).

![Figure 9: Decision procedure of the KYA verifier.](../figures/verifier_flow.png)

*Figure 9: Decision procedure of the KYA verifier. Red outcomes are hard failures; amber outcomes ask the principal to approve.*

![Figure 10: Agent-aware step-up approval.](../figures/stepup.png)

*Figure 10: Agent-aware step-up approval. The agent receives an explicit "approval required" answer, waits through a read-only status channel, and repeats the same request once the principal has approved. Only the principal, logged in to the console, can approve.*

### S5 – Trust observer

Python, FastAPI, scikit-learn, Redis Streams/NATS.

- **Input:** OpenTelemetry-compatible events from the signing proxy, runner and verifiers, plus AX lifecycle events from `WatchTask`.
- **Sliding-window features per task:**
  - request rate relative to the task's own baseline;
  - ratio of denied requests;
  - never-seen hosts and actions for this image;
  - spend velocity relative to the remaining budget;
  - a repetition index over the last k requests, which detects loops.
- **Score** (s₀ = 0.8):

  ```
  s_t = clip[0,1]( s_{t−1} + λ(s₀ − s_{t−1}) − Σᵢ pᵢ(t) − β·max(0, a_t − τ) )
  ```

  - pᵢ are rule penalties: egress denied 0.15, verifier 403 0.20, new action type 0.10, budget over 80% used 0.10.
  - a_t is the anomaly score of an isolation forest trained on benign runs of the same image, and τ is its tolerance.
  - λ is a slow recovery rate, so a single false alarm does not permanently cripple a task.
- **Actions:**
  - s ≥ 0.6: normal operation.
  - 0.4 ≤ s < 0.6: step-up approval for every state-changing request.
  - s < 0.4: call `SuspendTask` on AX, revoke the passport and notify the principal. The suspended task keeps its memory and workspace, so a false alarm is recoverable. Interactive inspection with `ax ssh` is only possible for tasks started with `spec.debug`.
- When AX ships native runner and gateway telemetry (on its roadmap), it can replace the KYA runner's own event reporting.

![Figure 11: Trust observer pipeline.](../figures/observer.png)

*Figure 11: Trust observer pipeline: signals from all components become per-task features, which are scored and mapped to actions.*

### S6 – Delegation and audit

- **Delegation:** a sub-task's `UpdateTask` also passes through the admission proxy, authenticated by the parent's proxy credential. The PA's delegate operation checks four rules:
  - the child's hosts and actions are a subset of the parent's;
  - every child limit is at most the parent's remaining budget, which is then reserved;
  - the delegation depth does not exceed `maxDelegationDepth`;
  - the child's expiry is not later than the parent's.

  The child is bound to the same `kya-egress-only` Gateway. Revoking a parent revokes all its descendants. As a research extension, we compare this PA-mediated design with offline Biscuit attenuation.
- **Audit ledger:** an append-only PostgreSQL table where each entry stores hₙ = SHA-256(hₙ₋₁ ∥ canonical(eₙ)). Every 5 minutes the PA signs and publishes a checkpoint, so even a database administrator cannot silently rewrite history.

![Figure 12: Delegation with attenuation.](../figures/delegation.png)

*Figure 12: Delegation with attenuation. Each sub-task receives only the permissions it needs; a sub-task that tries to use a permission of its parent is rejected.*

![Figure 13: Hash-chained audit entries.](../figures/audit_chain.png)

*Figure 13: Hash-chained audit entries. Each entry commits to all entries before it.*

### S7 – Console

React + TypeScript. Views:

- My agents: tasks, passport, visa, remaining budget and live trust score;
- Approvals: pending step-up requests with approve/reject;
- Incidents: denials and suspensions with their causes;
- Audit explorer: search the ledger and verify the hash chain in the browser;
- Operator views: policies, Gateway bindings, fleet statistics, revocation controls.

### S8 – Evaluation

- **Testbed:** the demo shop and payments API (both with the verifier) plus one service without the verifier as a control. Agents are both scripted (exact replay) and LLM-driven (shopping, trip planning, coding). At least 200 benign runs are used to train the anomaly model and measure false positives.
- **Attack suite:** 30+ scenarios in six families:
  - prompt injection;
  - scope escalation;
  - impersonation/replay;
  - delegation abuse;
  - resource abuse;
  - platform evasion, including attempts to widen the Gateway, bypass the admission proxy, or reach ax-server directly.
- **Ablation:** four configurations:
  1. **Baseline:** plain AX with a permissive gateway; agents use API keys directly;
  2. **Proxy only:** AX gateway restricted to the signing proxy; allowlist and metering, no signatures;
  3. **Proxy + verifier:** full KYA identity and authorization, no observer;
  4. **Full AgentPassport:** all components.

### Team roles and work division

| Member | Role | Steps owned | Responsibilities |
|---|---|---|---|
| Ömer Burak Taşkın | Platform & infrastructure | S0, S3 | Cluster, AX/Substrate install, Gateway and NetworkPolicy setup, KYA runner, signing proxy, Helm deployment |
| Alper Melkeli | Identity & cryptography | S1, S2, S6 | Passport spec, PA, admission proxy (UpdateTask/Gateway governance), signatures, delegation, audit ledger |
| Yiğit Mete Tuygun | Relying party & SDK | S4, S7 | Spring Boot/Go verifiers, demo shop and payments, step-up flow, console |
| Fikret Kerem Dinçer | Observer & evaluation | S5, S8 | Features, anomaly model, attack suite, benchmarks |

**Steps with more than one member:**

- **S1:** Alper writes the passport and signature specs. Burak contributes the Gateway and NetworkPolicy experiment results. Fikret leads the threat model.
- **S4 step-up flow:** Mete builds the verifier side and console UI. Alper builds approval-grant issuance in the PA. Burak adds problem-document handling in the signing proxy.
- **S8:** all members contribute. Each one writes the attack scenarios and benchmarks for their own layer; Fikret coordinates and runs the ablation. Interim and final reports are written jointly.

---

## 5. Outcome and Impact

### 5.1 Expected outcome (software product)

1. Open-source code for all components (admission proxy, passport authority, KYA runner, KYA signing proxy, trust observer, audit ledger, console), with Helm charts and a ready `kya-egress-only` Gateway manifest to install AgentPassport next to AX.
2. The AgentPassport specification (passport claims, signature profile, policy schema), written so that third parties can build compatible verifiers.
3. Verifier SDKs for Spring Boot and Go, published as packages. Protecting an existing endpoint takes under 20 lines of code and configuration:

   ```java
   @PostMapping("/api/orders")
   @KyaAction(value = "order.create", amount = "#body.amount", currency = "TRY")
   ResponseEntity<Order> create(@RequestBody OrderRequest body, KyaContext kya) { ... }
   ```

4. A reproducible attack suite and benchmark scripts.
5. Interim and final reports, a poster, and a recorded end-to-end demonstration.

### 5.2 Example screens and outputs

- *CLI, rejected launch:*
  ```
  PERMISSION_DENIED: task "shop-8b21" rejected by policy students-shopping:
    - image must be pinned by digest (got tag "latest")
    - action "payments.pay" is not allowed for group hacettepe-cs/design-team
  ```
- *CLI, rejected Gateway change:*
  ```
  PERMISSION_DENIED: UpdateGateway "kya-egress-only" is managed by AgentPassport
    and cannot be modified by principals
  ```
- *Console approval page (step-up):* verified facts "shop.example.com · USB-C hub, 7-port · 450 TRY", and separately the agent's unverified justification "best price per port among 14 results", with Approve / Reject and an optional note.
- *Console incident view:* "suspended: repetitive requests (412 identical POST /api/orders in 2 minutes)", with the full event trail.

![Figure 14: An indirect prompt-injection attack and how each AgentPassport component limits it.](../figures/injection.png)

*Figure 14: An indirect prompt-injection attack against the shopping agent and how each AgentPassport component limits it.*

![Figure 15: Trust score over time.](../figures/trust_score.png)

*Figure 15: Trust score over time. Normal activity keeps the score around 0.82. Two denied requests and a never-seen action type within 40 seconds push it below the suspend threshold, and the observer suspends the task.*

### 5.3 Demonstrated behaviour

- **Normal purchase:** the shop receives an order that proves which agent placed it, which code it ran, who is responsible, and that this person allowed exactly this kind of purchase.
- **Prompt injection:** a hidden instruction to send 9,000 TRY is blocked twice:
  - the request to the attacker's host never leaves the cluster, because the AX Gateway allows only the signing proxy and the proxy rejects the host;
  - the verifier rejects the out-of-visa action.

  The observer then suspends the agent within seconds. No money moves.
- **Looping agent:** stopped before it burns the model budget.
- **Stolen passport:** useless. A replayed request fails the timestamp and nonce checks, and a new request cannot be signed, because the task key exists only in the signing proxy.
- **Over-reaching sub-agent:** rejected by delegation attenuation.
- **Purchase above the threshold:** the agent learns it needs approval, waits, and either repeats the exact request with a grant or adapts after a rejection; it can never approve itself.

### 5.4 Measurable targets

| Metric | Target |
|---|---|
| In-scope attacks blocked (full configuration) | ≥ 95% |
| Added signing-proxy latency, p50 / p99 | < 10 ms / < 40 ms |
| Verification time with cached keys, p50 | < 2 ms |
| Signing-proxy throughput on one core | ≥ 2,000 signed req/s |
| Admission overhead per UpdateTask | < 150 ms |
| Task start-up time and actor density with vs. without per-task environment | reported |
| Detection rate of hijacked/looping tasks | ≥ 90% |
| False-positive suspensions | ≤ 2 per 100 benign runs |
| Time from triggering event to suspension | < 5 s |
| Changes needed in an existing agent | none |

### 5.5 Impact

- **For services receiving agent traffic** (shops, banks, SaaS): they can tell verified, accountable agents from anonymous bots and can enforce limits and human approval on risky actions. Today's common practice is to give agents long-lived API keys that work for anyone, for months, with full account power. AgentPassport replaces this with short-lived, scoped, revocable credentials.
- **For organisations running agent fleets** (universities, companies): every outbound action becomes attributable to an authenticated person. Damage from hijacked or buggy agents is bounded and stopped automatically, and a tamper-evident audit trail supports investigations and accountability.
- **For the field:** the project provides an open reference implementation of Know Your Agent tied to a real agent platform, which does not exist today, together with an empirical, layer-by-layer evaluation of how much each defence contributes.
- **Alignment with AX's direction:** passport subjects already use SPIFFE IDs, the observer consumes OpenTelemetry, and the KYA signing proxy is designed so it can later be offered as an implementation of AX's planned swappable-gateway interface. The work therefore complements AX's roadmap (identity, governance, telemetry) and could be contributed upstream.
- **Dissemination:** the code, specification and SDKs will be released as open source. Results will be presented in the final report, poster and live demo, and we plan to submit them to a student or workshop venue on AI agent security (to be chosen with our advisor).
