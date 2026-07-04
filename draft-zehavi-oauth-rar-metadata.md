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

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} standardizes the exchange and processing of authorization details but does not define metadata for describing authorization details types.

In addition, no interoperable guidance is offered to clients, to remediate failures by resource servers due to insufficient authorization details.

This document addresses this interoperability challenge, allowing clients to dynamically discover metadata instead of relying on out-of-band agreements.
It also standardizes failure signaling and interoperable remediation when insufficient authorization details are the cause of failure.

--- middle

# Introduction

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} allows OAuth clients to request detailed and structured authorization, enabling advanced authorization models across domains such as banking and healthcare.

However, RAR {{RFC9396}} does not specify how clients discover metadata describing valid authorization details objects. Such metadata and documentation are obtained out-of-band.

This document defines:

* A new authorization server endpoint: `authorization_details_types_metadata_endpoint`, providing authorization details type metadata, including documentation and JSON Schema definitions {{JSON.Schema}}.
* A new normative OAuth 2.0 WWW-Authenticate Error Code, for resource servers to indicate `insufficient_authorization` as the cause of the error.
* A new OAuth 2.0 WWW-Authenticate response parameter, `authorization_remediation`, which contains actionable authorization details objects, to be used directly for remediation in a follow-up OAuth request.
* Authorization server considerations for when RAR authorization details objects better be omitted from JWT access tokens, provided instead through token instrospection.

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

* The bearer authentication error code `insufficient_authorization` for use in the `WWW-Authenticate` header. Resource servers SHOULD return `insufficient_authorization` to signal that access is denied due to missing or insufficient authorization details. The `insufficient_authorization` error is intended whenever the authorization details associated with the token do not satisfy the resource's requirements. The `insufficient_authorization` error MUST NOT be used when the access token is missing, expired, malformed, or otherwise invalid. Existing Bearer authentication error codes are used for those cases.
* The `authorization_remediation` error parameter, which contains a base64url-encoded JSON object used to guide the client on remediating the error. Its attributes are:

   * `authorization_details`: REQUIRED. Array of actionable authorization details objects, matching the format specified in RAR {{RFC9396}} for the `authorization_details` request parameter, whose contents satisfy the resource's requirements and remediates the failure. The provided `authorization_details` are intended to be interoperable with all OAuth specifications and usable in any grant flow supporting RAR.
   * `authorization_reference`: RECOMMENDED. An opaque string generated by the resource server that enables the client to select an existing access token associated with equivalent authorization details without requiring the client to understand the semantics of the authorization details object. Clients MUST treat this value as opaque and MUST NOT attempt to interpret or derive meaning from it. The resource server SHOULD generate the authorization_reference by canonicalizing and hashing the authorization_details object or an equivalent stable representation, so that the same or semantically equivalent authorization details produce the same authorization_reference value. This stability allows clients to reliably match tokens to authorization requirements and avoid requesting new tokens when a matching token is already in their possession. The value MUST NOT reveal any sensitive or private information. The resource server SHALL NOT include this attribute when tokens issued for the provided authorization_details are intended for single-use only.

The `error_description` parameter MAY be included to provide a human-readable description.

Example HTTP response from a direct debit resource:

    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer error="insufficient_authorization",
    error_description="Additional authorization is required",
    authorization_remediation=eyJhdXRob3JpemF0aW9uX2RldGFpbHMiOlt7InR5cGUiOiJkaXJlY3RfZGViaXRfbWFuZG...

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
    authorization_remediation=eyJhdXRob3JpemF0aW9uX2RldGFpbHMiOlt7InR5cGUiOiJkaXJlY3RfZGViaXRfbWFuZG...

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

## Authorization Details Types Metadata Endpoint Response

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

* When receiving an `insufficient_authorization` error, if the `authorization_remediation` parameter contains an `authorization_reference` attribute that matches a valid token in the client's possession, the client MAY retry the failing request using the matching token.
* If the `authorization_remediation` parameter contains an `authorization_details` attribute, the client MAY include it in a subsequent OAuth request to obtain a token for retrying the failing endpoint.
* If the authorization server used so far by the client does not support the required authorization details types, the client MAY use Protected Resource Metadata {{RFC9728}} to discover additional `authorization_servers` supported by the resource, and attempt remediation through them.

## Resource Server Processing Rules

* Verify access token validity.
* Verify required authorization details are provided, by JWT token contents or through token introspection {{RFC7662}}.
* If authorization details are missing or insufficient, return HTTP 401 with WWW-Authenticate: Bearer error="insufficient_authorization" including the `authorization_remediation` parameter which provides actionable `authorization_details` objects constructed from the failing request's input.

# Security Considerations {#security-considerations}

## Cacheability and Intermediaries

HTTP 401 responses with response bodies may be cached or replayed in unexpected contexts.
Resource servers SHOULD mitigate by including the `Cache-Control: no-store` response header.

## Confidentiality of resource server provided authorization_details

Resource server providing actionable `authorization_details` SHOULD NOT include sensitive data in those objects. This is consistent with RAR {{RFC9396}} `authorization_details` OAuth request parameter, representing **request** semantics.

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

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that helped shape the final specification: Rune Grimstad.
