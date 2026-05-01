%%%
title = "OAuth 2.0 for RESTful Provisioning Protocol (RPP)"
abbrev = "OAuth 2.0 for RPP"
area = "Internet"
workgroup = "Network Working Group"
submissiontype = "IETF"
keyword = [""]
TocDepth = 4
date = 2026-04-30

[seriesInfo]
name = "Internet-Draft"
value = "draft-wullink-rpp-oauth2-00"
stream = "IETF"
status = "standard"

[[author]]
initials="M."
surname="Wullink"
fullname="Maarten Wullink"
abbrev = ""
organization = "SIDN Labs"
  [author.address]
  email = "maarten.wullink@sidn.nl"
  uri = "https://sidn.nl/"

[[author]]
initials="P."
surname="Kowalik"
fullname="Pawel Kowalik"
abbrev = ""
organization = "DENIC"
  [author.address]
  email = "pawel.kowalik@denic.de"
  uri = "https://denic.de/"

%%%

.# Abstract

This document describes how OAuth 2.0 [@!RFC6749] can be used to secure RESTful Provisioning Protocol (RPP) API requests described in [@!I-D.wullink-rpp-core].

{mainmatter}

# Introduction

A key design goal of RPP's authorization model is fine-grained access control. A registrar MUST be able to operate multiple user accounts within the registry, each carrying a distinct set of permissions appropriate to the user's role (e.g., read-only reporting accounts, accounts limited to a specific set of operations, or fully privileged administrative accounts). This allows registrars to implement the principle of least privilege within their own organizations without requiring separate registry-level registrar accounts. The registry's AS MUST support registrar self-service management of these user accounts, enabling registrars to create, modify, and revoke user credentials and their associated permission scopes without requiring manual intervention by the registry operator.

Due to the stateless nature of RPP, the client MUST include authorization credentials in each HTTP request. RPP uses OAuth 2.0 [@!RFC6749] for delegated authorization via Bearer tokens. Basic authentication [@!RFC7617] SHOULD NOT be used. The server MUST validate the Bearer token on each request and reject any request with an invalid or expired token with an appropriate HTTP status code.

# Terminology

In this document the following terminology is used.

REST - Representational State Transfer ([@!REST]). An architectural style.

RESTful - A RESTful web service is a web service or API implemented using HTTP and the principles of [@!REST].

EPP RFCs - This is a reference to the EPP version 1.0 specifications [@!RFC5730], [@!RFC5731], [@!RFC5732] and [@!RFC5733].

RESTful Provisioning Protocol or RPP - The protocol described in this document.

URL - A Uniform Resource Locator as defined in [@!RFC3986].

Resource - An object having a type, data, and possible relationship to other resources, identified by a URL.

RPP client - An HTTP user agent performing an RPP request

RPP server - An HTTP server responsible for processing requests and returning results in any supported media type.

JWT - JSON Web Token as defined in [@!RFC7519].

# Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT","SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [@!RFC2119].

In examples, indentation and white space in examples are provided only to illustrate element relationships and are not REQUIRED features of the protocol.

All example requests assume a RPP server using HTTP version 2 is listening on the standard HTTPS port on host rpp.example. An authorization token has been provided by an out of band process and MUST be used by the client to authenticate each request.

<!--  QUESTION: Don we want to use OIDC or is plain OAuth 2.0 using profile from rfc9068 sufficient? (sub, client_id and scope) claims -->

# Authorization

RPP MAY use OAuth 2.0 [@!RFC6749] as its authorization framework. OAuth 2.0 is an authorization protocol and does not perform authentication itself; authentication of the client or end-user is handled by the AS before a token is issued. RPP acts as an OAuth 2.0 Resource Server and MUST validate every incoming request against a Bearer token presented in the `Authorization` header. The registry MAY operate its own Authorization Server (AS) or MAY delegate to an external AS. Access control decisions are derived exclusively from verified claims in the presented JWT access token.

Access tokens MUST be JWTs conforming to the JWT Profile for OAuth 2.0 Access Tokens [@!RFC9068]. Tokens MUST be signed using asymmetric cryptography; symmetric signing algorithms (e.g., HS256) MUST NOT be used for tokens issued by external ASs. Short-lived tokens are RECOMMENDED and token caching and refresh strategies MUST follow the best practices defined in [@!RFC8725]. Sensitive claims in the JWT payload MUST be encrypted if the token is persisted to storage.

RPP operates as a Policy Enforcement Point (PEP). The AS that issues a token is identified by the `iss` claim; the RPP server MUST validate the token's signature against the issuing AS's public key, fetched and cached via OAuth 2.0 AS Metadata [@!RFC8414].

In both modes authorization is enforced identically. The `aud` claim MUST identify the RPP server as the intended audience; the RPP server MUST reject tokens where its own identifier is absent from `aud`.

RPP MUST support the `Client Credentials Grant` grant type described in [@!RFC6749, Section 4.4] to allow registrar client systems to obtain access tokens.

# Scopes

OAuth scopes are used for enforcing access control when accessing RPP resources. The server MUST define a set of scopes that can be requested by clients when obtaining access tokens. The scopes MUST be based on the principle of least privilege, allowing clients to request only the permissions they need to perform their intended operations. The server MUST also define the mapping between scopes and the specific resources and operations that they grant access to.

When a client requests an access token from the AS, it MUST include the desired scopes in the scope parameter of the token request. The AS MUST validate the requested scopes against the client's registered permissions and issue an access token with the appropriate scopes if the request is valid. The server MUST enforce access control based on the scopes included in the access token, allowing or denying access to resources based on the client's authorized scopes. The server MUST also implement appropriate error handling for cases where a client attempts to access a resource without the necessary scopes, returning an appropriate HTTP status code and error message.

RPP scopes are based on the objects, processes and operations defined in RESTful Provisioning Protocol (RPP) data objects in [@!I-D.kowalik-rpp-data-objects]. Each scope corresponds to a specific set of permissions for accessing and manipulating RPP resources.

## Scope Derivation Rules

RPP scopes are derived systematically from the data object types and operation categories defined in [@!I-D.kowalik-rpp-data-objects]. The derivation rules are as follows:

- The scope identifier MUST use the format `<object>:<access-level>`, where `<object>` is the lowercase stable identifier of the data object and `<access-level>` is one of the access levels defined below.
- The `create` access level grants permission to perform the Create operation on the corresponding data object.
- The `read` access level grants permission to perform the Read operation on the corresponding data object.
- The `update` access level grants permission to perform the Update operation on the corresponding data object.
- The `renew` access level grants permission to perform the Renew operation on the corresponding data object.
- The `restore` access level grants permission to perform the Restore operation on the corresponding data object.
- The `delete` access level grants permission to perform the Delete operation on the corresponding data object.
- The `transfer` access level grants permission to perform all Transfer operations on the corresponding data object.
- The `list` access level grants permission to perform List operations on the corresponding data object collections.
- For interactive OAuth 2.0 federated transfers, authorization MUST convey the specific object being transferred so the registrant can give informed, object-scoped consent. Two mechanisms are defined in order of preference:
  - **Primary - Rich Authorization Requests (RAR) [@!RFC9396]**: The gaining registrar MUST include an `authorization_details` parameter in the authorization request containing an object of type `rpp_transfer` with `object_type` and `object_identifier` fields (see Table 3). This is the RECOMMENDED approach when the losing registrar's AS supports RAR.
  - **Fallback - Transfer Scope**: When the losing registrar's AS does not support RAR, a static transfer scope of the form `<object>:transfer` MUST be used (e.g., `domain:transfer`). The specific object identifier MUST be carried in the `rpp_object_id` claim in the issued JWT (see (#rpp-specific-claims)). This scope MUST be presented to the registrant for approval, and the AS MUST include the `rpp_object_id` claim in the token.
- Extensions and additional data objects registered in the IANA RPP Data Object Registry [@!I-D.kowalik-rpp-data-objects] SHOULD define their own scopes following the same derivation rules, using the registered stable identifier of the extension object as the `<object>` component.

## Scope Registry

Table (#tbl-scopes) defines the RPP scopes derived from the data objects specified in [@!I-D.kowalik-rpp-data-objects].

| Scope | Data Object | Operations Granted |
| ----- | ----------- | ------------------ |
| `domain:create` | Domain Name | Create |
| `domain:read` | Domain Name | Read |
| `domain:update` | Domain Name | Update |
| `domain:renew` | Domain Name | Renew |
| `domain:restore` | Domain Name | Restore |
| `domain:delete` | Domain Name | Delete |
| `domain:transfer` | Domain Name | Create, Approve, Reject, Cancel, Query |
| `domain:list` | Domain Name | List domain collection |
| `contact:create` | Contact | Create |
| `contact:read` | Contact | Read |
| `contact:update` | Contact | Update, Restore |
| `contact:delete` | Contact | Delete |
| `contact:transfer` | Contact | Create, Approve, Reject, Cancel, Query |
| `contact:list` | Contact | List contact collection |
| `host:create` | Host | Create |
| `host:read` | Host | Read |
| `host:update` | Host | Update, Restore |
| `host:delete` | Host | Delete |
| `host:list` | Host | List host collection |
Table: RPP OAuth 2.0 Scopes
{#tbl-scopes}

**TODO:** add more scopes such as for listing collections, process statussen, power/admin scope? etc.

# Rich Authorization Requests (RAR)

**TODO:** make this RAR section more generic so that is can be used for multiple operations, not just transfer. For example delegation management could also benefit from RAR to convey the specific domain whose delegation data is being managed.

OAuth 2.0 Rich Authorization Requests [@!RFC9396] extends the standard OAuth 2.0 authorization request with an `authorization_details` parameter that carries a structured JSON object describing precisely what the client is requesting authorization for. Unlike scopes, which are coarse-grained string tokens, `authorization_details` allows the request to include typed, fine-grained authorization data, such as the specific object being transferred. The  AS can present to the user in a meaningful consent screen.

Table (#tbl-rar) lists the RAR fields defined for RPP.

| Field | Type | Requirement | Description |
| ----- | ---- | ----------- | ----------- |
| `type` | String | REQUIRED | The type of RPP operation being authorized. |
| `object_type` | String | REQUIRED | The RPP object type being transferred. |
| `object_identifier` | String | REQUIRED | The unique identifier of the specific object the operation applies to (e.g., `foo.example` or `CID-12345`). |
Table: RPP Transfer Authorization, RAR `authorization_details` object (Primary Method, [@!RFC9396])
{#tbl-rar}

Example RAR `authorization_details` value for a domain transfer:

```json
[{
  "type": "transfer",
  "object_type": "domain",
  "object_identifier": "foo.example"
}]
```

The authorizing registrar's AS MUST echo the `authorization_details` back as a claim in the issued JWT. The registry MUST validate the `authorization_details` claim in the token and MUST verify that `object_type` and `object_identifier` match the object being transferred.

When the authorizing registrar's AS does not support RAR ([@!RFC9396]), the RAR attributes MUST be conveyed using individual claims;

Table (#tbl-rar) lists the RAR fields as claims defined for RPP.

| Claim | RAR Field | Type | Requirement | Description |
| ----- | --------- | ---- | ----------- | ----------- |
| `rpp_op_type` | `type` | String | REQUIRED | The type of RPP operation being authorized. |
| `rpp_object_type` | `object_type` | String | REQUIRED | The RPP object type being transferred. |
| `rpp_object_id` | `object_identifier` | String | REQUIRED | The unique identifier of the specific object the operation applies to (e.g., `foo.example` or `CID-12345`). |
Table: RPP RAR Attributes as JWT Claims (Fallback Method)
{#tbl-rar-claims}

# Claims

RPP access tokens MUST conform to the JWT Profile for OAuth 2.0 Access Tokens defined in [@!RFC9068]. This profile establishes a standardized set of claims that RPP uses to make authorization decisions consistently across implementations and identity providers.

## Required Claims

Table (#tbl-oauth-claims) lists the claims defined in [@!RFC9068], these MUST be present in every RPP access token:

| Claim | Type | Description |
| ----- | ---- | ----------- |
| `iss` | String (URI) | Identifies the issuing AS. The RPP server MUST validate this against its set of trusted issuers. |
| `sub` | String | The subject of the token. For machine-to-machine flows this MUST be the client identifier. For interactive flows this MUST be the end-user identifier of the authenticated end-user at the issuing AS. The end-user MAY be a registrar employee operating the registrar's management system, or a registrar customer (registrant) directly interacting with the registrar's client application, for example when initiating an object transfer. |
| `aud` | String or Array | Identifies the intended audience. MUST include the RPP server's resource identifier. The RPP server MUST reject tokens where its own identifier is not present in this claim. |
| `exp` | Numeric date | Expiry time. The RPP server MUST reject tokens that have expired. |
| `iat` | Numeric date | Time at which the token was issued. |
| `jti` | String | Unique identifier for the token, used to prevent token replay attacks. |
| `client_id` | String | The OAuth 2.0 client identifier of the gaining registrar or RPP client application. |
| `scope` | String | Space-separated list of granted scopes (see (#scopes)). The RPP server MUST enforce access control based on the scopes present in this claim. |
Table: OAuth 2.0 Access Token Claims for RPP ([@!RFC9068])
{#tbl-oauth-claims}

## RPP-Specific Claims

In addition to the standard [@!RFC9068] claims, table (#tbl-rpp-claims) lists the RPP-specific claims that are defined to enable fine-grained authorization decisions. Required claims MUST be present in every RPP access token. Optional claims SHOULD be included when applicable to the deployment or request context.

| Claim | Requirement | Type | Description |
| ----- | ----------- | ---- | ----------- |
| `rpp_registrar_id` | REQUIRED | String | The identifier of the registrar on whose behalf the request is made. The RPP server MUST validate that this identifier matches a known and authorized registrar. This claim MUST be present in all access tokens used for RPP requests. |
| `rpp_reseller_id` | OPTIONAL | String | The identifier of the reseller acting through the registrar's client application. This claim SHOULD be included when the request originates from a reseller operating under the registrar's account. The RPP server MAY use this claim for access control, auditing, and attribution purposes. The value is interpreted within the namespace of the registrar identified by `rpp_registrar_id`. |
| `authorization_details` | OPTIONAL | Array of Objects | Rich authorization details as defined by [@!RFC9396]. MUST be present in access tokens for interactive federated transfers when the RAR method is used. Each object in the array MUST have a `type` field; for RPP transfer tokens the `type` MUST be `rpp_transfer` and the object MUST include `object_type` and `object_identifier` fields (see (#tbl-rar)). The RPP server MUST validate that `object_type` and `object_identifier` match the object being transferred. When this claim is present, the `rpp_object_id` claim MUST NOT be present. |
| `rpp_object_id` | OPTIONAL | String | The identifier of the specific object being transferred. MUST be present in access tokens for interactive federated transfers when the fallback dynamic scope method is used (`transfer:domain` or `transfer:contact`) and the `authorization_details` claim ([@!RFC9396]) is not present. The value MUST be the fully qualified name or handle of the object (e.g., `foo.example` or `CID-12345`). The RPP server MUST validate that this value matches the object being transferred. This claim MUST NOT be present when `authorization_details` is present. |
| `rpp_op_type` | OPTIONAL | String | The type of RPP operation being authorized. Used in the fallback method (see (#tbl-rar-claims)) when the AS does not support RAR. MUST NOT be present when `authorization_details` is present. |
| `rpp_object_type` | OPTIONAL | String | The RPP object type the operation applies to (e.g., `domain` or `contact`). Used in the fallback method (see (#tbl-rar-claims)) when the AS does not support RAR. MUST NOT be present when `authorization_details` is present. |
Table: RPP Specific Access Token Claims
{#tbl-rpp-claims}

The combination of the `sub` claim and the `rpp_registrar_id` claim provides a complete, two-dimensional identity for every RPP request: `sub` identifies the individual principal (registrar employee, automated process, or registrar customer) that initiated the request, while `rpp_registrar_id` identifies the registrar organization as a whole within the registry's domain.

The identity of the `sub` depends on both the flow type and the identity domain of the principal:

- In **machine-to-machine** flows, `sub` is the registrar's OAuth 2.0 client identifier, representing an automated system acting on behalf of the registrar. The token is issued by the **registry's AS**, which maintains the registrar's client credentials.
- In **interactive flows initiated by registrar staff**, `sub` is the identifier of the registrar employee. Registrar employees are managed as users in the **registry's AS**. The token is therefore also issued by the registry's AS, and the `sub` value is the employee's account identifier within that server. The `iss` claim will identify the registry's AS.
- In **interactive flows initiated by a registrant customer**, the situation is different. Registrant customers are not managed in the registry's AS, they are maintained in the **registrar's own AS** (the registrar acts as the AS for its customers). The token is therefore issued by the registrar's AS, and the `iss` claim will identify the registrar's AS as the issuer. The registry MUST have a pre-established trust relationship with the registrar's AS to accept and validate such tokens. In this case, the `sub` value MUST be the registrant's identifier as it exists in the registry database. The registrar MUST use this registry-assigned id, not any registrar-internal customer identifier, as the `sub` value. This ensures the registry can unambiguously correlate the token's subject to an existing provisioned contact object. This enables verification of ownership and consent for operations. The registry MUST reject tokens where the `sub` value does not match a known contact handle associated with the object being acted upon.

Together, these claims allow the RPP server to:

- **Enforce access control** at both the organizational level (is this registrar authorized to manage this object?) and the individual level (has this principal been granted the necessary permissions?).
- **Attribute actions to individuals** for audit trail purposes, enabling the registry to record not just which registrar performed an operation, but which employee or customer initiated it.
- **Support delegation models**, where a registrar may grant different employees different scopes (e.g., a junior employee may hold only `domain:read` scope while a senior employee holds `domain:create` and `domain:update`).
- **Verify registrant consent** in interactive transfer flows, where a `sub` containing the registrant's handle provides the registry with verifiable evidence that the object owner explicitly authorized the transfer.
- **Facilitate incident response**, allowing the registry to correlate suspicious activity back to a specific principal rather than only to a registrar organization.

The RPP server SHOULD log both the `sub` and `rpp_registrar_id` claims for every request in its audit log. The `sub` value is only meaningful within the namespace of the issuing AS identified by the `iss` claim. The RPP server MUST NOT compare `sub` values across different issuers, except when the `sub` is a registry contact handle, in which case the registry MAY validate it against its own database regardless of issuer.

Extensions and profiles MAY define additional claims. All additional claims MUST use a URI or a collision-resistant name as the claim name to prevent conflicts with registered claims.

## Claim Validation

The RPP server MUST validate all required claims in accordance with [@!RFC9068] and [@!RFC8725]. Specifically:

- The `iss` claim MUST identify a trusted AS whose public keys are known to the RPP server, either through static configuration or dynamic discovery (e.g., OAuth 2.0 AS Metadata [@!RFC8414]).
- The `aud` claim MUST be validated to confirm the token is intended for this RPP server.
- The `exp` claim MUST be checked and expired tokens MUST be rejected with HTTP 401 Unauthorized.
- The JWT signature MUST be verified using asymmetric cryptography (e.g., RS256 or ES256). Symmetric algorithms (e.g., HS256) MUST NOT be used for tokens issued by external ASs.
- If the `scope` claim is absent or does not contain the scope required for the requested operation, the RPP server MUST return HTTP 403 Forbidden.

# Data Objects

The RPP Data Object Catalog described in [@!I-D.kowalik-rpp-data-objects] is extended to include new objects required for using OAuth 2.0 as a framework for authorization in RPP.

- *Client Object*: A registrar MUST register at least one OAuth 2.0 client to interact with the RPP server. The Client Object MUST include the following attributes:
  - Name: Unique (in registrar namespace) name of application.
  - Description: A brief description of the application's purpose.
  - Redirect URL: The URL to which the authorization server will redirect the user after granting or denying access.
  - Client Id: The OAuth 2.0 client identifier issued to the client during the registration process.
  - Client Secret: The OAuth 2.0 client secret issued to the client during the registration process.
- TODO

<!-- TODO: add text about security considerations related to the Client Secret -->

# Managing Trust

For more advanced use cases, enabled by OAuth 2.0, such as interactive federated transfers it is necessary for the RPP server to validate tokens issued by external ASs operated by registrars. This requires the RPP server to establish trust with these external ASs. The RPP server MUST maintain a trust store of authorized issuers and their associated public keys for validating access tokens. RPP MUST allow for registrars to register and maintain their ASs and public key information.

**TODO** Create additional endpoints for managing trusted issuers and their keys, or specify a manual process for registrars to submit this information to the registry operator.

# Flows

RPP defines two authorization flows: machine-to-machine flows and interactive flows. Machine-to-machine flows are designed for use in automated systems where no user interaction is required. Interactive flows are designed for use in scenarios where end-user interaction is required, such as when a registrant employee needs to interact with the registry system.
For these interactive flows, the OAuth 2.0 Authorization Code grant [@!RFC6749, Section 4.1] MUST be used.

## Machine to Machine

The machine-to-machine (M2M) flow is used for automated, unattended RPP requests such as domain provisioning, renewal, or host management sent by a registrar's backend systems to the registry. No end-user interaction is involved. The OAuth 2.0 Client Credentials grant [@!RFC6749, Section 4.4] MUST be used for this flow.

In this flow the registrar's system authenticates directly with the registry's AS using a pre-registered `client_id` and `client_secret`. The AS issues a short-lived access token scoped to the requested RPP operations. The registrar's system then includes this token in the `Authorization` header of each RPP request sent to the registry.

The `sub` claim in the resulting token MUST be set to the `client_id` of the registrar's system. The `rpp_registrar_id` claim MUST also be present, identifying the registrar organization within the registry's namespace.

```ascii
  Registrar                  Registry             Registry
  Backend                    Auth Server          RPP Server
     |                           |                    |
     | 1. Token request          |                    |
     |  (client_id +             |                    |
     |   client_secret           |                    |
     |   + scope)                |                    |
     +-------------------------->|                    |
     |                           |                    |
     | 2. Access token (JWT)     |                    |
     |<--------------------------|                    |
     |                           |                    |
     | 3. RPP request            |                    |
     |  (Authorization:          |                    |
     |   Bearer <token>)         |                    |
     +----------------------------------------------->|
     |                           |                    |
     |                           |  4. Validate JWT   |
     |                           |  (verify signature |
     |                           |   using cached     |
     |                           |   public key,      |
     |                           |   check claims     |
     |                           |   and scopes)      |
     |                           |                    |
     | 5. RPP response           |                    |
     |<-----------------------------------------------|
     |                           |                    |
```
Figure: OAuth 2.0 Client Credentials Flow for Registrar-to-Registry Requests

The steps in the diagram are as follows:

1. The registrar's backend system sends a token request to the registry's AS, providing its `client_id`, `client_secret` (or a signed client assertion), and the requested RPP scopes (e.g., `domain:create`).
2. The AS validates the client credentials and issues a signed, short-lived JWT access token containing the granted scopes, `sub` (set to `client_id`), and `rpp_registrar_id`.
3. The registrar's system sends the RPP request to the registry's RPP server, including the access token in the HTTP `Authorization` header as a Bearer token.
4. The RPP server validates the JWT entirely locally, using the without contacting the AS. It verifies the token's signature using the AS's public key (previously fetched and cached via OAuth 2.0 AS Metadata [@!RFC8414] and the referenced JWKS [@!RFC7517] endpoint), checks the standard claims (`iss`, `aud`, `exp`), and confirms that the `scope` claim includes the scope required for the requested operation.
5. If validation succeeds, the RPP server processes the request and returns the RPP response.

It is RECOMMENDED that access tokens be short-lived (e.g., minutes to hours) and that the registrar's system obtain a new token before the current token expires rather than waiting for a 401 response. Token caching and refresh strategies SHOULD follow the best practices in [@!RFC8725].

### High-Risk Operations

The machine-to-machine (M2M) Client Credentials flow is appropriate for routine automated provisioning, but certain high-risk operations, e.g. object transfer or modification of authorisation information, MUST require proof that an authorized human principal explicitly approved the action. Without enforcement, a registrar could use M2M tokens for all operations, eliminating individual accountability. The operations for which interactive authentication is required are a matter of registry policy.

The following mechanisms MAY be used by the registry to enforce interactive authentication for designated operations:

**`sub` MUST identify a human principal**: The registry MAY require that for high-risk operations, the `sub` claim MUST contain the identifier of an authenticated human principal and MUST NOT equal the `client_id`. In M2M tokens issued via the Client Credentials grant, `sub` is always set to `client_id`, representing an automated system rather than a human. By mandating `sub` != `client_id` for designated operations, the registry ensures those operations can only be performed with a token issued on behalf of a real, identified individual. The RPP server MUST reject requests for these operations when `sub` equals `client_id`.

**Scope restriction by grant type**: The registry's AS SHOULD be configured to refuse issuing certain high-risk scopes to the Client Credentials grant type. Only the Authorization Code grant (interactive) MAY be permitted to obtain these scopes. This prevents a registrar from obtaining the necessary scope for a high-risk operation through unattended M2M authentication.

The set of operations for which interactive authentication is required is a matter of registry policy and MUST be discoverable using the RPP discovery mechanism.

## Interactive

The interactive flow is used for RPP requests that require end-user interaction. The OAuth 2.0 Authorization Code grant [@!RFC6749, Section 4.1] MUST be used for this flow. **Registrar employees** are managed as users in the **registry's AS**. A registrar employee uses the registrar's client application to perform operations on behalf of the registrar, such as updating domain records or managing contacts. The registry's AS authenticates the employee and issues an access token. The `sub` claim will contain the employee's account identifier in the registry's AS, and the `iss` claim will identify the registry's AS.

This will enable a registrar to implement fine-grained access control for its employees by assigning different scopes to different employee accounts in the registry's AS. For example, a junior employee may be granted only `domain:read` scope, while a senior employee may be granted `domain:create` and `domain:update` scopes. This is an optional use case, and a registrar may choose to use the M2M flow for all operations if it does not require individual employee accountability or if it deems this to be too complex to manage.

In this flow, the employee authenticates with the registry's AS. The registry acts as both the AS and the RPP resource server.

```ascii
  Registrar        Registrar          Registry             Registry
  Employee         Client App         Auth Server          RPP Server
  (Browser)            |                  |                    |
     | 1. Login /      |                  |                    |
     |  auth request   |                  |                    |
     +---------------->|                  |                    |
     |                 |                  |                    |
     |                 | 2. Forward auth  |                    |
     |                 |  request         |                    |
     |                 +----------------->|                    |
     |                 |                  |                    |
     |                 | 3. Access token  |                    |
     |                 |  (sub=employee_id|                    |
     |                 |   iss=registry)  |                    |
     |                 |<-----------------|                    |
     |                 |                  |                    |
     | 4. Access token |                  |                    |
     |<----------------|                  |                    |
     |                 |                  |                    |
     | 5. RPP request  |                  |                    |
     +---------------->|                  |                    |
     |                 |                  |                    |
     |                 | 6. RPP request   |                    |
     |                 |  (Bearer token)  |                    |
     |                 +-------------------------------------->|
     |                 |                  |                    |
     |                 |                  |  7. Validate JWT   |
     |                 |                  |  (local, cached    |
     |                 |                  |   registry public  |
     |                 |                  |   key)             |
     |                 |                  |                    |
     |                 | 8. RPP response  |                    |
     |                 |<--------------------------------------|
     |                 |                  |                    |
     | 9. RPP response |                  |                    |
     |<----------------|                  |                    |
     |                 |                  |                    |
```
Figure: Interactive Flow - Registrar Employee

The steps in the diagram are as follows:

1. The registrar employee initiates a login or authorization request using the registrar's client application.
2. The client application forwards the authorization request to the registry's AS using the OAuth 2.0 Authorization Code grant.
3. The registry's AS authenticates the employee and issues a signed, short-lived JWT access token containing the granted scopes, `sub` (set to the employee's account identifier), and `rpp_registrar_id`. The `iss` claim identifies the registry's AS.
4. The client application returns the access token to the registrar employee.
5. The registrar employee submits an RPP request via the client application.
6. The client application forwards the RPP request to the registry's RPP server, including the access token in the HTTP `Authorization` header as a Bearer token.
7. The RPP server validates the JWT locally using the cached registry public key (fetched via OAuth 2.0 AS Metadata [@!RFC8414] and the referenced JWKS [@!RFC7517] endpoint). It verifies the signature, checks the standard claims (`iss`, `aud`, `exp`), and confirms the `scope` claim includes the scope required for the requested operation.
8. The RPP server processes the request and returns the RPP response to the client application.
9. The client application returns the RPP response to the registrar employee.


# IANA Considerations

TODO

# Internationalization Considerations

TODO

# Security Considerations

TODO

# Change History

## Version 00

- Created initial draft with core concepts and flows.

{backmatter}

{numbered="false"}
# Acknowledgements

TODO

<reference anchor="REST" target="http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm">
  <front>
    <title>Architectural Styles and the Design of Network-based Software Architectures</title>
    <author initials="R." surname="Fielding" fullname="Roy Fielding">
      <organization/>
    </author>
    <date year="2000"/>
  </front>
</reference>
