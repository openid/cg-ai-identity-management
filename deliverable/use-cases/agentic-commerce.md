# An agent buying from a merchant it has never used

Revision 3, 26 November 2026. Posted to the AIIM list on 26 November 2026: https://lists.openid.net/pipermail/openid-aiim/Week-of-Mon-20261123/000277.html

Contributors: Tejaswi Suresh (RankSaga Ltd), consumer side; Francesco Marinoni Moretto, merchant side; Andrii Akhtanin (Radiolaria), late approval, recovery and acceptance cases; Staford Titus S., the view afterwards.

Actors: the consumer and their shopping agent; the merchant and its checkout agent; the payment provider; the issuer of the consumer's mandate. The agent and merchant have no prior relationship.

## First actor: the consumer (Tejaswi Suresh)

**As a** consumer,

**I need** to let a shopping agent buy from a merchant I have never used, within limits I set in advance, without giving the agent my card, my password or my merchant account,

**so that** the merchant can tell which agent is calling and on whose behalf, the payment side can check that the purchase is inside what I authorised, and I get a record of what was bought, from whom and why, that I can dispute if something goes wrong.

**For example,** I ask an agent to "buy running shoes in size 10 for under £120 from a shop that delivers by Friday". The agent finds three shops through their published agent profiles (for example a UCP business profile at /.well-known/ucp). It signs its HTTP requests with HTTP Message Signatures under Web Bot Auth, so each shop can look up the agent operator's key directory. At checkout it presents a mandate that says "this person, up to £120, running shoes, until Friday". Shipping raises the total from £118 to £126, so I am asked to approve revision R1 at £126. While I answer, the merchant issues R2 with a different total or payee. My approval of R1 covers R1 only, unless R2 stays within a range I explicitly allowed. One shop rejects the request, and the agent needs to know whether the problem was its signature, its key or the mandate, so it can tell me.

## Second actor: the merchant the agent has never used (Francesco Marinoni Moretto)

**As a** merchant,

**I need** a buyer's agent that has never dealt with me to be able to check, before it pays, which legal entity stands behind my shop and that the checkout it is talking to speaks for that entity,

**so that** the agent can tell me apart from a look-alike shop, the payment side knows who it is paying, and the record of the purchase names the right business if the buyer disputes it later.

**For example,** two of the three shops the agent finds sell the same shoes at the same price. One is shop.example, run by a registered company; the other copies its catalogue under a similar domain. Both publish a UCP business profile with signing keys and sign their checkout responses, as UCP recommends. Before presenting the mandate, the agent needs to know that shop.example is operated by that company, and that the key signing the checkout belongs to it and not only to whoever controls the domain today. After the purchase, the buyer's record should hold which merchant entity and which checkout agent were involved, who was paid and in what role, and when the agent's authority ended.

## Observations

### From the agent side (Tejaswi)

* Operator identity is not principal identity. Web Bot Auth tells the merchant which operator signed the request (its key directory), not which person the agent acts for, nor which mandate applies. Nothing standard binds the HTTP signing key to the mandate holder. Verifiable Intent carries the person's delegation and the transaction details, but is not tied to the request or its signature either, so a merchant verifies separate things and has to trust they belong together.

* Mandate issuer trust anchors. Verifiable credential mandates are only as good as the merchant's list of trusted issuers. There is no standard way to discover, pin or rotate those issuers, and getting that configuration wrong fails silently rather than loudly.

* Sites express agent policy by user-agent string, not verified identity. In a scan of the Tranco top 10,000 (6,478 sites measured, November 2026), GPTBot is fully disallowed on 14.5% of sites with a robots.txt, while user-driven agents are disallowed far less (ChatGPT-User 6.2%, Claude-User 4.5%). Sites can only tell the two apart by a string anyone can send. Only 10 of the measured sites publish a Web Bot Auth key directory.

* An approval covers terms, not a checkout (Andrii). When the basket goes over the mandate, the person, not the agent, has to approve, and that approval is bound to the material terms it covers: amount, items, merchant entity, payee, validity, and any range the person explicitly allowed. A late approval cannot move to changed terms. Revoking it before payment stops the payment even if its stated expiry is still in the future. The purchase terms, permitted action and validity must be represented in the protected authorization inputs. Buyer, merchant and payment provider each keep their own current decision. Some protocols already bind an approval to one set of terms. In AP2, for example, the merchant must check that the checkout hash in the Checkout Mandate matches the checkout it was sent, and that the closed checkout meets every constraint of the open mandate; the Payment Mandate identifies the checkout by the same hash and names the payee [AP2-1, AP2-4]. That binding is an existing mechanism to verify. Revocation before payment and recovery are not covered by it, and are tested end to end in the acceptance cases below.

### From the merchant side (Francesco)

* Domain control is not legal identity. A UCP business profile at /.well-known/ucp ties the business's capabilities and signing keys to its origin. A self-issued merchant identity assertion (draft-anders-merchant-identity-assertions) proves control of the domain at signing time, not that the legal details are true; it points to registers such as GLEIF for that, and leaves the decision to the verifier.

* Workload identity needs a trust domain the verifier can find. If the merchant's checkout agent has a WIMSE or SPIFFE identity, a verifier with no prior relationship has to learn which trust domain speaks for shop.example and get its keys. SPIFFE federation expects the trust domain name, bundle endpoint and profile to be configured out of band, and warns against inferring the trust domain name from the endpoint's host. draft-schwenkschuster-wimse-trust-domain-discovery proposes a well-known endpoint that resolves a trust domain name to its trust bundle for exactly this open case; it proves who controls the DNS name, and leaves the decision whether to trust that domain, and for what, to the relying party.

* Agent names and accountable entities. draft-ihsanullah-dnsid binds an agent's name to the domain of the entity accountable for it, and leaves to consuming systems whether that entity is trustworthy or legally qualified.

* The cross-domain case is recognised as open. WIMSE AIMS issue #146 asks to name cross-domain access without prior federation as a gap.

* Nothing ties four things together for a stranger: the legal entity, the domain, the key or workload identity of the agent acting at checkout, and the payee the payment provider actually pays, with its role relative to the merchant entity. The payee is often not the legal entity behind the shop, for legitimate reasons (a marketplace acting as merchant of record, a payment facilitator settling for its sub-merchants, a platform collecting for the seller). So the binding should not require the payee to be the same entity. It should say who is paid and in which role, so that the agent and the person see it before approving, and the record shows it if the purchase is disputed. Each piece exists; the binding between them, verifiable by a counterparty seen for the first time, does not.

### From afterwards: the person, or their bank in a dispute (Staford and Andrii)

* An unknown outcome is not settled by fresh permission. The checkout response for attempt A is lost, so A's outcome is unknown. The agent obtains new authorization (a new token or a new approval) and tries again as attempt B. If both took effect, the person paid twice, and if nothing links B to A, no record shows it: each attempt looks authorized on its own. The record needs a purchase identity that stays the same on every attempt, separate attempt identities, and the authorization and outcome evidence for each attempt. A stays "unknown" until evidence about A itself resolves it, such as a refusal before dispatch or the provider's record of the charge. A verified refusal before dispatch settles the attempt it identifies, not every attempt at that purchase. Verified reconciliation, or deduplication by the merchant or provider, may support safe continuation; fresh permission alone does not. Preserving the uncertainty must not block unrelated work: a separate, independently authorized purchase can still complete. This links the agent's decision before trying again with the reconstruction afterwards by the person or their bank.

* Receipts are bound to a mandate, not to an attempt (Andrii). In AP2's autonomous mode, an agent must not present a further open Checkout or Payment Mandate until it has a rejection receipt for the previous one, and every Checkout or Payment Receipt carries a reference to the hash of one closed mandate [AP2-2, AP2-3, AP2-4]. That orders the presentations and ties each receipt to its mandate. It does not settle a different attempt's outcome. Where A and B use different closed mandates, B's receipt does not by itself establish A's outcome. If the same closed mandate is presented more than once, the reference alone does not tell those presentations apart. And an Error status alone does not establish that the payment was refused before dispatch.

* The merchant's own figure is not an independent one. After the purchase, the amount charged is usually known from the merchant's record. To check that the charge stayed within the mandate, the person or their bank needs the payment provider's figure for the same purchase, with who was actually paid and in what role relative to the merchant entity. Where only the merchant's figure exists, the honest result is "not established", not "within the mandate".

### Shared: why a request was refused, and what is known afterwards

* When a merchant rejects a signed agent request, today it usually returns a bare 401 or 403. The agent can't tell an expired signature from an unknown key, a missing directory or an authorization it doesn't hold, so it can't repair the request or explain the failure to its person. The same reason is what the buyer's record needs to show that the agent did not overstep. In AAuth, a resource token can say what the agent must still obtain; the open part is a refusal that happens before that, at the signature or key. I've proposed an RFC 9457 problem type for this in the IETF Web Bot Auth WG (draft-suresh-webbotauth-verification-outcome).

* Named outcomes make a missing check visible instead of letting it pass silently. Illustrative names, not a proposal for the exact vocabulary:
  * signature: invalid or unverified, with a reason such as expired or key not found
  * mandate: unknown mandate issuer; mandate expired or revoked; outside mandate scope; approval covers a different revision
  * principal: principal approval required
  * attempt: refused before dispatch; outcome unknown
  * afterwards: within the mandate; outside the mandate; not established

## Acceptance cases (Andrii, with Staford's rows)

These are stated in checkout terms and do not depend on any one implementation.

1. The original £120 mandate does not authorize £126.
2. A valid new approval permits the payment when the actual purchase remains within its scope and the required parties permit it.
3. A delayed approval does not cover changed terms outside its scope. Revocation before dispatch means the payment is not initiated, even before the approval's stated expiry.
4. If an earlier attempt was dispatched but its outcome is unknown, a new approval does not settle that outcome or erase the attempt's relationship to the purchase.
5. A separate, independently authorized purchase can still complete while the first outcome remains unknown (positive control).
6. Negative control for case 4. Attempt A for purchase P is dispatched, but its response is lost and its outcome remains unknown. A recovery path presents attempt B for the same intended payment under fresh approval; B receives a verified refusal before payment dispatch. The record must retain A as unknown and B as refused. B's refusal must not turn the purchase into "nothing was paid" or clear A's unresolved outcome. Mandate sequencing and receipt binding, as in AP2, support this case but do not satisfy it on their own: an Error receipt for B is not yet a verified refusal before dispatch, and it says nothing about A.
7. Where only the merchant's figure for the charge exists, the result afterwards is "not established".

## Standards gaps (headings from [standards-gaps.md](../standards-gaps.md))

* Identity: binding the agent operator's key to the principal and the mandate; binding the merchant's checkout agent to the legal entity behind the domain
* Delegated Authority and Authorization: mandate constraints (amount, category, merchant, expiry); step-up to the person; an approval bound to the terms it covers, including payee and role; a purchase identity that ties every attempt and re-authorization to one purchase, so that an unknown outcome holds until it is resolved; an identity for each attempt, since a reference to the mandate does not distinguish two presentations of it
* Cross-Domain Federation & Data Sharing: the agent, merchant, payment provider and issuer sit in four trust domains; verifying a counterparty with no prior federation; the independent figure comes from the payment provider, not the merchant
* Ecosystem Interoperability: discovering agent-facing merchant endpoints, mandate issuer trust anchors, and which assertions and keys a merchant domain publishes about itself; a shared vocabulary for refusals and outcomes
* Life Cycle Management: revocation before payment; keeping each attempt and its outcome in the record; reconciling the provider's figure, payee and payee role against the merchant's record and the mandate; keeping the buyer's record verifiable after the merchant rotates keys or the agent's authority ends

## Possible building blocks (pointers, not conclusions)

* AAuth Protocol (person identity and PS authorization modes, resource tokens), Budgets (the consumption record, and a request identifier that survives re-authorization), Supervision, R3, HTTP Signature Keys
* AuthZEN Access Request Approval Profile (current approval checks and scope matching, sections 10.1 and 12.3)
* AP2 (merchant checks of the checkout hash and the open mandate's constraints; Payment Mandate bound to the checkout hash and naming the payee; in autonomous mode, a rejection receipt before any further open mandate; Checkout and Payment Receipts bound to the hash of the closed mandate), with Verifiable Intent and Mastercard Agent Pay
* Web Bot Auth (draft-ietf-webbotauth-httpsig-protocol) and draft-suresh-webbotauth-verification-outcome
* OpenID Federation Well Known Bindings (in call for adoption in the A/B Connect WG)
* draft-anders-merchant-identity-assertions, draft-schwenkschuster-wimse-trust-domain-discovery, draft-ihsanullah-dnsid
* AMEP, draft-stafordtituss-agentic-mandate-evidence (evaluates afterwards whether each charge stayed within the mandate, with an operation identifier for retries; Staford is its author)

## AP2 references

* [AP2-1] Specification, Verification, Merchant: https://ap2-protocol.org/ap2/specification/#merchant
* [AP2-2] Specification, Autonomous (Human Not Present): https://ap2-protocol.org/ap2/specification/#autonomous-human-not-present
* [AP2-3] Checkout Receipt: https://ap2-protocol.org/ap2/checkout_mandate/#checkout-receipt
* [AP2-4] Payment Mandate schema and Payment Receipt: https://ap2-protocol.org/ap2/payment_mandate/#mandate-schema and https://ap2-protocol.org/ap2/payment_mandate/#payment-receipt
