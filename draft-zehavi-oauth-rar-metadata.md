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
  RFC7662:
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
      - ins: B. Hutton, Ed.
      - ins: G. Dennis

--- abstract

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} standardizes the exchange and processing of authorization details but does not define metadata to describe authorization details types.

This document addresses a practical interoperability challenge regarding metadata of authorization details types, allowing clients to dynamically discover metadata instead of relying on out-of-band agreements.
It also standardizes error signaling, in case insufficient authorization details are provided and offers structured ways of remediation.

--- middle

# Introduction

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} allows OAuth clients to request detailed and structured authorization, enabling advanced authorization models across domains such as banking and healthcare.

However, RAR {{RFC9396}} does not specify how clients discover metadata describing valid authorization details objects. Such metadata and documentation are obtained out-of-band.

This document defines:

* A new authorization server endpoint: `authorization_details_types_metadata_endpoint`, providing metadata for authorization details types, including documentation and JSON Schema definitions {{JSON.Schema}}.
* A new normative OAuth 2.0 WWW-Authenticate Error Code, for resource servers to indicate `insufficient_authorization_details` as the cause of the error, as well as a RECOMMENDED response body specifying required authorization details types to satisfy the resource's requirements.
* An OPTIONAL response body attribute that MAY accompany the insufficient_authorization_details error, providing an informative and actionable authorization details object. This object can be used directly in a follow-up OAuth request.
* RECOMMENDED handling of large RAR {{RFC9396}} authorization details objects when issuing JWT access tokens, to avoid failures due to token sizes exceeding header size restrictions.

Providing actionable authorization details objects in the resource server's error response enables:

* Simplification for clients who can directly remediate without learning to construct valid authorization details objects.
* Support for ephemeral, interaction-specific claims from the resource server, such as a risk profile or an internal interaction identifier, guiding authorization servers on required authentication strength and consent flows.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

There are two main proposed flows:

* Client remediates using **metadata of required authorization details types**.
* Client remediates using **actionable authorization details objects** provided by resource server.

## Client remediates using metadata of required authorization details types

~~~ ascii-art
                                                +---------------------+
             +----------+ (B) API Request       |                     |
             |          |---------------------->|      Resource       |
(A) User +---|          |                       |       Server        |
   Starts|   |          |<----------------------|                     |
   Flow  +-->|          | (C) 403 Forbidden     +---------------------+
             |          |     WWW-Authenticate: Bearer
             |          |     error="insufficient_authorization_details"
             |          |        +
             |          |     HTTP body specifies
             |          |     required authorization_details
             |          |        :
             |          |                       +---------------------+
             |          |        RAR Types      |    Authorization    |
             |          | (D) Metadata Request  |       Server        |
             |          |---------------------->|+-------------------+|
             |          |                       ||     RAR Types     ||
             |          |<----------------------|| Metadata Endpoint ||
             |          | (E) Metadata Response |+-------------------+|
             |          |        :              |                     |
             |          | (F) Construct RAR     |                     |
             |          |     Using Metadata    |                     |
             |          |        :              |                     |
             |          | (G) Authorization     |                     |
             |          |     Request + RAR     |                     |
             |          |---------------------->|+-------------------+|
             |          |                       ||   Authorization   ||
             |          |<----------------------||     Endpoint      ||
             |          | (H) Authorization Code||                   ||
             |          |        :              |+-------------------+|
             |          |        :              |                     |
             |          | (I) Token Request     |+-------------------+|
             |          |---------------------->||                   ||
             |          |                       ||   Token Endpoint  ||
             |          |<----------------------||                   ||
             |          | (J) Access Token      |+-------------------+|
             |          |        :              +---------------------+
             |          |        :
             |          | (K) API Call with
             |          |     Access Token      +---------------------+
             |          |---------------------->|                     |
             |          |                       |   Resource Server   |
             |          |<----------------------|                     |
             |          | (L) 200 OK + Resource +---------------------+
             |          |
             +----------+
~~~
Figure: Client remediates using metadata of required authorization details types

- (A) The user starts the flow.
- (B) The client calls an API with an access token.
- (C) Resource server returns HTTP 403 Forbidden including a WWW-Authenticate header with error code `insufficient_authorization_details` and in the response body specifies required authorization details types.
- (D-E) The client consumes authorization details types metadata from authorization server's `authorization_details_types_metadata_endpoint`.
- (F-G) The client constructs valid authorization details objects and makes an OAuth + RAR {{RFC9396}} request.
- (H) Authorization server returns authorization code.
- (I-J) The client exchanges authorization code for access token.
- (K) The client makes an API request with the (RAR) access token.
- (L) Resource server validates access token and returns successful response.

## Client remediates using actionable authorization details objects provided by resource server

~~~ ascii-art
                                                +--------------------+
             +----------+ (B) API Request       |                    |
             |          |---------------------->|      Resource      |
(A) User +---|          |                       |       Server       |
   Starts|   |          |<----------------------|                    |
   Flow  +-->|  Client  | (C) 403 Forbidden     +--------------------+
             |          |     WWW-Authenticate: Bearer
             |          |     error="insufficient_authorization_details"
             |          |        +
             |          |     HTTP body provides authorization_details
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
- (C) Resource server returns HTTP 403 Forbidden including a WWW-Authenticate header with error code `insufficient_authorization_details` and in the response body includes the **required authorization details objects**.
- (D) The client uses the obtained authorization details objects in a new OAuth + RAR {{RFC9396}} request.
- (E) Authorization server returns authorization code.
- (F-G) The client exchanges authorization code for access token.
- (H) The client makes an API request with the (RAR) access token.
- (I) Resource server validates access token and returns successful response.

# Authorization Details Types Metadata Endpoint

The following authorization server metadata {{RFC8414}} parameter is introduced to indicate the server's support for Authorization Details Types Metadata:

"authorization_details_types_metadata_endpoint":
:    OPTIONAL.  The URL of the Authorization Details Types Metadata endpoint.

## Authorization Details Types Metadata Endpoint Response

The Authorization Details Types Metadata endpoint is called with HTTP GET and responds with content type `application/json` and JSON body whose each attribute is an authorization details type identifier.

Each identifier is an object describing a single authorization details type.

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

# Resource Server Error Signaling of insufficient authorization_details

This document defines a new error code in the OAuth 2.0 WWW-Authenticate Error Code Registry, `insufficient_authorization_details`, which resource servers SHALL return using the `WWW-Authenticate` header, to signal that access is denied due to missing or insufficient authorization details.

Example HTTP response:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details"

## RECOMMENDED authorization_details_types_required in response body

It is RECOMMENDED that resource servers return, alongside the `insufficient_authorization_details` error, an HTTP body of content type `application/json` containing the new attribute: `authorization_details_types_required`. This attribute specifies the required authorization details **types** satisfying resource's RAR {{RFC9396}} requirements.

"authorization_details_types_required":
:    RECOMMENDED.  a JSON object that conforms to the syntax described in {{syntax}} for a *required types expression*.

Note: It is RECOMMENDED that *authorization_details_types_required* specifies ALL acceptable authorization details types combinations, from ALL supported authorization servers.

Example resource server response with RECOMMENDED `authorization_details_types_required`:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details"
    Content-Type: application/json

    {
        "authorization_details_types_required": {
            "oneOf": ["payment_initiation", "payment_approval",
                      "beneficiary_designation"]
        }
    }


### Required types expression syntax {#syntax}

The following JSON syntax defines a **required types expression** that describes permitted combinations of required *authorization_details* types. This expression allows selection operators (oneOf, allOf) and boolean composition (and, or) to be combined in a predictable manner.

A **required types expression** is a JSON object whose top-level claims MUST contain **exactly** one of the following attributes:

* and
* or
* oneOf
* allOf

Attribute definition:

"and":
:    OPTIONAL.  a non-empty JSON array of *required types expressions*. When **and** is specified, the expression is satisfied if **all** contained expressions are satisfied.

"or":
:    OPTIONAL.  a non-empty JSON array of *required types expressions*. When **or** is specified, the expression is satisfied if **at least one** contained expression is satisfied.

"oneOf":
:    OPTIONAL.  a non-empty JSON array of strings identifying `authorization_details` types. When **oneOf** is specified, the expression is satisfied if **exactly one** of the listed types is present.

"allOf":
:    OPTIONAL.  a non-empty JSON array of strings identifying `authorization_details` types. When **allOf** is specified, the expression is satisfied if **all** of the listed types are present.

### Required types expression examples

#### Example expression using "and" operator

Specifies that the selection MUST include `a` and `b`, **and** one of `c` **or** `d`.

    {
      "and": [
        { "allOf": ["a", "b"] },
        { "oneOf": ["c", "d"] }
      ]
    }

#### Example expression using "or" operator

Specifies that the selection MUST include **either** `c` **and** `d`, **or** one of `a` or `b`.

    {
      "or": [
        { "allOf": ["c", "d"] },
        { "oneOf": ["a", "b"] }
      ]
    }

## OPTIONAL actionable authorization_details in response body

Resource server MAY include, alongside the `authorization_details_types_required` attribute, actionable authorization details objects capable of satisfying the resource's requirements.

Note:

* Resource servers SHOULD provide `authorization_details` objects whose audience is the authorization server that issued the access token's used for the current (failing) request. This simplifies client and end-user interaction, allowing them to continue with the same authorization server previously used.

HTTP response body definition:

"authorization_details":
: OPTIONAL. Array of authorization details objects, matching the format specified in RAR {{RFC9396}} for the `authorization_details` request parameter.

"authorization_hint":
: RECOMMENDED. String serving as a stable reference, enabling the client to select existing access tokens linked to authorization details objects without having to understand RAR object semantics. Its value SHALL be identical for semantically equal `authorization_details` and it SHALL NOT be returned in case tokens resulting from provided `authorization_details` are single-use only.

Clients MAY use the provided `authorization_details` in a subsequent OAuth request to obtain an access token satisfying the resource's requirements.

Example resource server response with OPTIONAL `authorization_details`:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details"
    Content-Type: application/json
    Cache-Control: no-store

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
      }],
      "authorization_hint": "Yb7q3AC5d"
    }

# Handling large RAR objects when issuing access tokens

RAR {{RFC9396}} section 9 instructs that authorization servers MUST provide approved RAR objects to resource servers for enforcement. The authorization server MAY add the `authorization_details` attribute to access tokens in JSON Web Token (JWT) format or to token introspection responses.

Including large RAR objects in JWT access tokens may cause interoperability loss due to token sizes exceeding header size restrictions.

Authorization servers SHOULD support a configurable **maximum approved RAR objects size threshold** (in bytes). If the size exceeds this threshold, JWT access tokens SHALL NOT include the `authorization_details` claim; instead, approved authorization details will be accessed via token introspection {{RFC7662}}.

# Processing Rules

## Client Processing Rules

* When receiving error `insufficient_authorization_details`, if response body contains an *authorization_hint* claim that matches a valid token in client's possession, client SHOULD retry calling the failing endpoint using the matching token.
* If the response body contains an *authorization_details* claim, the client MAY include it in a subsequent OAuth request to obtain a token, which it MAY then use to retry the failing endpoint.
* Otherwise, the client MAY consult metadata:
    * Fetch resource metadata to discover accepted authorization servers and required **authorization_details types**.
    * Fetch authorization server metadata to discover `authorization_details_types_supported`.
    * Fetch authorization server's `authorization_details_types_metadata_endpoint` to
    obtain authorization details type metadata and schemas.
    * Locate schema or retrieve schema_uri.
* Construct authorization details conforming to the schema and include in subsequent OAuth request to obtain a token with which it MAY retry calling the failing endpoint.

## Resource Server Processing Rules

* Verify access tokens against required authorization details.
* When a resource server receives a JWT access token that does not contain an `authorization_details` claim, and authorization details are required for the requested resource, the resource server SHOULD attempt to obtain the authorization details using token introspection {{RFC7662}}, if introspection is available.
* If authorization details are missing or insufficient, the resource server MUST return HTTP 403 with WWW-Authenticate: Bearer error="insufficient_authorization_details".
* It is RECOMMENDED to also provide an HTTP body with the `authorization_details_types_required` attribute, specifying the required authorization details types to satisfy the resource's requirement. This attribute applies to ALL supported authorization servers.
* Resource server MAY also provide in the HTTP body, OPTIONAL actionable `authorization_details` objects, whose audience is the authorization server that issued the current access token, which MAY be used directly for remediation in a subsequent OAuth request.

# Security Considerations {#security-considerations}

## Cacheability and Intermediaries

HTTP 403 responses with response bodies may be cached or replayed in unexpected contexts.
Recommended mitigation is resource servers SHALL use `Cache-Control: no-store` response header.

## Confidentiality of resource server provided authorization_details

Resource server providing actionable `authorization_details` SHOULD NOT include sensitive data within them. This is consistent with RAR {{RFC9396}} `authorization_details` OAuth request parameter, representing **request** semantics.

Confidentiality-preserving `authorization_details` types SHOULD NOT include sensitive data. Instead, end-user SHALL provide such information when interacting with the authorization server.

Alternatively, `authorization_details` MAY refer to specific end-user resources using opaque reference handles (e.g "account_1a" instead of using explicit IBAN).

# IANA Considerations

## OAuth 2.0 WWW-Authenticate Error Code Registry

| Error Code | Description |
|------------|-------------|
| insufficient_authorization_details | The request is missing required authorization details or the provided authorization details are insufficient. |

## OAuth Metadata Attribute Registration

The metadata attribute `authorization_details_types_metadata_endpoint` is defined for OAuth 2.0 authorization server metadata as a URL.

--- back

# Examples

This section provides non-normative examples of how this specification may be used to support specific use cases.

## Authorization Server Metadata Examples {#metadata-examples}

### Example authorization_details_types_metadata_endpoint response with Payment Initiation

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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

### Example authorization_details_types_metadata_endpoint response for the Norwegian Health Sector (HelseID)

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "authorization_details_types_metadata": {
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
    }

## Protected Resource Metadata Examples

### Example Protected Resource Metadata response of payments resource

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "resource": "https://resource.example.com/payments",
        "authorization_servers":
            ["https://as1.example.com",
            "https://as2.example.net"],
        "bearer_methods_supported": ["header"],
        "scopes_supported": ["payment"],
        "resource_documentation":
            "https://resource.example.com/docs/payments.html",
        "authorization_details_types_required": {
            "oneOf": ["payment_initiation", "payment_approval",
                      "beneficiary_designation"]
        }
    }


### Example Protected Resource Metadata response from the Norwegian Health Sector (HelseID)

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "resource": "https://health-api.nhn.no/health-information",
        "authorization_servers": ["https://helseid-sts.nhn.no"],
        "bearer_methods_supported": ["header"],
        "scopes_supported":
            ["nhn:health-api/read", "nhn:health-api/write"],
        "resource_documentation": "https://utviklerportal.nhn.no",
        "authorization_details_types_required": {
            "allOf": ["helseid_authorization",
                      "nhn:tillitsrammeverk:parameters"]
        }
    }


## Payment initiation with RAR error signaling

### Client initiates API request

Client uses access token obtained at login to call payment initiation API

    POST /payments HTTP/1.1
    Host: resource.example.com
    Content-Type: application/json
    Authorization: Bearer eyj... (access token from login)

    {
        "type": "payment_initiation",
        "locations": [
            "https://resource.example.com/payments"
        ],
        "instructed_amount": {
            "currency": "EUR",
            "amount": "123.50"
        },
        "creditor_name": "Merchant A",
        "creditor_account": {
            "bic": "ABCIDEFFXXX",
            "iban": "DE02100100109307118603"
        }
    }

### Resource server signals insufficient_authorization_details with actionable RAR object

Resource server requires payment approval and responds with:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details"
    Content-Type: application/json
    Cache-Control: no-store

    {
        "authorization_details": [{
          "type": "payment_initiation",
          "locations": [
              "https://example.com/payments"
          ],
          "instructed_amount": {
              "currency": "EUR",
              "amount": "123.50"
          },
          "creditor_name": "Merchant A",
          "creditor_account": {
              "bic": "ABCIDEFFXXX",
              "iban": "DE02100100109307118603"
          },
          "interaction_id": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
          "risk_profile": "B-71"
        }
      ]
    }

Note: the resource server has added the ephemeral attributes `interaction_id` and `risk_profile`.

### Client initiates OAuth flow using the provided authorization_details object

After user approves the request, client obtains an access token representing the approved payment

### Client re-attempts API request

    POST /payments HTTP/1.1
    Host: resource.example.com
    Content-Type: application/json
    Authorization: Bearer eyj... (payment approval access token)

    {
        "type": "payment_initiation",
        "locations": [
            "https://resource.example.com/payments"
        ],
        "instructed_amount": {
            "currency": "EUR",
            "amount": "123.50"
        },
        "creditor_name": "Merchant A",
        "creditor_account": {
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

-03

* Added authorization_hint to guide client on token selection and updated client processing rules accordingly
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

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that helped shape the final specification: Rune Grimstad.
