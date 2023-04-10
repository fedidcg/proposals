# Proposal Summary: Offloading Trust
The problem we met to solve is how browsers can help prevent hidden tracking without breaking federation protocols for Research and Education federations. Our hope is that this scenario could be more broadly implemented for other federated use cases.

The RP, the browser, the IdP, and the user all need to own a piece of the consent that suggests the user is aware of and accepts that an authentication flow is about to happen. If this is done by interrupting the actual protocol flow, certain scenarios (e.g., when the IdP must know who the RP is before responding with a response that will allow the user to progress) will fail.

The group did not focus on third-party cookies specifically; the focus was on how to enable browser-level consent without breaking the protocol, regardless of the primitives being used. This means that this proposal might solve not only for third-party cookies but also for future issues with link decoration (aka, navigation-based tracking) and redirects (aka, bounce tracking).

Research & Education have tools and services, particularly federation operators that provide governance to the federation system, that we considered leveraging as part of the way to move from a consent model that focuses solely on gaining the user’s consent to a transaction to a classification model that allows the browser to differentiate between a federated authentication flow and a tracking flow. Consent can still be included, but by classifying the action correctly, the user experience should be much simpler.

The scenario in this proposal assumes that metadata is provided by both the Relying Party and the Federation Operator to the browser; the browser will then compare the two lists (‘trust but verify’ the relationships) and will allow the browser to both classify the flow and provide a common UX for further IdP discovery (aka, resolve the NASCAR problem).

For more background see [20230309 Background for proposals](https://docs.google.com/document/d/1L3O3fefitV6CxmoaG29JP9nYA9S9WAq2J2tJfIAR1yM/edit#)

This proposal bridges the current management of trust by identity federations by expanding on the notion of a browser settings service, with a collection of servers that aggregates federation data (nightly or more often), and makes it available for consumption from the user agents.

This browser service depends on a governance process by which each browser provider decides which federations to support, configuring a process for consuming and distributing metadata for validation in the flows described below.

## Relevant Flows
Flows for discovery-initiated sign-in (where the IdP is initially unknown) and pairwise sign-in (where both IdP and SP are already known) have been developed. Common to each flow is the notion of the federation metadata service, a new component of the browser settings service. This service pushes trusted federation metadata to user-agents (browsers) where it can be utilised by the discovery-initiated and pairwise flows to both; aid in trust decisions the browser makes about the information it receives from either an SP or IdP, and or, to aid the user-experience by enhancing the user-interface presented to the user (e.g. including federation supplied UI elements, or improving organisation display names etc.).

### Browser Settings Service (common component for both flows)
1. Fed Bootstrapper bootstrap trust
*Browser service pulls in the public key for the federation*
  a. Based on governance/policy about allowed federations.
  b. Governance model is an agreement between the browser provider and the federation; different browsers have latitude to make different decisions about supporting different federations (i.e., Federation X is supported by Browser Y but not Browser Z)

2. Fed Fetcher (fetch and validate metadata)
*Browser service gets and validates metadata and UI (nightly or more frequently)*
  a. Need to also push to user aggregate (browser clients)

![Offloaded trust flow diagram](https://user-images.githubusercontent.com/111496976/226541832-7821c86a-eb36-4414-b67e-234113db2513.png)

### Discovery-initiated flow
1. Federated Login at SP (payload)
*Browser client navigates to SP; SP asks the user if they want to log in. User affirms, and:*
  a. SP sends payload to browser (JSON or other format)
    i. The payload (FedPayload) contains: federation(s) the browser belongs to, and optionally an allowed list of IdPs the SP wants to restrict authentication to

2. Validation
*Browser reconciles FedPayload request against metadata*
  a. Is this SP in the Federation they seeded in FedPayload?
  b. Are these IdPs in the Federation they seeded in FedPayload?
  c. Form UI for IdP selection with a filtered list (dropping anything that doesn’t validate (and optionally wasn’t in the set of IdPs the SP wants displayed/allowed))

3. The browser displays IdP discovery dialog, the user chooses an IdP, and consents to <IdP,SP> pairing
  a. Browser returns IdP options to user
    i. The UI for this can be driven by the federation operator
    ii. Using federation metadata to improve the UX
  b. User selects IdP
  c. Browser records user choice as pair (IdP<->SP tuple) and resolves the selected IdP as a JS promise.

4. SP is able to initiate SSO session with metadata-trusted and user-selected IdP
  a. Browser excludes pairing from cross-site tracking protection mechanisms
    i. Based on IdP and SP origins, or something else.
  b. Redirect to IdP (SAML), GET (something else), maintain protocol agnosticism (“All HTTP verbs allowed between this pair of entities”)

What is the expected user experience with Discovery initiated-flow?

1. User requests to log in at the SP
2. User is presented with a browser modal with one-to-many Federation selectors which may be filtered in the reconciliation between SP’s requested Federations/IdPs and the browser’s validation against metadata
  a. User selects an IdP to log in
  b. Browser steps out of flow and IdP and SP interact as normal

![Discovery-initiated flow diagram](https://user-images.githubusercontent.com/111496976/227154365-ad169999-a537-490b-a1d2-93e57c7f224f.png)

^end of discovery initiated-flow^

Pairwise flow
1. Federated Login with known IdP and SP pairing (no discovery)
  a. User opens URL in the browser and requests a resource
  b. The browser receives only one tuple/pairing of federation + SP + single IdP (no discovery needed)

2. Validation and user consent
  a. Browser validates the IdP and SP pairing against metadata for the chosen federation i.e. are they trusted by that federation
  b. The browser asks the user for approval/consent
    i. using federation metadata to improve the UX
  c. [If approved] Browser records user choice as pair (IdP<->SP tuple) and resolves the selected IdP as a JS promise.
  i. Authentication is initiated as normal
  d. [If not approved] perform exception logic

What is the expected user experience with pairwise-flow?

1. User requests a resource, for example:
  a. An IdP-initiated portal, or
  b. An SP that only uses one IdP
2. User is presented with IdP/SP pair and given the opportunity to approve/consent; Browser steps out of the flow and allows IdP and SP to interact as normal

![Pairwise flow diagram](https://user-images.githubusercontent.com/111496976/227154948-f38c3ce9-4062-4021-b375-3c2080e67c99.png)

^end of pairwise flow^

Having completed either of the flows above, the browser has sufficient confidence in the validity of the transaction to allow authentication flows to happen as normal.

## Open questions
1. What happens when validating against metadata that frequently rolls over? (if we have a fairly static root cert, assuming other certs could rotate more frequently)

2. What happens if the user does not select or approve IdP offering?

3. Scenario - Federation or IdP is compromised and now modal includes corrupted content (inappropriate thumbnail image etc)
  a. How does the browser communicate that content in the discovery modal comes not from the browser, but the Federation
  b. Discovery has attribution (e.g., Seamless Access)

4. How does the browser signal to the user that a discovery model with vetted IdPs is official vs something recreated/spoofed by the RP? (scenario - RP scrapes Seamless Access, invokes window.open and presents an unverified version of discovery - a browser should signal difference between these for user protection)

### Implementation considerations
1. What is the Expected RP (or IDP Library) Developer Experience?
  a. How is this Accomplished in Specification?
2. What is the Expected IDP Developer Experience?
  a. How is this Accomplished in Specification?
