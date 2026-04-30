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

A key design goal of RPP's authorization model is fine-grained access control. A registrar MUST be able to operate multiple user accounts within the registry, each carrying a distinct set of permissions appropriate to the user's role (e.g., read-only reporting accounts, accounts limited to a specific set of operations, or fully privileged administrative accounts). This allows registrars to implement the principle of least privilege within their own organizations without requiring separate registry-level registrar accounts. The registry's authorization server MUST support registrar self-service management of these user accounts, enabling registrars to create, modify, and revoke user credentials and their associated permission scopes without requiring manual intervention by the registry operator.

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

# Authorization Modes

RPP MAY use OAuth 2.0 [@!RFC6749] as its authorization framework. OAuth 2.0 is an authorization protocol and does not perform authentication itself; authentication of the client or end-user is handled by the authorization server before a token is issued. RPP acts as an OAuth 2.0 Resource Server and MUST validate every incoming request against a Bearer token presented in the `Authorization` header. The registry MAY operate its own authorization server or MAY delegate to an external authorization server. Access control decisions are derived exclusively from verified claims in the presented JWT access token.

Access tokens MUST be JWTs conforming to the JWT Profile for OAuth 2.0 Access Tokens [@!RFC9068]. Tokens MUST be signed using asymmetric cryptography; symmetric signing algorithms (e.g., HS256) MUST NOT be used for tokens issued by external authorization servers. Short-lived tokens are RECOMMENDED and token caching and refresh strategies MUST follow the best practices defined in [@!RFC8725]. Sensitive claims in the JWT payload MUST be encrypted if the token is persisted to storage.

RPP operates as a Policy Enforcement Point (PEP). The authorization server that issues a token is identified by the `iss` claim; the RPP server MUST validate the token's signature against the issuing authorization server's public key, fetched and cached via OAuth 2.0 Authorization Server Metadata [@!RFC8414].

RPP supports two registry authorization modes:

- **Internal mode**: the registry operates its own authorization server and issues tokens directly to clients.

  `Client → Registry Authorization Server → JWT → RPP API`

- **External mode**: the registry trusts one or more external authorization servers and validates tokens they issue. This mode is also required for federated object transfers, where tokens are issued by a registrar's own authorization server.

  `Client → External Authorization Server → JWT → RPP API`

In both modes authorization is enforced identically. The `aud` claim MUST identify the RPP server as the intended audience; the RPP server MUST reject tokens where its own identifier is absent from `aud`.

# Scopes

OAuth scopes are used for enforcing access control when accessing RPP resources. The server MUST define a set of scopes that can be requested by clients when obtaining access tokens. The scopes MUST be based on the principle of least privilege, allowing clients to request only the permissions they need to perform their intended operations. The server MUST also define the mapping between scopes and the specific resources and operations that they grant access to.

When a client requests an access token from the authorization server, it MUST include the desired scopes in the scope parameter of the token request. The authorization server MUST validate the requested scopes against the client's registered permissions and issue an access token with the appropriate scopes if the request is valid. The server MUST enforce access control based on the scopes included in the access token, allowing or denying access to resources based on the client's authorized scopes. The server MUST also implement appropriate error handling for cases where a client attempts to access a resource without the necessary scopes, returning an appropriate HTTP status code and error message.

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
  - **Primary - Rich Authorization Requests (RAR) [@!RFC9396]**: The gaining registrar MUST include an `authorization_details` parameter in the authorization request containing an object of type `rpp_transfer` with `object_type` and `object_identifier` fields (see Table 3). This is the RECOMMENDED approach when the losing registrar's authorization server supports RAR.
  - **Fallback - Transfer Scope**: When the losing registrar's authorization server does not support RAR, a static transfer scope of the form `<object>:transfer` MUST be used (e.g., `domain:transfer`). The specific object identifier MUST be carried in the `rpp_object_id` claim in the issued JWT (see (#rpp-specific-claims)). This scope MUST be presented to the registrant for approval, and the authorization server MUST include the `rpp_object_id` claim in the token.
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

OAuth 2.0 Rich Authorization Requests [@!RFC9396] extends the standard OAuth 2.0 authorization request with an `authorization_details` parameter that carries a structured JSON object describing precisely what the client is requesting authorization for. Unlike scopes, which are coarse-grained string tokens, `authorization_details` allows the request to include typed, fine-grained authorization data, such as the specific object being transferred. The  authorization server can present to the user in a meaningful consent screen.

In RPP, RAR is used for interactive federated object transfers to convey the specific domain name or contact handle being transferred. The gaining registrar includes an `authorization_details` object of type `rpp_transfer` in the authorization request to the losing registrar's authorization server. The registrant then sees exactly which object they are consenting to transfer. The authorization server MUST echo the `authorization_details` object back as a claim in the issued JWT, giving the registry verifiable, tamper-proof evidence of what was authorized and for which object.

The `type` field MUST be set to `rpp_transfer`. Table (#tbl-rar) lists the RAR fields defined for RPP.

| Field | Type | Requirement | Description |
| ----- | ---- | ----------- | ----------- |
| `type` | String | REQUIRED | MUST be `rpp_transfer`. |
| `object_type` | String | REQUIRED | The RPP object type being transferred. MUST be one of `domain` or `contact`. |
| `object_identifier` | String | REQUIRED | The fully qualified name or handle of the specific object to be transferred (e.g., `foo.example` or `CID-12345`). |
Table: RPP Transfer Authorization, RAR `authorization_details` object (Primary Method, [@!RFC9396])
{#tbl-rar}

Example RAR `authorization_details` value for a domain transfer:

```json
[{
  "type": "rpp_transfer",
  "object_type": "domain",
  "object_identifier": "foo.example"
}]
```

The losing registrar's authorization server MUST echo the `authorization_details` back as a claim in the issued JWT. The registry MUST validate the `authorization_details` claim in the token and MUST verify that `object_type` and `object_identifier` match the object being transferred.

When the losing registrar's authorization server does not support RAR ([@!RFC9396]), the specific object being transferred MUST be conveyed via the `rpp_object_id` claim rather than encoded in the scope string:

- To authorize a domain transfer, the specific domain name MUST be present in the `rpp_object_id` claim (e.g., `foo.example`). 
- To authorize a contact transfer, the specific contact identifier MUST be present in the `rpp_object_id` claim (e.g., `REG-12345`). 

# Claims

RPP access tokens MUST conform to the JWT Profile for OAuth 2.0 Access Tokens defined in [@!RFC9068]. This profile establishes a standardized set of claims that RPP uses to make authorization decisions consistently across implementations and identity providers.

## Required Claims

Table (#tbl-oauth-claims) lists the claims defined in [@!RFC9068], these MUST be present in every RPP access token:

| Claim | Type | Description |
| ----- | ---- | ----------- |
| `iss` | String (URI) | Identifies the issuing authorization server. The RPP server MUST validate this against its set of trusted issuers. |
| `sub` | String | The subject of the token. For machine-to-machine flows this MUST be the client identifier. For interactive flows this MUST be the end-user identifier of the authenticated end-user at the issuing authorization server. The end-user MAY be a registrar employee operating the registrar's management system, or a registrar customer (registrant) directly interacting with the registrar's client application, for example when initiating an object transfer. |
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
| `rpp_transfer_authinfo` | OPTIONAL | String | The transfer authorization information used in Fallback flow object transfer. When present, the value MUST be equivalent to the value that would otherwise be sent in the `RPP-Authorization` header, using the same `<method> <authorization information>` format. The RPP server MUST treat this claim as equivalent to the `RPP-Authorization` header; if both are present in the same request, the claim value MUST take precedence. This claim MUST only be present in access tokens used for transfer operations. |
| `rpp_object_id` | OPTIONAL | String | The identifier of the specific object being transferred. MUST be present in access tokens for interactive federated transfers when the fallback dynamic scope method is used (`transfer:domain` or `transfer:contact`) and the `authorization_details` claim ([@!RFC9396]) is not present. The value MUST be the fully qualified name or handle of the object (e.g., `foo.example` or `CID-12345`). The RPP server MUST validate that this value matches the object being transferred. This claim MUST NOT be present when `authorization_details` is present. |
| `authorization_details` | OPTIONAL | Array of Objects | Rich authorization details as defined by [@!RFC9396]. MUST be present in access tokens for interactive federated transfers when the RAR method is used. Each object in the array MUST have a `type` field; for RPP transfer tokens the `type` MUST be `rpp_transfer` and the object MUST include `object_type` and `object_identifier` fields (see (#tbl-rar)). The RPP server MUST validate that `object_type` and `object_identifier` match the object being transferred. When this claim is present, the `rpp_object_id` claim MUST NOT be present. |
Table: RPP Specific Access Token Claims
{#tbl-rpp-claims}

The combination of the `sub` claim and the `rpp_registrar_id` claim provides a complete, two-dimensional identity for every RPP request: `sub` identifies the individual principal (registrar employee, automated process, or registrar customer) that initiated the request, while `rpp_registrar_id` identifies the registrar organization as a whole within the registry's domain.

The identity of the `sub` depends on both the flow type and the identity domain of the principal:

- In **machine-to-machine** flows, `sub` is the registrar's OAuth 2.0 client identifier, representing an automated system acting on behalf of the registrar. The token is issued by the **registry's authorization server**, which maintains the registrar's client credentials.
- In **interactive flows initiated by registrar staff**, `sub` is the identifier of the registrar employee. Registrar employees are managed as users in the **registry's authorization server**. The token is therefore also issued by the registry's authorization server, and the `sub` value is the employee's account identifier within that server. The `iss` claim will identify the registry's authorization server.
- In **interactive flows initiated by a registrant customer**, the situation is different. Registrant customers are not managed in the registry's authorization server, they are maintained in the **registrar's own authorization server** (the registrar acts as the authorization server for its customers). The token is therefore issued by the registrar's authorization server, and the `iss` claim will identify the registrar's authorization server as the issuer. The registry MUST have a pre-established trust relationship with the registrar's authorization server to accept and validate such tokens. In this case, the `sub` value MUST be the registrant's identifier as it exists in the registry database. The registrar MUST use this registry-assigned id, not any registrar-internal customer identifier, as the `sub` value. This ensures the registry can unambiguously correlate the token's subject to an existing provisioned contact object. This enables verification of ownership and consent for operations. The registry MUST reject tokens where the `sub` value does not match a known contact handle associated with the object being acted upon.

Together, these claims allow the RPP server to:

- **Enforce access control** at both the organizational level (is this registrar authorized to manage this object?) and the individual level (has this principal been granted the necessary permissions?).
- **Attribute actions to individuals** for audit trail purposes, enabling the registry to record not just which registrar performed an operation, but which employee or customer initiated it.
- **Support delegation models**, where a registrar may grant different employees different scopes (e.g., a junior employee may hold only `domain:read` scope while a senior employee holds `domain:create` and `domain:update`).
- **Verify registrant consent** in interactive transfer flows, where a `sub` containing the registrant's handle provides the registry with verifiable evidence that the object owner explicitly authorized the transfer.
- **Facilitate incident response**, allowing the registry to correlate suspicious activity back to a specific principal rather than only to a registrar organization.

The RPP server SHOULD log both the `sub` and `rpp_registrar_id` claims for every request in its audit log. The `sub` value is only meaningful within the namespace of the issuing authorization server identified by the `iss` claim. The RPP server MUST NOT compare `sub` values across different issuers, except when the `sub` is a registry contact handle, in which case the registry MAY validate it against its own database regardless of issuer.

Extensions and profiles MAY define additional claims. All additional claims MUST use a URI or a collision-resistant name as the claim name to prevent conflicts with registered claims.

## Claim Validation

The RPP server MUST validate all required claims in accordance with [@!RFC9068] and [@!RFC8725]. Specifically:

- The `iss` claim MUST identify a trusted authorization server whose public keys are known to the RPP server, either through static configuration or dynamic discovery (e.g., OAuth 2.0 Authorization Server Metadata [@!RFC8414]).
- The `aud` claim MUST be validated to confirm the token is intended for this RPP server.
- The `exp` claim MUST be checked and expired tokens MUST be rejected with HTTP 401 Unauthorized.
- The JWT signature MUST be verified using asymmetric cryptography (e.g., RS256 or ES256). Symmetric algorithms (e.g., HS256) MUST NOT be used for tokens issued by external authorization servers.
- If the `scope` claim is absent or does not contain the scope required for the requested operation, the RPP server MUST return HTTP 403 Forbidden.

# Managing Trust and Keys

The RPP server MUST maintain a trust store of authorized issuers and their associated public keys for validating JWT signatures. RPP MUST support registrars being able to register their authorization servers and public keys with the registry operator to establish trust for interactive federated transfers.

**TODO** Create additional endpoints for managing trusted issuers and their keys, or specify a manual process for registrars to submit this information to the registry operator.

# Flows

RPP defines two authorization flows: machine-to-machine flows and interactive flows. Machine-to-machine flows are designed for use in automated systems where no user interaction is required. Interactive flows are designed for use in scenarios where end-user interaction is required, such as when a registrant initiates a transfer request using a registrar's web interface.

# Machine to Machine

The machine-to-machine (M2M) flow is used for automated, unattended RPP requests such as domain provisioning, renewal, or host management sent by a registrar's backend systems to the registry. No end-user interaction is involved. The OAuth 2.0 Client Credentials grant [@!RFC6749, Section 4.4] MUST be used for this flow.

In this flow the registrar's system authenticates directly with the registry's authorization server using a pre-registered `client_id` and `client_secret` (or a private key JWT for higher assurance). The authorization server issues a short-lived access token scoped to the requested RPP operations. The registrar's system then includes this token in the `Authorization` header of each RPP request sent to the registry.

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

1. The registrar's backend system sends a token request to the registry's authorization server, providing its `client_id`, `client_secret` (or a signed client assertion), and the requested RPP scopes (e.g., `domain:create`).
2. The authorization server validates the client credentials and issues a signed, short-lived JWT access token containing the granted scopes, `sub` (set to `client_id`), and `rpp_registrar_id`.
3. The registrar's system sends the RPP request to the registry's RPP server, including the access token in the HTTP `Authorization` header as a Bearer token.
4. The RPP server validates the JWT entirely locally, using the without contacting the authorization server. It verifies the token's signature using the authorization server's public key (previously fetched and cached via OAuth 2.0 Authorization Server Metadata [@!RFC8414] and the referenced JWKS [@!RFC7517] endpoint), checks the standard claims (`iss`, `aud`, `exp`), and confirms that the `scope` claim includes the scope required for the requested operation.
5. If validation succeeds, the RPP server processes the request and returns the RPP response.

It is RECOMMENDED that access tokens be short-lived (e.g., minutes to hours) and that the registrar's system obtain a new token before the current token expires rather than waiting for a 401 response. Token caching and refresh strategies SHOULD follow the best practices in [@!RFC8725].

## Interactive

The interactive flow is used for RPP requests that require end-user interaction. The OAuth 2.0 Authorization Code grant [@!RFC6749, Section 4.1] MUST be used for this flow. Two distinct types of end-users may initiate interactive RPP requests, and they are managed in different authorization servers:

- **Registrar employees** are managed as users in the **registry's authorization server**. A registrar employee uses the registrar's client application to perform operations on behalf of the registrar, such as updating domain records or managing contacts. The registry's authorization server authenticates the employee and issues an access token. The `sub` claim will contain the employee's account identifier in the registry's authorization server, and the `iss` claim will identify the registry's authorization server.

- **Registrant customers** are managed in the **registrar's own authorization server**. A registrant customer interacts directly with the registrar's client application, for example to authorize a domain transfer. The registrar's authorization server authenticates the customer and issues an access token. The `sub` claim MUST be set to the registrant's contact handle as registered in the registry database. The `iss` claim will identify the registrar's authorization server. The registry MUST have a pre-established trust relationship with the registrar's authorization server to accept tokens it issues.

### Registrar Employee Flow

In this flow, the employee authenticates with the registry's authorization server. The registry acts as both the authorization server and the RPP resource server.

```ascii
  Registrar          Registry             Registry
  Employee +         Auth Server          RPP Server
  Client App             |                    |
     |                   |                    |
     | 1. Login /        |                    |
     |  auth request     |                    |
     +------------------>|                    |
     |                   |                    |
     | 2. Access token   |                    |
     |  (sub=employee_id |                    |
     |   iss=registry)   |                    |
     |<------------------|                    |
     |                   |                    |
     | 3. RPP request    |                    |
     |  (Bearer token)   |                    |
     +--------------------------------------->|
     |                   |                    |
     |                   |  4. Validate JWT   |
     |                   |  (local, cached    |
     |                   |   registry public  |
     |                   |   key)             |
     |                   |                    |
     | 5. RPP response   |                    |
     |<---------------------------------------|
     |                   |                    |
```
Figure: Interactive Flow - Registrar Employee (Registry Auth Server)

### Registrant Customer Flow

In this flow, the registrant authenticates with the registrar's own authorization server. The registry must trust the registrar's authorization server and validate the `sub` against its contact database.

```ascii
  Registrant         Registrar            Registry
  Customer +         Auth Server          RPP Server
  Client App             |                    |
     |                   |                    |
     | 1. Login /        |                    |
     |  auth request     |                    |
     +------------------>|                    |
     |                   |                    |
     | 2. Access token   |                    |
     |  (sub=contact     |                    |
     |   handle,         |                    |
     |   iss=registrar)  |                    |
     |<------------------|                    |
     |                   |                    |
     | 3. RPP request    |                    |
     |  (Bearer token)   |                    |
     +--------------------------------------->|
     |                   |                    |
     |                   |  4. Validate JWT   |
     |                   |  (local, cached    |
     |                   |   registrar public |
     |                   |   key via          |
     |                   |   federation)      |
     |                   |                    |
     |                   |  5. Verify sub     |
     |                   |  matches contact   |
     |                   |  handle in         |
     |                   |  registry DB       |
     |                   |                    |
     | 6. RPP response   |                    |
     |<---------------------------------------|
     |                   |                    |
```
Figure: Interactive Flow - Registrant Customer (Registrar Auth Server)

The steps for the registrant customer flow are as follows:

1. The registrant authenticates with the registrar's authorization server through the registrar's client application, using the Authorization Code grant.
2. The registrar's authorization server issues a signed JWT access token. The `sub` claim MUST be set to the registrant's contact handle as it exists in the registry database. The `iss` claim identifies the registrar's authorization server. The `rpp_registrar_id` claim MUST be present, identifying the sponsoring registrar.
3. The registrar's client application sends the RPP request to the registry's RPP server, including the access token as a Bearer token.
4. The RPP server validates the JWT locally using the registrar's authorization server public key, previously obtained via federation metadata. No live call to the registrar's authorization server is needed.
5. The RPP server verifies that the `sub` value matches a known contact handle in the registry database and that the contact is associated with the object being acted upon.
6. If all validation succeeds, the RPP server processes the request and returns the response.

## Enforcing Interactive Authentication for High-Risk Operations

The machine-to-machine (M2M) Client Credentials flow is appropriate for routine automated provisioning, but certain high-risk operations, such as object transfer or modification of authorisation information, MUST require a token demonstrating that a human principal explicitly authenticated and authorized the action. Without enforcement, a registrar could use M2M tokens for all operations, eliminating individual accountability.

The following mechanisms MAY be used by the registry to enforce interactive authentication for designated operations:

**`sub` MUST identify a human principal**: The registry MAY require that for high-risk operations, the `sub` claim MUST contain the identifier of an authenticated human principal, either a registrar employee or a registrant customer, and MUST NOT equal the `client_id`. In M2M tokens issued via the Client Credentials grant, `sub` is always set to `client_id`, representing an automated system rather than a human. By mandating `sub` ≠ `client_id` for designated operations, the registry ensures those operations can only be performed with a token issued on behalf of a real, identified individual. The RPP server MUST reject requests for these operations when `sub` equals `client_id`.

**Scope restriction by grant type**: The registry's authorization server SHOULD be configured to refuse issuing certain high-risk scopes (e.g., `domain:transfer`) to the Client Credentials grant type. Only the Authorization Code grant (interactive) MAY be permitted to obtain these scopes. This prevents a registrar from obtaining the necessary scope for a high-risk operation through unattended M2M authentication.

The set of operations for which interactive authentication is required is a matter of registry policy and MUST be discoverable using the RPP discovery mechanism.

## Federation Trust Model

The RPP federation model uses the registry as the central trust anchor, operating as a hub-and-spoke topology. Registrars establish a trust relationship with the registry during accreditation; they do not need to establish direct trust relationships with each other. This allows any two registrars to participate in a federated transfer without any prior bilateral arrangement.

**Registry as trust anchor.** As part of registrar onboarding, each registrar that operates its own authorization server (i.e., maintains registrant accounts) MUST register its authorization server metadata with the registry. This includes at minimum:

- The authorization server's JWKS endpoint URI or the public key material itself, used by the registry to validate tokens issued by that authorization server.

The registry stores this metadata as part of the registrar's profile and makes it available via the discovery mechanism.

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
