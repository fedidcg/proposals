# Proposal Summary: idp-sp-storage API
The problem we met to solve is how browsers can help prevent hidden tracking without breaking federation protocols for Research and Education federations. Our hope is that this scenario could be more broadly implemented for other federated use cases.

The RP, the browser, the IdP, and the user all need to own a piece of the consent that suggests the user is aware of and accepts that an authentication flow is about to happen. If this is done by interrupting the actual protocol flow, certain scenarios (e.g., when the IdP must know who the RP is before responding with a response that will allow the user to progress) will fail.

The group did not focus on third-party cookies specifically; the focus was on how to enable browser-level consent without breaking the protocol, regardless of the primitives being used. This means that this proposal might solve not only for third-party cookies but also for future issues with link decoration (aka, navigation-based tracking) and redirects (aka, bounce tracking).

Research & Education have tools and services, particularly federation operators that provide governance to the federation system, that we considered leveraging as part of the way to move from a consent model that focuses solely on gaining the user’s consent for a transaction to a classification model that allows the browser to differentiate between a federated authentication flow and a tracking flow. Consent can still be included, but by classifying the action correctly, the user experience should be much simpler.

The scenario in this proposal assumes that the metadata provided by the Relying Party to the browser will allow the browser to both classify the flow and provide a common UX for further IdP discovery (aka, resolve the NASCAR problem).

For more background see [20230309 Background for proposals](https://docs.google.com/document/d/1L3O3fefitV6CxmoaG29JP9nYA9S9WAq2J2tJfIAR1yM/edit#)

We propose a javascript API for authorizing a set of mappings between an SP and an IdP in the browser.
The data that defines the SP and IdP in the API calls defined below (e.g. n.c.allowed.put) is either the ORIGIN of the SP/IdP or a signed data structure which includes one or more of the URLs which the IdP has configured as the authorized location(s) to which the authentication response may be returned for a specific SP. Note that current authentication protocols have a limited list of permitted URLs that may be used by an SP; the SP must assert URLs from that list.

The data that defines the IdP is, similarly, either the ORIGIN or a signed data structure which includes the URL the SP will use (via the appropriate protocol binding) to make its request, and a URL for UX information that is in the same ORIGIN as the IdP. The .well-known/ location cannot be used for this purpose as many IdPs for different organizations may be hosted at the same ORIGIN. By requiring this URL to be at the same ORIGIN as the authentication endpoint the SP cannot insert misleading data. In the absence of an SP-asserted UX endpoint the .well-known location could be used. In the absence of a .well-known endpoint the URL could be presented to the user.

By using a signed data structure, instead of simply specifying the ORIGIN, the caller can include additional information used to drive the UX–such as the human-friendly and/or localized name of the IdP/SP. The signed data structure is submitted, along with a proof of membership of the public key, in a recognized transparency log allowing the browser to validate the origin of the data.

Calls to the API will trigger browser-provided UX at certain stages to ensure that the user is aware that a mapping exists between the SP and the IdP. The browser will use this mapping – including the SP and IdP endpoints used in identity protocol flows – to allow interaction between these endpoints which would otherwise not be permitted. (That is: future bounce tracking and other cross domain exchanges that appear to be tracking data.)

#### Privacy threat model
There are 3 actors involved in the standard federated identity exchange:

1. The Identity Provider (IdP). The IdP has information about associated users and can upon request create an assertion (aka claim-set). Each claim-set is created specifically for transfer to a single service provider (cf below) in response to an authentication request.
2. The Service Provider (SP). The SP represents a service that wants to authenticate users and get access to attributes (claims) about the authenticated user.
3. The user-agent that mediates the protocol flow between the IdP and SP.
The following is a brief description of a typical authentication flow using the most common protocol bindings for SAML. OIDC is quite similar.

1. The user expresses an interest to authenticate at a service provider. This typically involves a user gesture such as clicking a button marked “Login”.
2. The SP creates a user flow, the purpose of which is to determine which IdP the user wants to use. This flow creates information at the user-agent about which IdP the user believes themselves to be associated with. Minimally this information is a public identifier of the IdP (eg a URI). NB that this is not the same thing as account information which typically involves information about the user and not only information about the IdP. In point of fact the information created in this step is equivalent to information about an affiliation between the user agent and the IdP.
3. The SP creates an authentication request and directs the user-agent to send the authentication requests as a URI parameter in a GET request to the IdP. The authentication request is sometimes encrypted to the IdP and may contain sensitive information about the authentication that is taking place. For instance when the user is involved in a financial transaction the authentication request may contain summary information for display by the IdP about the nature of the transaction. The information in the authentication request (if encrypted) must be protected from the user-agent and other AITM.
4. The IdP presents UX to the user prompting them to authenticate. After successful authentication the IdP creates a set of claims encrypted with the SP public key. The user-agent is directed to POST the encoded and encrypted claims to the SP. In this step the set of claims may contain information about the user which the user may not themselves have access to. By encrypting the set of claims to the SP the claims are protected against the user-agent and AITM.
5. The SP unpacks the claims and decrypts them with its private key. At this state the SP has access to the claims the IdP sent. These claims are not necessarily equivalent to account information in the traditional sense. In many cases the only claim provided is detailed information about the affiliation of the user and some organizations (eg that they are students at a certain university) but no further information may be provided. The information in the claims are not necessarily available to the user-agent at this stage either since they are POSTed to the ORIGIN.
In summary:

- User information is never made available to the user-agent unless by the SP after a successful authentication response has been received by the IDP
- The only information needed to start the flow is the users choice of IdP which does not involve any user information beyond the belief (by the user) that they have an association with the IdP
- Authenticated information about the user is controlled by the IdP and created specifically for a particular transaction with a particular SP
- The user-agent is never privy to any information exchanged between the SP and the IdP.
- Both the authentication request and response may contain sensitive information which must not be shared with the user-agent.
#### API calls
<code>n.c.allowed.isEmpty(<sp>) -> Boolean</code>
Returns true if there are no objects associated with .
This is called the unconfigured state for the .

  <code>n.c.allowed.put(<sp>, <idp>, [<ttl>]) -> Promise<IdP></code>

The browser prompts the user to allow the - mapping to be authorized. If the user agrees, the mapping between and is stored. Optionally this mapping expires after time-to-live . If the user authorizes the mapping the Promise resolves with the choice, allowing the calling page to invoke the login flow for the chosen . Alternatively, if the user denies the mapping the Promise resolves as an error which can be handled in an exception flow by the caller.

  <code>n.c.allowed.invoke(<sp>) -> Promise<IdP></code>

The invoke method displays a UX allowing the user to select one among the ones already associated with (from a previous n.c.allowed.put call). When the user has chosen, the Promise resolves with the object. The user should be given the choice to not invoke any of the existing objects and have the UX behave as if it is in the unconfigured state for . This is signaled by resolving the Promise as an error which can be handled in an exception flow by the caller.

    <code>n.c.allowed.get(<sp>) -> Promise<[IdP]></code>

The get method returns a list of the objects associated with the object. This API call will throw an exception through the Promise if called outside of a Chrome [fenced frame](https://developer.chrome.com/en/docs/privacy-sandbox/fenced-frame/) (or equivalent code-execution sandbox in other browser engines). This allows an SP-supplied (not browser-provided) UX (in a sandbox) to be presented, which is not visible to the SP, but allows cross-origin resources, etc.

      <code>n.c.allowed.deleteAll(<sp>)</code>

Delete all for this .

        <code>n.c.allowed.delete(<sp>, <idp>)</code>

Delete this particular - pair.

## What is the Expected RP (or IDP Library) Developer Experience?
An RP would either call a SAML or OIDC discovery service that calls the API or would call the API directly on the SP page.

## What is the Expected IDP Developer Experience?
This proposal does not impact any of the current identity protocols.

## What is the Expected User Experience?
Code executing on the page will call n.c.allowed.put(<sp>,<idp>) which would cause the user to be prompted to allow the <idp> to be authorized for use by the page associated with <sp>. The <idp> and <sp> are expected to be objects representing “metadata” for the two entities. The UX is expected to be similar to current credit card autofill behaviour. When the calling page wants to use the mapping it calls n.c.allowed.invoke(<sp>) which prompts the user to select among the existing stored <idp>-objects associated with the <sp> object.

The following pseudocode outlines the different API calls in use by the SP in the process of connecting the user to their IdP. The embedded STOP signals where the proposed new flow terminates and moves to existing protocol behavior.

1. SP wants to invoke SSO protocol
  a. SP knows the IdP it wants to use, so it calls n.c.allowed.put(<sp>, <idp>, [<ttl>]) to request consent for establishing that linkage and permission to invoke SSO.
    i. On error, go to b, otherwise proceed with SSO protocol. STOP
  b. SP does not know the IdP to use.
    i. SP calls n.c.allowed.isEmpty(<sp>) to check for existing relationships to reuse.
      If true, go to ii
      If false, SP may call n.c.allowed.invoke(<sp>) to attempt browser-mediated reuse of an IdP choice.
  c. If error, go to ii, otherwise proceed with SSO protocol. STOP
  d. SP invokes a protocol and community appropriate Discovery Service (either embedded or external) using existing discovery protocol.
  e. Discovery service pattern out of scope of this document, go to 2.
2. DS returns IdP choice to SP’s discovery endpoint
a. SP now knows the IdP it wants to use, so it calls n.c.allowed.put(<sp>, <idp>, [<ttl>]) to request consent for establishing that linkage and permission to invoke SSO
  i. On error (either IdP refuses or user rejects registering the IdP chosen) repeat discovery of a suitable IdP. Otherwise proceed with SSO protocol. STOP
![IdP storage API flow diagram](https://user-images.githubusercontent.com/111496976/224849248-b19b89ba-4aa4-4344-847b-5807c224ccea.png)

Regarding discovery, this narrative assumes discovery may be embedded within SP or shared, but is not formally part of the browser apart from the “invoke” option to select from an existing set of relationships (not select new ones). This would obviate the need for a shared discovery service to ever retain state/choices since that would be managed by the browser alone and reused via n.c.allowed.invoke.

In a typical case, a user would be forced through discovery once, pick their IdP and establish the connection for some TTL, then reuse it via the ç method until it expires; discovery would be bypassed entirely by the SP for that TTL period. Multiple IdPs works fine, allowing that additional trips through discovery and the n.c.allowed.put call (to obtain consent) would occur. The user would have to select “none of the above” after n.c.allowed.invoke is called to make that happen.

#### Open questions
- What UI “sprinkles” are meaningful to add to the UX when/if there is out of band knowledge about the trust status of the and/or .
- Represent some version of the “context” label in the API to enable groups of SPs to enable a mechanism for buckets of ’s to share their mapping.
- Would a user be able to map an unlimited number of IdP / SP connections?
- How does the browser know when to ask for user’s consent?
- Would this process mean that the user can potentially approve of a connection with a malicious SP, one that’s not trusted by their IdP? In what way does this system preserve the existing trust infrastructure?
#### Out of scope
Consuming federation trust elements (e.g. metadata or public keys) is explicitly out of scope for this proposal however it is possible to combine this proposal with additional signals based on out-of-band import of metadata.

#### Flows
1. Brand new computer / session, never used SP or IdP
2. Coming back to the SP and reuse a used IdP
3. Going to a brand new IdP
![](https://user-images.githubusercontent.com/111496976/224849343-b61afade-4301-48b2-82fe-ea46ac2392cf.png)

![](https://user-images.githubusercontent.com/111496976/224849479-3d40542c-ff75-49c1-a49d-31f450b29929.png)

![](https://user-images.githubusercontent.com/111496976/224849510-1f51e490-f4b4-412e-ad01-f98228905dac.png)

![](https://user-images.githubusercontent.com/111496976/224849549-f3794103-2ac8-4313-b610-6dff5639460d.png)

![](https://user-images.githubusercontent.com/111496976/224849571-65b8953e-9c46-44d2-90e9-1c34bcc70260.png)

![](https://user-images.githubusercontent.com/111496976/224849617-47e8a414-048c-4a58-ba86-70f76b4998ea.png)

![](https://user-images.githubusercontent.com/111496976/224849663-369d3a6e-9d3e-4136-8eed-abe903d0addf.png)

![](https://user-images.githubusercontent.com/111496976/224849726-0c650f53-c397-4040-b1f7-9a7c296e43a1.png)

![](https://user-images.githubusercontent.com/111496976/224849759-e0ccee3d-72e7-4c30-87d0-5751cfb57df3.png)

![](https://user-images.githubusercontent.com/111496976/224849776-26744198-5a73-4727-b9ea-42d8437bfdd7.png)

![](https://user-images.githubusercontent.com/111496976/224849800-3c28a73c-6b72-475f-9fbd-1432b9120ca5.png)

![](https://user-images.githubusercontent.com/111496976/224849850-51e38c2b-6efd-4e88-85f5-1742775de724.png)

![](https://user-images.githubusercontent.com/111496976/224849876-722c4e09-6a09-42cd-b35c-ccc8c88fa9cc.png)

An account management module for editing and removing - for federated accounts
![](https://user-images.githubusercontent.com/111496976/224849911-a287c6a6-4ebb-46b5-8a2f-0067af219d11.png)
