---
title: "OAuth 2.0 RAR Metadata and Error Remediation"
abbrev: "OAuth 2.0 RAR Metadata and Error Remediation"
category: std

docname: draft-zehavi-oauth-rar-metadata-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - RAR
 - Step-up
 - oauth
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "yaron-zehavi/oauth-rich-authorization-requests-metadata"
  latest: "https://yaron-zehavi.github.io/oauth-rich-authorization-requests-metadata/draft-zehavi-oauth-rar-metadata.html"

author:
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com

normative:
  RFC3986:
  RFC6750:
  RFC7662:
  RFC8414:
  RFC9068:
  RFC9101:
  RFC9126:
  RFC9396:
  RFC9470:
  RFC9728:
  IANA.oauth-parameters:
  JSON.Schema:
    title: "JSON Schema: A Media Type for Describing JSON Documents"
    target: https://json-schema.org/draft/2020-12/json-schema-core
    date: June 16, 2022
    author:
      - ins: A. Wright, Ed.
      - ins: H. Andrews, Ed.
      - ins: B. Hutton, Ed.
      - ins: G. Dennis

--- abstract

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} standardizes the exchange and processing of authorization details but does not define metadata for describing authorization details types.

In addition, no interoperable guidance is offered to clients, to remediate failures by resource servers due to insufficient authorization details.

This document addresses this interoperability challenge, allowing clients to dynamically discover metadata instead of relying on out-of-band agreements, as well as standardizes failure signaling including interoperable remediation when insufficient authorization details are the cause of failure.

--- middle

# Introduction

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} allows OAuth clients to request detailed and structured authorization, enabling advanced authorization models across domains such as banking and healthcare.

However, RAR {{RFC9396}} does not specify how clients discover metadata describing valid authorization details objects. Such metadata and documentation are obtained out-of-band.

This document defines:

* A new authorization server endpoint: `authorization_details_types_metadata_endpoint`, providing authorization details type metadata, including documentation and JSON Schema definitions {{JSON.Schema}}.
* A new normative OAuth 2.0 WWW-Authenticate Error Code, for resource servers to indicate `insufficient_authorization` as the cause of the error.
* A new OAuth 2.0 WWW-Authenticate response parameter, `authorization_remediation`, providing actionable authorization details objects, to be used directly for remediation in a follow-up OAuth request.
* Authorization server considerations for when RAR authorization details objects should perhaps be omitted from JWT access tokens and provided instead through token instrospection.

Providing clients with actionable authorization details objects enables:

* Interoperability benefit as clients can simply and directly proceed to remediate, without first learning how to construct valid authorization details objects.
* Support for ephemeral, interaction-specific attributes included by the resource server, such as a risk profile or an internal interaction identifier, guiding authorization servers on the required authentication strength and consent flows.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

Client remediates using actionable authorization details objects provided by resource server:

~~~ ascii-art
                                                +--------------------+
             +----------+ (B) API Request       |                    |
             |          |---------------------->|      Resource      |
(A) User +---|          |                       |       Server       |
   Starts|   |          |<----------------------|                    |
   Flow  +-->|  Client  | (C) 401 Unauthorized     +--------------------+
             |          |     WWW-Authenticate: Bearer
             |          |     error="insufficient_authorization",
             |          |     error_description=[human readable message],
             |          |     authorization_remediation=[required
             |          |     authorization_details]
             |          |        :
             |          |        :              +--------------------+
             |          |        :              |   Authorization    |
             |          | (D) Authorization     |      Server        |
             |          |     Request + RAR     |+------------------+|
             |          |---------------------->||                  ||
             |          |                       ||  Authorization   ||
             |          |<----------------------||    Endpoint      ||
             |          | (E) Authorization Code||                  ||
             |          |        :              |+------------------+|
             |          |        :              |                    |
             |          | (F) Token Request     |+------------------+|
             |          |---------------------->||                  ||
             |          |                       || Token Endpoint   ||
             |          |<----------------------||                  ||
             |          | (G) Access Token      |+------------------+|
             |          |        :              +--------------------+
             |          |        :
             |          |        :
             |          | (H) Retry API Call    +--------------------+
             |          |     with Token        |                    |
             |          |---------------------->|      Resource      |
             |          |                       |       Server       |
             |          |<----------------------|                    |
             |          | (I) 200 OK + Resource +--------------------+
             |          |
             +----------+
~~~
Figure: Client remediates using actionable authorization details objects provided by resource server

- (A) The user starts the flow.
- (B) The client calls an API with an access token.
- (C) Resource server returns HTTP 401 with a WWW-Authenticate header with error code `insufficient_authorization` and in `authorization_remediation` the **required authorization details objects**.
- (D) The client uses the provided authorization details objects in a new OAuth + RAR {{RFC9396}} request.
- (E) Authorization server returns authorization code.
- (F-G) The client exchanges authorization code for access token.
- (H) The client makes an API request with the (RAR) access token.
- (I) Resource server validates access token and returns successful response.

# Remediation of failures due to insufficient authorization

This document defines:

* The authentication error code `insufficient_authorization` for the `WWW-Authenticate` header. Resource servers SHOULD return `insufficient_authorization` when access is denied due to missing or insufficient authorization details.
* The `authorization_remediation` error parameter, which contains a base64url-encoded JSON object guiding the client on remediating the error. Its attributes are:

   * `authorization_details`: RECOMMENDED. Array of actionable authorization details objects, matching the format specified in RAR {{RFC9396}} for the `authorization_details` request parameter, built using the failed resource request. Their inclusion in successful new OAuth grant SHALL satisfy the resource's requirements and remediate the failure.
   * `authorization_reference`: RECOMMENDED. An opaque string generated by the resource server to enable the client to select an existing access token associated with equivalent authorization details, without requiring the client to understand the semantics of the authorization details object:

      * Resource server SHOULD generate the `authorization_reference` by canonicalizing and hashing the authorization_details object or an equivalent stable representation, so that the same or semantically equivalent authorization details produce the same authorization_reference value.
      * The value MUST NOT reveal any sensitive or private information.
      * Clients MUST treat this value as opaque and MUST NOT attempt to interpret or derive meaning from it.
      * Returning stable `authorization_reference` values enables clients to reliably match existing tokens to incoming `authorization_remediation` responses, to avoid requesting new tokens when a matching token is already in their possession.
      * The resource server SHALL NOT include this attribute when tokens issued for the provided `authorization_details` are intended for single-use only.

Notes:

* The `error_description` parameter MAY be included to provide a human-readable description.
* The provided `authorization_details` are intended to be interoperable with all OAuth specifications and usable in any grant flow supporting RAR.
* Deployments where resource servers have out-of-band agreements with clients to provide other types of payloads for authorization failure remediation, MAY define and use different attributes of `authorization_remediation` as they see fit.

Example HTTP response from a direct debit resource:

    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer error="insufficient_authorization",
    error_description="Additional authorization is required",
    authorization_remediation=eyJhdXRob3JpemF0aW9uX2RldGFpbHMiOlt7InR5cGUiOiJkaX...

The decoded `authorization_remediation` contents in this example are:

    {
        "authorization_details": [{
                "type": "direct_debit_mandate",
                "DebtorAccount": {
                    "SchemeName": "UK.OBIE.SortCodeAccountNumber",
                    "Identification": "08080021325698",
                    "Name": "JohnDoe"
                },
                "CreditorAgent": {
                    "SchemeName": "UK.OBIE.BICFI",
                    "Identification": "NWBKGB22"
                },
                "CreditorAccount": {
                    "SchemeName": "UK.OBIE.SortCodeAccountNumber",
                    "Identification": "08080021325698",
                    "Name": "ACMECorp"
                },
                "MandateStatus": "Active",
                "CreationDateTime": "2026-06-01T09:00:00+00:00"
            }
        ],
        "authorization_reference": "Yb7q3AC5d"
    }

Example HTTP response from a payment initiation resource:

    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer error="insufficient_authorization",
    error_description="Additional authorization is required",
    authorization_remediation=eyJhdXRob3JpemF0aW9uX2RldGFpbHMiOlt7InR5cGUiOiJkaX...

The decoded `authorization_remediation` contents in this example are:

    {
        "authorization_details": [{
               "type": "payment_initiation",
               "instructed_amount": {
                  "currency": "EUR",
                  "amount": "100.00"
               },
               "creditor_account": {
                  "iban": "DE02120300000000202051"
               }
           }
       ]
    }

# Authorization Details Types Metadata Endpoint

The following authorization server metadata {{RFC8414}} parameter is introduced to indicate the server's support for Authorization Details Types Metadata:

"authorization_details_types_metadata_endpoint":
:    OPTIONAL.  The URL of the Authorization Details Types Metadata endpoint.

The Authorization Details Types Metadata endpoint is called with HTTP GET and responds with Content-Type `application/json` and a JSON object whose members are authorization details type identifiers.

Each member value is an object describing a single authorization details type.

    {
      "type": {
        "version": "...",
        "description": "...",
        "documentation_uri": "...",
        "schema": { },
        "schema_uri": "...",
        "examples": [ ]
      }
    }

Attribute definition:

"version":
: OPTIONAL. String identifying the version of the authorization details type definition. The value is informational and does not imply semantic version negotiation.

"description":
: OPTIONAL. String containing a description of the authorization details type. Clients MUST NOT rely on this value for authorization or validation decisions.

"documentation_uri":
: OPTIONAL. URI referencing external documentation describing the authorization details type.

"schema":
: The `schema` attribute contains a JSON Schema document {{JSON.Schema}} that describes a single authorization details object. The schema MUST validate exactly one authorization details object and MUST restrict the `type` attribute to the corresponding authorization details type identifier. This attribute is REQUIRED unless `schema_uri` is specified. If present, `schema_uri` MUST NOT be included.

"schema_uri":
: The `schema_uri` attribute is an absolute URI, as defined by RFC 3986 {{RFC3986}}, referencing a JSON Schema document describing a single authorization details object. The referenced schema MUST satisfy the same requirements as the `schema` attribute. This attribute is REQUIRED unless `schema` is specified. If this attribute is present, `schema` MUST NOT be present.

"examples":
: OPTIONAL. An array of example authorization details objects. Examples are non-normative.

See Examples {{metadata-examples}} for non-normative response example.

# RAR objects in JWT access tokens

Pursuant with RAR {{RFC9396}} section 9, authorization servers MUST provide approved RAR objects to resource servers for enforcement. The authorization server MAY add the `authorization_details` attribute to access tokens in JSON Web Token (JWT) format or to token introspection responses.

There may however be cases, where due to various considerations such as token size or information privacy, including approved RAR objects in JWT access tokens would be advised against.

It is RECOMMENDED that when an authorization server issues JWT access tokens, it should consider the size, sensitivity, and privacy implications of including the authorization_details attribute. Where appropriate, the authorization server SHOULD omit this attribute from JWT tokens and instead provide the approved RAR objects to resource servers via the token introspection endpoint. This endpoint SHOULD use appropriate client authentication methods to prevent unauthorized access, in case of token leakage.

# Processing Rules

## Client Processing Rules

When a client receives an HTTP 401 response with WWW-Authenticate error code `insufficient_authorization` and an `authorization_remediation` parameter, it SHOULD process it as follows:

### Step 1 - Parse the remediation response

The client decodes the base64url-encoded `authorization_remediation` JSON object and extracts:

- `authorization_details` (RECOMMENDED): the actionable RAR objects.
- `authorization_reference` (OPTIONAL): an opaque string for token-bag lookup.


### Step 2 - Attempt token reuse via `authorization_reference` (if present)

1. If the `authorization_remediation` contains an `authorization_reference` attribute, the client SHOULD search its **in-session tokens** for a token previously associated with that reference value **and** the same resource server origin.
2. Matching is a simple string comparison — the client MUST NOT attempt to compute, parse, or derive meaning from the reference value.
3. If a matching, non-expired token is found, the client MAY retry the failing request with that token. If the retry also fails with `insufficient_authorization`, the client MUST NOT retry again with the same token for the same reference and SHOULD proceed to Step 3.
4. If no matching token is found, the client proceeds to Step 3.

### Step 3 - Obtain a new token via OAuth + RAR

The client proceeds to this step if: (a) no `authorization_reference` was present, (b) no matching token in client's possession was found, (c) a matched token was rejected by the resource server (Step 2, item 4), or (d) the client elects to skip an existing token lookup and use `authorization_details` directly.

1. The client uses the `authorization_details` from the `authorization_remediation` response in a new OAuth authorization request per {{RFC9396}}. The client MAY use any grant type or extension that supports RAR (including PAR {{RFC9126}}, JAR {{RFC9101}}, etc.).
2. Upon successful token issuance, if the triggering resource server response included an `authorization_reference`, the client SHOULD persist the newly obtained token associated with that reference value and the resource server origin in its in-session token storage. This token-to-reference association enables future lookups in Step 2 when the same `authorization_reference` is encountered again.
3. The client retries the failing request with the newly obtained token.

### Step 4 - Handle continued failure

If after obtaining a new token and retrying, the resource server still returns `insufficient_authorization`:

- If the new response contains a **different** `authorization_reference`, the client MAY attempt remediation again (subject to implementation-defined retry limits).
- If the new response contains the **same** `authorization_reference`, the client MUST NOT loop — it SHOULD treat the failure as non-remediable and report an error to the user or calling application.

### Additional guidance

- Clients MAY ignore `authorization_reference` entirely if they do not implement token caching or reuse. In that case, each `insufficient_authorization` response triggers a fresh authorization request using the provided `authorization_details`.
- If the client's current authorization server does not support the required authorization details types (as indicated by its metadata), the client MAY use Protected Resource Metadata {{RFC9728}} to discover alternative authorization servers for the resource.
- The token storage MUST be scoped per end-user session. Concurrent users operating through the same client instance MUST maintain separate token storage instances.

## Resource Server Processing Rules

When a resource server receives a request with an OAuth token:

### Step 1 - Validate the access token

Verify token validity following {{RFC6750}} or {{RFC9068}} if JWT profiled. If the token is invalid for reasons other than insufficient authorization details, return the appropriate existing error code (e.g., `invalid_token`).

### Step 2 - Verify authorization details (if present)

Determine whether the token carries sufficient authorization details for the requested operation. Authorization details MAY be obtained from the JWT access token payload or via token introspection {{RFC7662}}.

### Step 3 - If authorization details are missing or insufficient

The resource server responds with an error per the bearer token error framework {{RFC6750}} Section 3. The specific error code depends on the nature of the failure:
- If the token is valid but lacks sufficient **scope**, the resource server returns `insufficient_scope` per {{RFC6750}} Section 3.1.
- If the token is valid but lacks sufficient **authentication context** (e.g., ACR/AMR level), the resource server returns `insufficient_user_authentication` per {{RFC9470}}.
- If the token is valid but lacks sufficient **authorization details**, the RS returns `insufficient_authorization` per Section 4 of this document, with an `authorization_remediation` parameter as defined in Section 4.1.
The `authorization_remediation` parameter carries the actionable `authorization_details` and optional `authorization_reference` as specified in Section 4. The resource server constructs these per the rules defined below.

## Limitations and Considerations for `authorization_reference`

Implementers should be aware of the following limitations:

### Token reuse is opportunistic, not guaranteed

A matching `authorization_reference` with client's existing tokens does NOT guarantee the token will be accepted by the resource server. The token may have been issued under conditions that no longer apply:

- The resource owner may have revoked consent since the token was issued.
- Contextual risk may have changed (e.g., geolocation, device posture), causing the resource server to require stronger authorization ceremonies.
- The authorization server may have issued the token with a subset of the requested authorization details (per {{RFC9396}} Section 7).

Clients MUST handle the case where a reused token is rejected despite matching the `authorization_reference` (see Section 7.1, Step 2, item 4).

### `authorization_reference` does not replace `authorization_details`

The `authorization_reference` is an optimization for token selection. It is NOT a substitute for `authorization_details`:

- `authorization_details` is RECOMMENDED in every `authorization_remediation` response providing actionable RAR objects and is what the client uses when initiating a new authorization request.
- `authorization_reference` is RECOMMENDED and enables the client to avoid unnecessary authorization flows when it already possesses a suitable token.

Clients that do not implement token caching MAY safely ignore `authorization_reference` with no loss of interoperability.

### Loop prevention

If the resource server consistently returns the same `authorization_reference` and rejects tokens obtained via the associated `authorization_details`, the client may enter an infinite loop. To prevent this:

- Clients MUST implement a maximum retry count (RECOMMENDED: 1 retry with a cached token, then 1 fresh authorization attempt, then fail).
- If a freshly obtained token (from a new authorization flow using the resource server provided `authorization_details`) is immediately rejected by the same resource server with the same `authorization_reference`, the client MUST stop and report the error.

### No cross-resource-server portability

The `authorization_reference` value is scoped to the producing resource server. It MUST NOT be used for token selection when interacting with a different resource server, even if the two servers enforce similar authorization details types.

### Analogy to scope-based token selection

The `authorization_reference` mechanism is analogous to how clients select tokens based on OAuth scopes in traditional deployments. Just as a client maintains a mapping of `{scope → token}` and selects the appropriate token for each resource server call, this mechanism extends that pattern to RAR:

| Traditional (scope-based) | RAR + authorization_reference |
|---|---|
| Resource server returns `insufficient_scope` with required `scope` value | Resource server returns `insufficient_authorization` with `authorization_remediation` |
| Client checks if it has a token with matching scope | Client checks if it has a token with matching `authorization_reference` |
| Simple string comparison on scope values | Simple string comparison on reference values |
| If not found, request new token with required scope | If not found, request new token with provided `authorization_details` |

The key advantage: clients do not need to understand, parse, or compare complex JSON `authorization_details` objects — the resource server has already reduced the comparison to an opaque string.

# Security Considerations {#security-considerations}

## Confidentiality of resource server provided authorization_details

Resource servers when providing actionable `authorization_details` SHOULD NOT include sensitive data in those objects. This is consistent with RAR {{RFC9396}} `authorization_details` OAuth request parameter, representing **request** semantics.

Confidentiality-preserving `authorization_details` types SHOULD NOT include sensitive data. Instead, the end-user SHOULD provide such information when interacting with the authorization server.

Alternatively, `authorization_details` MAY refer to specific end-user resources using opaque reference handles (e.g., "account_1a" instead of using explicit IBAN).

# IANA Considerations

## OAuth 2.0 WWW-Authenticate Error Code Registry

| Error Code | Error Usage Location | Change Controller | Specification Document |
|------------|----------------------|-------------------|------------------------|
| insufficient_authorization | Resource access error response | IETF | RFC XXXX, Section X |

## OAuth Authorization Server Metadata Registry

This specification registers the following authorization server metadata parameter in the OAuth Authorization Server Metadata registry:

| Metadata Name | Metadata Description | Change Controller | Specification Document |
|---------------|----------------------|-------------------|------------------------|
| authorization_details_types_metadata_endpoint | URL of the Authorization Details Types Metadata endpoint | IETF | RFC XXXX, Section X |


--- back

# Examples

This section provides non-normative examples of how this specification may be used to support specific use cases.

## Authorization Server Metadata Examples {#metadata-examples}

### Example authorization_details_types_metadata_endpoint response with Payment Initiation

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "payment_initiation": {
            "version": "1.0",
            "description": "Authorization to initiate a single payment from a payer account to a creditor account.",
            "documentation_uri": "https://example.com/docs/payment-initiation",
            "schema": {
                "$schema": "https://json-schema.org/draft/2020-12/schema",
                "title": "Payment Initiation Authorization Detail",
                "type": "object",
                "required": [
                    "type",
                    "instructed_amount",
                    "creditor_account"
                ],
                "properties": {
                    "type": {
                        "const": "payment_initiation",
                        "description": "Authorization details type identifier."
                    },
                    "actions": {
                        "type": "array",
                        "description": "Permitted actions for this authorization.",
                        "items": {
                            "type": "string",
                            "enum": ["initiate"]
                        },
                        "minItems": 1,
                        "uniqueItems": true
                    },
                    "instructed_amount": {
                        "type": "object",
                        "description": "Amount and currency of the payment to be initiated.",
                        "required": ["currency", "amount"],
                        "properties": {
                            "currency": {
                                "type": "string",
                                "description": "ISO 4217 currency code.",
                                "pattern": "^[A-Z]{3}$"
                            },
                            "amount": {
                                "type": "string",
                                "description": "Decimal monetary amount represented as a string.",
                                "pattern": "^[0-9]+(\\.[0-9]{1,2})?$"
                            }
                        }
                    },
                    "creditor_account": {
                        "type": "object",
                        "description": "Account to which the payment will be credited.",
                        "required": ["iban"],
                        "properties": {
                            "iban": {
                                "type": "string",
                                "description": "International Bank Account Number (IBAN).",
                                "pattern": "^[A-Z0-9]{15,34}$"
                            }
                        }
                    },
                    "remittance_information": {
                        "type": "string",
                        "description": "Unstructured remittance information for the payment.",
                        "maxLength": 140
                    }
                }
            }
        }
    }

### Example authorization_details_types_metadata_endpoint response for the Norwegian Health Sector (HelseID)

    HTTP/1.1 200 OK
    Content-Type: application/json

    {

        "helseid_authorization": {
            "version": "1.0",
            "description": "Allows the OAuth client to pass organization information to HelseID.",
            "documentation_uri": "https://utviklerportal.nhn.no/informasjonstjenester/helseid/bruksmoenstre-og-eksempelkode/bruk-av-helseid/docs/tekniske-mekanismer/organisasjonsnumre_enmd",
            "schema": {
                "$schema": "http://json-schema.org/draft-07/schema#",
                "title": "Organization numbers for a multi-tenant client",
                "type": "object",
                "properties": {
                    "type": {
                        "type": "string",
                        "const": "helseid_autorization"
                    },
                    "practitioner_role": {
                        "type": "object",
                        "properties": {
                            "organization": {
                                "type": "object",
                                "properties": {
                                    "identifier": {
                                        "type": "object",
                                        "properties": {
                                            "system": {
                                                "type": "string"
                                            },
                                            "type": {
                                                "type": "string"
                                            },
                                            "value": {
                                                "type": "string"
                                            }
                                        },
                                        "required": [
                                            "system",
                                            "type",
                                            "value"
                                        ]
                                    }
                                },
                                "required": [
                                    "identifier"
                                ]
                            }
                        },
                        "required": [
                            "organization"
                        ]
                    }
                },
                "required": [
                    "type",
                    "practitioner_role"
                ]
            }
        },
        "helseid_trust_framework": {
            "version": "1.0",
            "description": "HelseID Trust Framework Information",
            "documentation_uri": "https://utviklerportal.nhn.no/informasjonstjenester/helseid/bruksmoenstre-og-eksempelkode/bruk-av-helseid/docs/tekniske-mekanismer/trust-framework",
            "schema": {
                "$schema": "http://json-schema.org/draft-07/schema#",
                "description": "Complete Trust Framework structure",
                "documentation_uri": "https://utviklerportal.nhn.no/informasjonstjenester/helseid/bruksmoenstre-og-eksempelkode/bruk-av-helseid/docs/tillitsrammeverk/profil_for_tillitsrammeverkmd",
                "type": "object",
                "properties": {
                    "type": {
                        "type": "string",
                        "const": "nhn:tillitsrammeverk:parameters"
                    },
                    "practitioner": {
                        "type": "object",
                        "properties": {
                            "authorization": {
                                "type": "object",
                                "properties": {
                                    "code": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "code",
                                    "system"
                                ]
                            },
                            "legal_entity": {
                                "type": "object",
                                "properties": {
                                    "id": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "id",
                                    "system"
                                ]
                            },
                            "point_of_care": {
                                "type": "object",
                                "properties": {
                                    "id": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "id",
                                    "system"
                                ]
                            },
                            "department": {
                                "type": "object",
                                "properties": {
                                    "id": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "id",
                                    "system"
                                ]
                            }
                        },
                        "required": [
                            "authorization",
                            "legal_entity",
                            "point_of_care",
                            "department"
                        ]
                    },
                    "care_relationship": {
                        "type": "object",
                        "properties": {
                            "healthcare_service": {
                                "type": "object",
                                "properties": {
                                    "code": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "code",
                                    "system"
                                ]
                            },
                            "purpose_of_use": {
                                "type": "object",
                                "properties": {
                                    "code": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "code",
                                    "system"
                                ]
                            },
                            "purpose_of_use_details": {
                                "type": "object",
                                "properties": {
                                    "code": {
                                        "type": "string"
                                    },
                                    "system": {
                                        "type": "string"
                                    }
                                },
                                "required": [
                                    "code",
                                    "system"
                                ]
                            },
                            "decision_ref": {
                                "type": "object",
                                "properties": {
                                    "id": {
                                        "type": "string"
                                    },
                                    "user_selected": {
                                        "type": "boolean"
                                    }
                                },
                                "required": [
                                    "id",
                                    "user_selected"
                                ]
                            }
                        },
                        "required": [
                            "healthcare_service",
                            "purpose_of_use",
                            "purpose_of_use_details",
                            "decision_ref"
                        ]
                    },
                    "patients": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "point_of_care": {
                                    "type": "object",
                                    "properties": {
                                        "id": {
                                            "type": "string"
                                        },
                                        "system": {
                                            "type": "string"
                                        }
                                    },
                                    "required": [
                                        "id",
                                        "system"
                                    ]
                                },
                                "department": {
                                    "type": "object",
                                    "properties": {
                                        "id": {
                                            "type": "string"
                                        },
                                        "system": {
                                            "type": "string"
                                        }
                                    },
                                    "required": [
                                        "id",
                                        "system"
                                    ]
                                }
                            },
                            "required": [
                                "point_of_care",
                                "department"
                            ]
                        }
                    }
                },
                "required": [
                    "type",
                    "practitioner",
                    "care_relationship",
                    "patients"
                ]
            }
        }
    }

# Document History

-05

* Removed required authorization details types.
* Changed from HTTP 403 to 401.
* Moved resource servers response from body to WWW-Authenticate header.
* Renamed authorization_hint to authorization_reference and clarified its usage.
* Clarified authorization server broader considerations on omitting RAR from JWT access tokens.
* Clarified document's interoperability with any OAuth rfc and any grant that supports RAR.


-04

* Moved required authorization details types from resource metadata to resource server's response.
* Adapted resource server processing rules to reflect error signaling and handling of large RAR payloads.

-03

* Added authorization_reference to guide client on token selection and updated client processing rules accordingly
* Added security consideration on confidentiality of RS-provided authorization_details
* Added authorization server considerations for handling large RAR objects in JWT access tokens

-02

* Defined the required types expression
* Added Protected Resource Metadata examples

-01

* Authorization details moved to HTTP body and made OPTIONAL
* Metadata pointer from resource metadata url, full authorization details types metadata on authorization server new endpoint

-00

* Document creation


# Acknowledgments
{:numbered="false"}

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that helped shape the final specification: Rune Grimstad, Justin Richer, Jeff Lombardo, Judith Kahrer, Pieter Kasselman.
