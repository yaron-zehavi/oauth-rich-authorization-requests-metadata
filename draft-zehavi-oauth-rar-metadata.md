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

OAuth 2.0 Rich Authorization Requests (RAR), as defined in {{RFC9396}}, enables fine-grained authorization requests, using structured JSON objects.

While RAR {{RFC9396}} standardizes the exchange and handling of authorization details, it does not define a mechanism for clients to discover how to construct valid authorization details types.

This document defines a machine-readable metadata format for authorization servers to provide authorization details type documentation and JSON Schema {{JSON.Schema}} definitions, as well as interoperable discovery via OAuth Resource Server Metadata {{RFC9728}}.

This document also defines a new WWW-Authenticate normative OAuth error code, `insufficient_authorization_details`, enabling resource servers to indicate inadequate authorization details as the cause of failure.

It also defines an OPTIONAL response body which MAY be returned alongside the `insufficient_authorization_details` error, providing an informative yet actionable authorization details object, which can be used directly in a subsequent OAuth request.

--- middle

# Introduction

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} allows OAuth clients to request structured, fine-grained authorization, beyond the coarse-grained access offered by simple scopes, which has enabled advanced authorization models across many domains, such as open banking & API marketplaces.

However, RAR {{RFC9396}} does not specify how a client learns how to construct syntactically valid authorization details objects. As a result, clients must rely on out-of-band documentation or static ecosystem profiles, limiting interoperability and preventing dynamic client behavior.

This document addresses this gap by:

* Defining a new authorization server endpoint: `authorization_details_types_metadata_endpoint`, providing metadata for authorization details types, including human-readable documentation as well as embedded JSON Schema definitions {{JSON.Schema}}.
* Adding accepted authorization details types to OAuth 2.0 Protected Resource Metadata {{RFC9728}} response, facilitating RAR metadata discovery.
* Defining a standardized error signaling mechanism using the WWW-Authenticate response header, allowing resource servers to specify `insufficient_authorization_details` as the cause of error.
* Defining an OPTIONAL response body, included with an `insufficient_authorization_details` error, providing an informative authorization details object, whose inclusion in a new OAuth request shall result, if approved, in an access token satisfying the endpoint's requirements.

The OPTIONAL providing of actionable authorization details objects by resource servers enables:

* High interoperability and simplification by relieving clients from having to figure out how to construct valid authorization details objects, instead providing them with required authorization_details object, to be included in a subsequent OAuth request.
* Support for including ephemeral, interaction-specific details originating from the resource domain, in the authorization details object, such as for example a risk score, a risk profile or an internal interaction identifier. Resource servers MAY use this to guide authorization servers as to the required authentication strength and consent flow.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

There are two main proposed flows:

* Client obtains **metadata** of required authorization details types.
* Client obtains an **actionable authorization details object** from resource server's error response.

## Client obtains metadata of required authorization details types

~~~ ascii-art
                                                +---------------------+
             +----------+ (B) API Request       |                     |
             |          |---------------------->|      Resource       |
(A) User +---|          |                       |       Server        |
   Starts|   |          |<----------------------|                     |
   Flow  +-->|          | (C) 403 Forbidden     +---------------------+
             |          |     WWW-Authenticate: Bearer
             |          |     error="insufficient_authorization_details",
             |          |     resource_metadata="[resource metadata url]"
             |          |           :
             |          |        Resource       +---------------------+
             |          | (D) Metadata Request  |   Resource Server   |
             |          |---------------------->|+-------------------+|
             |          |                       || Resource Metadata ||
             |  Client  |<----------------------||    Endpoint       ||
             |          | (E) Metadata Response |+-------------------+|
             |          |    (Discover also     +---------------------+
             |          |     expected RAR types)
             |          |           :           +---------------------+
             |          |        RAR Types      |    Authorization    |
             |          | (F) Metadata Request  |       Server        |
             |          |---------------------->|+-------------------+|
             |          |                       ||     RAR Types     ||
             |          |<----------------------|| Metadata Endpoint ||
             |          | (G) Metadata Response |+-------------------+|
             |          |           :           |                     |
             |          | (H) Construct RAR     |                     |
             |          |     Using Metadata    |                     |
             |          |        :              |                     |
             |          | (I) Authorization     |                     |
             |          |     Request + RAR     |                     |
             |          |---------------------->|+-------------------+|
             |          |                       ||   Authorization   ||
             |          |<----------------------||     Endpoint      ||
             |          | (J) Authorization Code||                   ||
             |          |        :              |+-------------------+|
             |          |        :              |                     |
             |          | (K) Token Request     |+-------------------+|
             |          |---------------------->||                   ||
             |          |                       ||   Token Endpoint  ||
             |          |<----------------------||                   ||
             |          | (L) Access Token      |+-------------------+|
             |          |        :              +---------------------+
             |          |        :
             |          | (M) API Call with
             |          |     Access Token      +---------------------+
             |          |---------------------->|                     |
             |          |                       |   Resource Server   |
             |          |<----------------------|                     |
             |          | (N) 200 OK + Resource +---------------------+
             |          |
             +----------+
~~~
Figure: Client learns to construct valid authorization details objects from metadata

- (A) The user starts the flow.
- (B) The client calls an API with an access token.
- (C) Resource server returns HTTP 403 forbidden including a WWW-Authenticate header with error code `insufficient_authorization_details` and the resource metadata url (OAuth 2.0 Protected Resource Metadata {{RFC9728}}).
- (D-E) The client discovers expected authorization details types from resource metadata endpoint's response.
- (F-G) The client consumes authorization details type metadata from authorization server's `authorization_details_types_metadata_endpoint`.
- (H-I) The client constructs a valid authorization details object and makes an OAuth + RAR {{RFC9396}} request.
- (J) Authorization server returns authorization code.
- (K-L) The client exchanges authorization code for access token.
- (M) The client makes an API request with the (RAR) access token.
- (N) Resource server validates access token and returns successful response.

## Client obtains authorization details object from resource server's error response

~~~ ascii-art
                                                +--------------------+
             +----------+ (B) API Request       |                    |
             |          |---------------------->|      Resource      |
(A) User +---|          |                       |       Server       |
   Starts|   |          |<----------------------|                    |
   Flow  +-->|  Client  | (C) 403 Forbidden     +--------------------+
             |          |     WWW-Authenticate: Bearer
             |          |     error="insufficient_authorization_details",
             |          |     resource_metadata="[resource metadata url]"
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
- (C) Resource server returns HTTP 403 forbidden including a WWW-Authenticate header with error code `insufficient_authorization_details` and in the response body **includes the authorization details object requiring approval**.
- (D) The client uses the obtained authorization details object in a new OAuth + RAR {{RFC9396}} request.
- (E) Authorization server returns authorization code.
- (G-H) The client exchanges authorization code for access token.
- (I) The client makes an API request with the (RAR) access token.
- (J) Resource server validates access token and returns successful response.

# OAuth 2.0 Protected Resource Metadata {{RFC9728}}

This document specifies a new OPTIONAL metadata attribute: `authorization_details_types_supported`, to be included in the response of OAuth Protected Resource Metadata {{RFC9728}}.

"authorization_details_types_supported":
:    OPTIONAL.  a JSON object that conforms to the syntax described in {{syntax}} for a *required types expression*.

The following is a non-normative example response with the added `authorization_details_types_supported` attribute:

    GET /.well-known/oauth-protected-resource/payments HTTP/1.1
    Host: resource.example.com

    ..
    ..

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "resource":
      "https://resource.example.com/payments",
      "authorization_servers":
        ["https://as1.example.com",
         "https://as2.example.net"],
      "bearer_methods_supported":
        ["header", "body"],
      "scopes_supported": ["payment"],
      "resource_documentation":
        "https://resource.example.com/docs/payments.html",
      "authorization_details_types_supported":
        "oneOf": ["payment_initiation", "payment_approval",
                  "beneficiary_designation"]
    }

Note: When resource servers accept access tokens *from several authorization servers*, interoperability is maintained and confusion is prevented, because clients can discover which authorization details types each authorization server supports.

## Required types expression syntax {#syntax}

The following JSON syntax defines a **required types expression** to declaratively describe permitted combinations of required *authorization_details* types. This expression allows selection operators (oneOf, allOf, constraints) and boolean composition (and, or) to be combined in a predictable manner.

A **required types expression** is a JSON object that MUST contain **exactly** one of the following attributes:

* and
* or
* oneOf
* allOf
* constraints


Attributes definition:

"and":
:    OPTIONAL.  a non-empty JSON array of *required types expressions*. When **and** is specified, the expression is satisfied if **all** contained expressions are satisfied.

"or":
:    OPTIONAL.  a non-empty JSON array of *required types expressions*. When **or** is specified, the expression is satisfied if **at least one** contained expression is satisfied.

"oneOf":
:    OPTIONAL.  a non-empty JSON array of strings identifying authorization_details types. When **oneOf** is specified, the expression is satisfied if **exactly one** of the listed types is present.

"allOf":
:    OPTIONAL.  a non-empty JSON array of strings identifying authorization_details types. When **allOf** is specified, the expression is satisfied if **all** of the listed types are present.

"constraints":
:    OPTIONAL.  a JSON object defining cardinality and exclusion constraints over a set of authorization_details types. The object MUST contain the **types** attribute and MAY contain the attributes **min**, **max**, **exact**, and **forbidden**. Constraints attributes definition:

     "types":
     :    REQUIRED.  a non-empty JSON array of strings
          identifying the authorization_details types
          to which the constraints apply.

     "min":
     :    OPTIONAL.  a non-negative integer indicating
          the minimum number of authorization_details
          types from `types` that MUST be present.
          This attribute MUST NOT be used together
          with the **exact** attribute.

     "max":
     :    OPTIONAL.  a non-negative integer indicating
          the maximum number of authorization_details
          types from `types` that MAY be present.
          This attribute MUST NOT be used together
          with the **exact** attribute.

     "exact":
     :    OPTIONAL.  a non-negative integer indicating
          the exact number of authorization_details
          types from `types` that MUST be present.
          This attribute MUST NOT be used together
          with the **min** or **max** attributes.

     "forbidden":
     :    OPTIONAL.  a non-empty JSON array, whose each
          element is an array of authorization_details
          types identifiers, representing a combination
          that MUST NOT be present together.

## Required types expression examples

### Example expression using "and" operator

Specifies that the selection MUST include a and b, **and** one of c **or** d.

    {
      "required_types": {
        "and": [
          { "allOf": ["a", "b"] },
          { "oneOf": ["c", "d"] }
        ]
      }
    }

Specifies that the selection MUST include one of a or b, **and** exactly two of [c,d,e], but the combination of d and e together is forbidden.

    {
      "required_types": {
        "and": [
          { "oneOf": ["a", "b"] },
          {
            "constraints": {
              "types": ["c", "d", "e"],
              "exact": 2,
              "forbidden": [["d", "e"]]
            }
          }
        ]
      }
    }

### Example expression using "or" operator

Specifies that the selection MUST include **either** c **and** d, **or** one of a or b.

    {
      "required_types": {
        "or": [
          { "allOf": ["c", "d"] },
          { "oneOf": ["a", "b"] }
        ]
      }
    }

### Example expression using "constraints" operator

Specifies that at least two of {a,b,c} MUST be present, but the combination of a and c together is forbidden.

    {
      "required_types": {
        "constraints": {
          "types": ["a","b","c"],
          "min": 2,
          "forbidden": [ ["a","c"] ]
        }
      }
    }

Specifies that exactly two of {a,b,c} MUST be present.

    {
      "required_types": {
        "constraints": {
          "types": ["a","b","c"],
          "exact": 2
        }
      }
    }


# Authorization Details Types Metadata Endpoint

The following authorization server metadata {{RFC8414}} parameter is introduced to signal the server's support for Authorization Details Types Metadata:

"authorization_details_types_metadata_endpoint":
:    OPTIONAL.  The URL of the Authorization Details Types Metadata endpoint.

## Authorization Details Types Metadata Endpoint Response

The Authorization Details Types Metadata endpoint's response is a JSON document with the key `authorization_details_types_metadata` whose attributes are authorization details type identifiers.

Each identifier is an object describing a single authorization details type.

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

See Examples {{metadata-examples}} for non-normative response example.

# Resource Server Error Signaling of Inadequate authorization_details

This document defines a new normative OAuth error code, `insufficient_authorization_details`, which resource servers SHALL return using the `WWW-Authenticate` header, to signal access is denied due to missing or insufficient authorization details.

Example HTTP response:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details",
        resource_metadata="https://resource.example.com/
        .well-known/oauth-protected-resource/payments"

## OPTIONAL authorization_details in response body

Resource server MAY provide alongside the `insufficient_authorization_details` error, an HTTP response body of content type application/json, containing the required authorization details to satisfy the currently failing request.

Note:

* Authorization details objects provided by a resource server in an error response are intended for its trusted authorization servers, as advertised by the Resource Server’s metadata endpoint.
* Resource servers SHALL provide authorization_details objects only if **all** trusted authorization servers accept the **authorization details type** used.

HTTP response body definition:

"authorization_details":
: OPTIONAL. Array of authorization details objects, matching the format specified in RAR {{RFC9396}} for the `authorization_details` request parameter.

Clients MAY use the provided `authorization_details` in a subsequent OAuth request to obtain an access token satisfying the resource's requirements.

Example resource server response with OPTIONAL authorization_details:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details",
        resource_metadata="https://resource.example.com/
        .well-known/oauth-protected-resource/payments"
    Content-Type: application/json
    Cache-Control: no-store

    {
      "authorization_details": [{
        "type": "payment_initiation",
        "instructedAmount": {
          "currency": "EUR",
          "amount": "100.00"
        },
        "creditorAccount": {
          "iban": "DE02120300000000202051"
        }
      }]
    }

# Processing Rules

## Client Processing Rules

* If encountering error `insufficient_authorization_details`, check if body.authorization_details exists and if provided MAY include in subsequent OAuth request.
* Otherwise consult metadata:
    * Fetch resource metadata to discover accepted authorization servers and supported **authorization_details types**.
    * Fetch authorization server metadata to discover `authorization_details_types_supported`.
    * Fetch authorization server's `authorization_details_types_metadata_endpoint` to obtain metadata and schema
    * Locate schema or retrieve schema_uri.
* Construct authorization details conforming to the schema and include in subsequent OAuth request.

## Resource Server Processing Rules

* Advertise in resource metadata `authorization_details_types_supported`, where relevant.
* Verify access tokens against required authorization details.
* If insufficient, return HTTP 403 with WWW-Authenticate: Bearer error="insufficient_authorization_details".
* OPTIONALLY provide also an HTTP body with an informative actionable authorization_details object.

# Security Considerations {#security-considerations}

## Cacheability and Intermediaries

HTTP 403 responses with response bodies may be cached or replayed in unexpected contexts.
Recommended mitigation is resource servers SHALL use `Cache-Control: no-store` response header.

# IANA Considerations

## OAuth 2.0 Bearer Token Error Registry

| Error Code | Description |
|------------|-------------|
| insufficient_authorization_details | The request is missing required authorization details or the provided authorization details are insufficient. |

## OAuth Metadata Attribute Registration

The metadata attribute `authorization_details_types_metadata_endpoint` is defined for OAuth 2.0 authorization server metadata as a URL.
The metadata attribute `authorization_details_types_supported` is defined for OAuth 2.0 protected resource metadata.

--- back

# Examples

This section provides non-normative examples of how this specification may be used to support specific use cases.

## Metadata Examples {#metadata-examples}

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
                            "const": "helseid_autorization",
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
            {
                "$schema": "http://json-schema.org/draft-07/schema#",
                "description": "Complete Trust Framework structure",
                "documentation_uri": "https://utviklerportal.nhn.no/informasjonstjenester/helseid/bruksmoenstre-og-eksempelkode/bruk-av-helseid/docs/tillitsrammeverk/profil_for_tillitsrammeverkmd",
                "type": "object",
                "properties": {
                    "type": {
                      "type": "string",
                      "const": "nhn:tillitsrammeverk:parameters",
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

### Resource server signals insufficient_authorization_details with actionable RAR object

Resource server requires payment approval and responds with:

    HTTP/1.1 403 Forbidden
    WWW-Authenticate: Bearer error="insufficient_authorization_details",
        resource_metadata="https://resource.example.com
        /.well-known/oauth-protected-resource/payments"
    Content-Type: application/json
    Cache-Control: no-store

    {
        "authorization_details": [{
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
    }

Note: the resource server has added the ephemeral attributes `interactionId` and `riskProfile`.

### Client initiates OAuth flow using the provided authorization_details object

After user approves the request, client obtains single-use access token representing the approved payment

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

-02

* Defined the required types expression
* Runes addition of a example from the Norwegian Health Sector (HelseID)

-01

* Authorization details moved to HTTP body and made OPTIONAL
* Metadata pointer from resource metadata url, full authorization details types metadata on authorization server new endpoint

-00

* Document creation


# Acknowledgments
{:numbered="false"}
