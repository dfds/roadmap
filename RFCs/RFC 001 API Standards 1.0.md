# RFC API Conventions DFDS 0.0

Status: Draft Proposal

**The drafting period runs from 30th of April to 15h of June 2019. Final comment and corrections are accepted between 16th of June to 31st of June 2019.**

--------

## Abstract

This document is the DFDS API Conventions and Requirements specification. All API enabled services and components, regardless of intended audience, authored within DFDS, must follow this specification.

The intended outcome is to eventually converge all DFDS APIs to have a similar look and feel.

This is not an IETF-style RFC. It is a draft proposal and follows only the most basic of RFC conventions and processes.

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

Additionally, the use of SHOULD is not synomyous with MAY. SHOULD indicates that in almost all circumstances, the implementor SHOULD follow the specification except in carefully considered circumstances. MAY means that an item is truly optional.

### Limitations and timetable

This RFC covers mainly topics on API specification, inter-service communication and inter-service message format structure. Specifically excluded for v1.0 are these topics:

- Application and transport protocols
- API authorization and authentication
- HATEOAS style hyperlinking
- Other-than-HTTP communication paradigms, such as GraphQL and event-driven systems
- Client behaviour restrictions

Some topics may be marked with [TODO]. These must be fleshed out during the drafting period, alternatively removed if consensus is that the topic should not be covered.

## Context

The strength of REST lies in the separation into logical resources (bookings, shipments, users, customers, items, ...) and using (relatively) well defined terms to retrieve (GET), update (PUT, PATCH), create (POST) and delete (DELETE) them. This document will often cite examples such as:

````text
    GET /users/{:user_id}/addresses/{:address_type}/city?format=UPCASE
````

In this contrived example, this would retrieve the uppercased city part of e.g. the home address of a specific user based on their unique user id. In the real world, it is also necessary to specify a protocol, server address and possibly port number. These parts of the invocation are always excluded.

### Terms and possible architectures

[todo: add list of common terms]

[todo: add sample architecture diagrams such as for BFF pattern]

--------------

## Contents

### API authentication

While we are not generally ready to standardise on authentication yet, it is very common to need at least some basic API security. Having even a basic API key mechanism in place gives a lot of flexibility in case of emergencies such as attacks, need for more finely grained rate-limiting, or auditing.

If the service does have API key type authentication, it must follow these rules:

1. The API key SHOULD chiefly be accepted only in the HTTP request header and the key MUST be named `X-Api-Key`
1. Some transport mechanisms, such as JavaScript websockets, does not support custom headers. In these cases, sending the API key in the URL can be acceptable. In these cases, a query parameter MUST be used.

### API format and publishing

To ensure that as near everybody as possible (man and machine) can read the API definitions (the description of the language that each service speaks), we want to standardise on the core formats. The exact tooling used can vary and evolve.

We have chosen to standardise on OpenAPI v2 (more commonly known as Swagger) due to ubiquity and support. OpenAPI v3 is available but tooling support is not universal.

1. API definitions MUST be defined via OpenAPI 2.0
1. API definitions MUST be versioned in git (or similarly revision controlled repository) and the version in git MUST be the authoritative version
1. API definitions MUST be stored either UTF8 or UTF16, as either YAML or JSON, using Unix line ending format (git will normally handle the last part for you)
1. Services MAY expose a API UI endpoint (e.g. `https://service_url/swagger` or similar) and indeed it is RECOMMENDED to do so
1. Services SHOULD NOT generate the API definition from code and instead validate the code using the contract. This is to avoid unintentionally breaking the API by for example changing the variable type of a class member.

#### Definition standards

OpenAPI 2.0 examples are openly available at <https://github.com/OAI/OpenAPI-Specification/tree/master/examples/v2.0>. Taking the preamble <https://github.com/OAI/OpenAPI-Specification/blob/master/examples/v2.0/yaml/petstore-minimal.yaml> as example

```yaml
---
  swagger: "2.0"
  info:
    version: "1.0.0"
    title: "Swagger Petstore"
    description: "A sample API that uses a petstore as an example to demonstrate features in the swagger-2.0 specification"
    termsOfService: "http://swagger.io/terms/"
    contact:
      name: "Swagger API Team"
    license:
      name: "MIT"
  host: "petstore.swagger.io"
  basePath: "/api"
  schemes:
    - "https"
  consumes:
    - "application/json"
  produces:
    - "application/json"
  ```

`host`, `basePath` and `termsOfService` are OPTIONAL. All other fields MUST be present and have meaningful content.

### Resources and grammar

Regarding method calls, as mentioned above, HTTP does have a wide variety of `methods` with an at-least somewhat clear meaning. It is also the case that some frameworks interpret the HTTP specification somewhat liberally.

1. The API MUST offer the most appropriate HTTP method for each operation
    1. It MAY be more flexible and allow POST calls to update a resource
1. A GET call MUST NOT result in altering or otherwise cause any side-effects to the state of the resource
1. A GET call MUST NOT be used for state changes, i.e. `GET /ships/{:id}/sink` is not allowed. `PUT /ships/{:id}/sink` is fine
1. It MAY offer support for other HTTP methods such as INFO, HEAD, OPTIONS, TRACE, CONNECT
1. It is NOT RECOMMENDED to use situational or compound resources. While it can be tempting to shift the load from the consumer in the cases where they need to fetch multiple resources, or want an abridged variant of the data, it is generally more futureproof and incurs less technical debt to offer either deep-loading of subresources or field-level verbosity instead

#### Naming convention

It is much easier to read and understand an API if the resources and parameters are clearly and comprehensively named:

1. Urls MUST be lowercase
1. Urls MUST use hyphens (-) to improve readability
1. All API resources MUST be named with nouns (`/users, /ships` is fine) and MUST NOT use verbs (`/add_shipment` is not allowed)
    1. Actions upon a resource, if relevant, SHOULD be verbs, i.e. `/users/{:id}/activate`
1. The API SHOULD NOT mix singular and plural nouns. It is much more readable to stick with plural nouns for all resources
1. The API SHOULD use subresources for relations, i.e. `/users/{:id}/bookings
1. The API SHOULD NOT use CRUD function names in URIs

#### Query Parameters

[todo: urgent]

1. Paging
1. Filtering
1. Sorting

### Responses
A typical response consists of a set of headers and the body. All responses must be wrapped in a return envelope, both to carry additional metadata and to standardise on the return format.

[todo: note that the meta field is intended to carry paging, data version if any. This definition is not yet complete]

For a response where a single object is returned, follow this format

```json
{
    data: { object },
    meta: { <todo> }
}
```

To return a list:

```json
{
    items: [
        { object },
        { object }
    ],
    meta: { <todo> }
}
```

To return an error that needs additional explanation beyond what the HTTP status code provides:

```json
{
  "error": {
    "status": 409, # MUST be provided
    "error": "EMAIL_EXISTS", # MUST be provided and MUST be unique
    "description": "The provided email address is not available." # OPTIONAL
  }
}
```

#### Typical error codes

This is an non-exhaustive list, obviously.

| Code  | Use when |
|---|---|
| 200 | OK - returned whenever there are nothing wrong and nothing else to add   |
| 201 | Created - use on successful POST of new item  |
| 202 | Request accepted but result may not be immediately or ever available  |
| 301 | Used when a deprecated version is requested  |
| 400 | Used to indicate that client sent a nonsense request  |
| 402 | Not authorised - use when additional authorization is required|
| 403 | Forbidden - CAN be used to state that this is not and will never allowed. Alternatively use 404 |
| 404 | Not found - for when a resource, item, action or version is not available. |
| 409 | Conflict - a resource was posted that would violate data integrity restraints |
| 500 | Server Error, meaning internal error. Additional information MUST NOT be provided in the response in production builds |


See also the status code overview[1]

[todo: urgent]

1. Envelopes
1. Errors

#### Headers for Request and Response

[todo]

#### CORS

[todo]

### Search vs filters

[todo: urgent]

### Graph Resources

[todo]

### API Versioning

Data and format changes all the time and API specifications are much like
contracts. They can be renegotiated but it is typically cumbersome and costly,
particularly if the API is open to all interested parties.

Breaking changes include but is probably not limited to changing or removing any existing route or field that has not been explicitly listed as optional, including changing the meaning or format. That means, that it is for most pragmatic purposes, always allowed to add more contents: fields, resources, query parameters etc.

API versioning are often done by embedding the version in the URL, in the form `https://service/api/v2/users/{:id}`. Other typical approaches are by having an explicit header in the HTTP request, in the form `X-Api-Version: 2`, or by being specific in the `Accept` HTTP request header, in this case by following this form: `Accept: application/vnd.dfds.v2+json`

1. REST API services MUST Support URL versioning
    1. They MAY support Header and/or Accepts format additionally provided they follow the form specified above
    1. MUST NOT use subdomains or query parameters solely for this purpose
    1. The service MUST NOT offer further mechanisms to versioning
1. The version MUST always be a positive whole number, digits-only, monotonically increasing, prefixed with the letter `v`, e.g. "`v1`, "`v18`"
    1. ~~It MAY additionally provide a date-based subselection as a date-based format, provided that the format is YYYYMMDD gregorian as that format would not break the parent rule (so `/api/v1/20190618/users/{:id}`)~~
1. MUST increment upon any breaking changes
    1. MAY chose to increment upon non-breaking but otherwise major changes
1. If a user requests a deprecated version, or does not specify a version at all, the service MAY chose to handle it gracefully
    1. Fallback to oldest still supported version if no version provided. If your service offers v2, v3, and v4, and a consumer issues `GET /api/users{:id}`, the v2 version must be delivered. In this case, the service MUST indicate via return headers which version was returned. If no version is provided, and the oldest supported does not offer the requested operation, it is a standard 404 situation
    1. If a deprecated version is requested, the service MAY issue a HTTP 301 redirect to the oldest stable. It does not need to preserve the requested route
    1. If none of the above two options are suitable, the service MUST return a HTTP 404 response with a suitable error message

### Rate limiting

[todo]

   1. Internal
   1. External

## Inspiration and additional reading

These blogs and resources have been great inspiration

- https://blog.mwaysolutions.com/2014/06/05/10-best-practices-for-better-restful-api/
- https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
- https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/ (note that the conclusion ends up being that URL versioning is what everybody uses)
- https://www.restapitutorial.com/httpstatuscodes.html


[1]: https://www.restapitutorial.com/httpstatuscodes.html