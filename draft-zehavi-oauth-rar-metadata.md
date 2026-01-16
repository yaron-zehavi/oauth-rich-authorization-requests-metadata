---
title: "OAuth 2.0 RAR Metadata and Error Signaling"
abbrev: "OAuth 2.0 RAR Metadata and Error Signaling"
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
  RFC6749:
  RFC8414:
  RFC9396:
  RFC9728:
  IANA.oauth-parameters:
  JSON.Schema:
    title: "JSON Schema: A Media Type for Describing JSON Documents"
    target: https://json-schema.org/draft/2020-12/json-schema-core
    date: June 16, 2022
    author:
      - ins: A. Wright, Ed.
      - ins: H. Andrews, Ed.
      - ins: B. Hutton, Ed. Postman
      - ins: G. Dennis

--- abstract

OAuth 2.0 Rich Authorization Requests (RAR), as defined in {{RFC9396}}, enables clients to request fine-grained authorization using structured JSON objects. While RAR {{RFC9396}} standardizes the exchange and handling of authorization details, it does not define a mechanism for clients to discover how to construct valid authorization details types.

This document defines a machine-readable metadata structure for advertising authorization details type documentation and JSON Schema {{JSON.Schema}} definitions by authorization servers and via OAuth Resource Server Metadata {{RFC9728}}.

This document also defines a new WWW-Authenticate OAuth error code, `insufficient_authorization_details`, enabling resource servers to indicate inadequate authorization details as the cause of failure.
It also defines an OPTIONAL response body which MAY be returned alongside the `insufficient_authorization_details` error, providing an authorization details object, which may serve as an example or be used directly in a subsequent OAuth request.

--- middle

# Introduction

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} allows OAuth clients to request structured, fine-grained authorization, beyond the coarse-grained access offered by simple scopes, which has enabled advanced authorization models across domains, such as open banking & API marketplaces.

However, RAR {{RFC9396}} does not specify how a client learns how to construct syntactically valid authorization details objects. As a result, clients must rely on out-of-band documentation or static ecosystem profiles, limiting interoperability and preventing dynamic client behavior.

This document addresses this gap by defining:

* A metadata structure for authorization details types, containing human-readable documentation as well as embedded JSON Schema definitions {{JSON.Schema}}.
* Discovery through a new authorization server `rar_types_metadata_endpoint`, as well as via OAuth 2.0 Protected Resource Metadata {{RFC9728}}.
* A standardized error signaling mechanism based on WWW-Authenticate response header, allowing resource servers to specify `insufficient_authorization_details` as the cause of error, as well as an OPTIONAL response body enabling resource servers to provide an informative authorization details object, whose inclusion in a new OAuth request shall result in a token satisfying the endpoint's requirements.

The OPTIONAL solution pattern in which resource server returns an actionable authorization details object offers:

* High interoperability and simplification by relieving clients from having to figure out how to construct valid authorization details objects, instead providing them the required authorization_details object, to be included in a subsequent OAuth request.
* Support for including ephemeral, interaction-specific details in the authorization details object, such as for example a risk score, a risk profile or an internal interaction identifier. This enables resource servers to guide the authorization server as to the required authentication strength and consent flow.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

There are two main proposed flows:

* Client learns to construct valid authorization details objects using authorization details types metadata.
* Client obtains an actionable authorization details object from resource server's error response.

## Client learns to construct valid authorization details objects using metadata

~~~ ascii-art
                                                +--------------------+
             +----------+ (B) API Request       |                    |
             |          |---------------------->|      Resource      |
(A) User +---|          |                       |       Server       |
   Starts|   |          |<----------------------|                    |
   Flow  +-->|          | (C) 403 Forbidden     +--------------------+
             |          |     WWW-Authenticate
             |          |     error="insufficient_authorization_details"
             |          |        :
             |          |        :              +--------------------+
             |          |        :              |  Authorization or  |
             |          | (D) Metadata Request  |  Resource Server   |
             |          |---------------------->|+------------------+|
             |          |                       ||    Metadata      ||
             |  Client  |<----------------------||    Endpoint      ||
             |          | (E) Metadata Response |+------------------+|
             |          |        :              +--------------------+
             |          | (F) Construct RAR
             |          |     Using Metadata
             |          |        :              +--------------------+
             |          | (G) Authorization     |   Authorization    |
             |          |     Request + RAR     |      Server        |
             |          |---------------------->|+------------------+|
             |          |                       ||  Authorization   ||
             |          |<----------------------||    Endpoint      ||
             |          | (H) Authorization Code||                  ||
             |          |        :              |+------------------+|
             |          |        :              |                    |
             |          | (I) Token Request     |+------------------+|
             |          |---------------------->||                  ||
             |          |                       || Token Endpoint   ||
             |          |<----------------------||                  ||
             |          | (J) Access Token      |+------------------+|
             |          |        :              +--------------------+
             |          |        :
             |          | (K) API Call with
             |          |     Access Token      +--------------------+
             |          |---------------------->|                    |
             |          |                       |   Resource Server  |
             |          |<----------------------|                    |
             |          | (L) 200 OK + Resource +--------------------+
             |          |
             +----------+
~~~
Figure: Client learns to construct valid authorization details objects from metadata

- (A) The user starts the flow.
- (B) The client calls an API with an access token.
- (C) Resource server returns HTTP 403 forbidden including a WWW-Authenticate header with error code `insufficient_authorization_details`.
- (D-E) The client consumes authorization details type metadata from authorization server's `rar_types_metadata_endpoint`, or from resource server Protected Resource Metadata {{RFC9728}}.
- (F-G) The client constructs a valid authorization details object and makes an OAuth + RAR {{RFC9396}} request.
- (H) Authorization server returns authorization code.
- (I-J) The client exchanges authorization code for access token.
- (K) The client makes API request with access token.
- (L) Resource server validates access token and returns successful response.

## Client obtains authorization details object from resource server's error response

~~~ ascii-art
                                                +--------------------+
             +----------+ (B) API Request       |                    |
             |          |---------------------->|      Resource      |
(A) User +---|          |                       |       Server       |
   Starts|   |          |<----------------------|                    |
   Flow  +-->|  Client  | (C) 403 Forbidden     +--------------------+
             |          |     WWW-Authenticate
             |          |     error="insufficient_authorization_details"
             |          |     + **authorization_details**
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
             |          | (G) Token Request     |+------------------+|
             |          |---------------------->||                  ||
             |          |                       || Token Endpoint   ||
             |          |<----------------------||                  ||
             |          | (H) Access Token      |+------------------+|
             |          |        :              +--------------------+
             |          |        :
             |          |        :
             |          | (I) Retry API Call    +--------------------+
             |          |     with Token        |                    |
             |          |---------------------->|      Resource      |
             |          |                       |       Server       |
             |          |<----------------------|                    |
             |          | (J) 200 OK + Resource +--------------------+
             |          |
             +----------+
~~~
Figure: Client obtains authorization details object from resource server's error response

- (A) The user starts the flow.
- (B) The client calls an API with an access token.
- (C) Resource server returns HTTP 403 forbidden because the access token does not contain required authorization details. Resource server's response includes a WWW-Authenticate header with the authorization details object requiring approval.
- (D) The client uses the obtained authorization details object in a new OAuth + RAR {{RFC9396}} request.
- (E) Authorization server returns authorization code.
- (G-H) The client exchanges authorization code for access token.
- (I) The client makes API request with access token.
- (J) Resource server validates access token and returns successful response.

# Authorization Details Type Metadata

## Overview

This document defines:

* A new metadata structure: `authorization_details_types_metadata`, detailing documentation and schema of authorization details types used with Rich Authorization Requests {{RFC9396}}.
* A new authorization server metadata endpoint: `rar_types_metadata_endpoint` which returns the `authorization_details_types_metadata` structure.
* A new WWW-Authenticate response header error code: `insufficient_authorization_details`, which resource servers MAY use as a normative indication to client, that the token used to call the endpoint was insufficient because it lacked or contained otherwise inadequate `authorization_details`.
* An OPTIONAL response body which resource servers MAY use, in addition to the WWW-Authenticate response header, to provide an actionable authorization details **object**, whose inclusion in a subsequent OAuth request, shall result in an access token satisfying the endpoint's requirements.

This document also proposes:

* The existing `authorization_details_types_supported` metadata attributed defined by RAR {{RFC9396}}
* As well as the herein defined `authorization_details_types_metadata`

Shall both be included as OPTIONAL response attributes in Protected Resource Metadata {{RFC9728}}.

## Metadata Location

The `authorization_details_types_metadata` attribute may be included in:

* Authorization Server Metadata {{RFC8414}}, to describe authorization details types supported by the authorization server. If present, its keys MUST be a subset of the values listed in `authorization_details_types_supported` as defined in {{RFC9396}}.
* OAuth 2.0 Protected Resource Metadata {{RFC9728}}, to describe authorization details types accepted by the protecred resource.

## Metadata Structure

The `authorization_details_types_metadata` attribute is a JSON object whose keys are authorization details type identifiers. Each value is an object describing a single authorization details type.

    {
    "authorization_details_types_metadata": {
        "type": {
        "version": "...",
        "description": "...",
        "documentation_uri": "...",
        "schema": { },
        "schema_uri": "...",
        "examples": [ ]
        }
    }
    }

Attributes definition:

"version":
: OPTIONAL. String identifying the version of the authorization details type definition. The value is informational and does not imply semantic version negotiation.

"description":
: OPTIONAL. String containing a human-readable description of the authorization details type. Clients MUST NOT rely on this value for authorization or validation decisions.

"documentation_uri":
: OPTIONAL. URI referencing external human-readable documentation describing the authorization details type.

"schema":
: The `schema` attribute is a JSON Schema document {{JSON.Schema}} describing a single authorization detail object. The schema MUST validate a single authorization detail object and MUST constrain the `type` attribute to the authorization detail type identifier. This attribute is REQUIRED unless `schema_uri` is specified. If this attribute is present, `schema_uri` MUST NOT be present.

"schema_uri":
: The `schema_uri` attribute is an absolute URI, as defined by RFC 3986 {{RFC3986}}, referencing a JSON Schema document describing a single authorization details object. The referenced schema MUST satisfy the same requirements as the `schema` attribute. This attribute is REQUIRED unless `schema` is specified. If this attribute is present, `schema` MUST NOT be present.

"examples":
: OPTIONAL. An array of example authorization details objects. Examples are non-normative.

## Metadata Examples

### Example RAR Metadata: Payment Initiation

    {
        "authorization_details_types_supported": ["payment_initiation"],
        "authorization_details_types_metadata": {
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
                            "description": "Authorization detail type identifier."
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
                            },
                            "additionalProperties": false
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
                            },
                            "additionalProperties": false
                        },
                        "remittance_information": {
                            "type": "string",
                            "description": "Unstructured remittance information for the payment.",
                            "maxLength": 140
                        }
                    },
                    "additionalProperties": false
                }
            }
        }
    }

# Resource Server Error Signaling with insufficient_authorization_details

## Overview

This document defines a new OAuth error code, `insufficient_authorization_details`, for use when access is denied due to missing or insufficient authorization details.

## Error Definition

The error MUST be conveyed using the `WWW-Authenticate` header and MUST include an `authorization_details` parameter.

## authorization_details Error Parameter

The parameter MUST contain a JSON object or array, representing the required authorization details, whose inclusion in a subsequent OAuth request is required to satisfy the resource server's requirements for this specific request.
The value MUST be base64url-encoded.

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details",
    authorization_details="{base64url-encoded JSON of required RAR}"

# Processing Rules

## Client Processing Rules

* Fetch authorization_details_types_metadata from the authorization or resource server's metadata endpoints.
* Locate schema or retrieve schema_uri.
* Construct authorization details conforming to the schema.
* If resource server returns error insufficient_authorization_details, use provided authorization_details in subsequent OAuth request, then provide the obtained token to resource server.

## Authorization Server Processing Rules

* Advertise authorization_details_types_metadata in metadata.
* Validate each authorization detail against schema.
* Enforce additional semantic checks.
* Reject missing or invalid details with standard OAuth error semantics.

## Resource Server Processing Rules

* Advertise authorization_details_types_metadata.
* Verify tokens against required authorization details.
* If insufficient, return HTTP 403 with WWW-Authenticate: Bearer error="insufficient_authorization_details".
* Do not reveal additional sensitive information.

# Security Considerations {#security-considerations}

# IANA Considerations

## OAuth 2.0 Bearer Token Error Registry

| Error Code | Description |
|------------|-------------|
| insufficient_authorization_details | The request is missing required authorization details or the provided authorization details are insufficient. The resource server SHOULD include the required `authorization_details` |

## OAuth Metadata Attribute Registration

The metadata attribute `authorization_details_types_metadata` is defined for OAuth authorization and resource server metadata, as a JSON object mapping authorization details types to documentation, schema, and examples.

--- back

# Examples

This section provides non-normative examples of how this specification may be used to support specific use cases.

## Payment initiation with RAR error signaling

### Client initiates API request

Client uses access token obtained at login to call payment initiation API

    POST /payments HTTP/1.1
    Host: server.example.com
    Content-Type: application/json
    Authorization: Bearer eyj... (access token from login)

    {
        "type": "payment_initiation",
        "locations": [
            "https://server.example.com/payments"
        ],
        "instructedAmount": {
            "currency": "EUR",
            "amount": "123.50"
        },
        "creditorName": "Merchant A",
        "creditorAccount": {
            "bic": "ABCIDEFFXXX",
            "iban": "DE02100100109307118603"
        }
    }

### Resource server signals insufficient_authorization_details

Resource server requires payment approval and responds with:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details",
        resource_metadata="https://server.example.com/.well-known/oauth-protected-resource/payments",
        authorization_details=W3sKICAgICJ0eXBlIjogInBheW1lbnRfaW5pdGlhdGlvbiIsCiAgICAibG9jYXRpb25zIjogWwogICAgICAgICJodHRwczovL2V4YW1wbGUuY29tL3BheW1lbnRzIgogICAgXSwKICAgICJpbnN0cnVjdGVkQW1vdW50IjogewogICAgICAgICJjdXJyZW5jeSI6ICJFVVIiLAogICAgICAgICJhbW91bnQiOiAiMTIzLjUwIgogICAgfSwKICAgICJjcmVkaXRvck5hbWUiOiAiTWVyY2hhbnQgQSIsCiAgICAiY3JlZGl0b3JBY2NvdW50IjogewogICAgICAgICJiaWMiOiAiQUJDSURFRkZYWFgiLAogICAgICAgICJpYmFuIjogIkRFMDIxMDAxMDAxMDkzMDcxMTg2MDMiCiAgICB9LAogICAgImludGVyYWN0aW9uSWQiOiAiZjgxZDRmYWUtN2RlYy0xMWQwLWE3NjUtMDBhMGM5MWU2YmY2IiwKCSJyaXNrUHJvZmlsZSI6ICJCLTcxIgp9XQ==

The base64 encoded authorization_details decodes to:

    [{
        "type": "payment_initiation",
        "locations": [
            "https://example.com/payments"
        ],
        "instructedAmount": {
            "currency": "EUR",
            "amount": "123.50"
        },
        "creditorName": "Merchant A",
        "creditorAccount": {
            "bic": "ABCIDEFFXXX",
            "iban": "DE02100100109307118603"
        },
        "interactionId": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
        "riskProfile": "B-71"
    }]

Note the resource server has added the ephemeral attributes: `interactionId`, `riskProfile`.

### Client initiates OAuth flow using the provided authorization_details object

After user approves the request, client obtains single-use access token representing the approved payment

### Client re-attempts API request

    POST /payments HTTP/1.1
    Host: server.example.com
    Content-Type: application/json
    Authorization: Bearer eyj... (payment approval access token)

    {
        "type": "payment_initiation",
        "locations": [
            "https://server.example.com/payments"
        ],
        "instructedAmount": {
            "currency": "EUR",
            "amount": "123.50"
        },
        "creditorName": "Merchant A",
        "creditorAccount": {
            "bic": "ABCIDEFFXXX",
            "iban": "DE02100100109307118603"
        }
    }

### Resource server authorizes the request

    HTTP/1.1 201 Accepted
    Content-Type: application/json
    Cache-Control: no-store

    {
        "paymentId": "a81bc81b-dead-4e5d-abff-90865d1e13b1",
        "status": "accepted"
    }

# Document History

-00

* Document creation


# Acknowledgments
{:numbered="false"}
