# DFDS REST API Standards
This is the DFDS API Standards document.

To avoid having to read a bricks' worth of text just to build an API, this document is divided into two main sections

1. A 'quick-start' list of the main points that DFDS is aligning on, including a full example
2. A slightly more text-heavy theory sections along with references to further reading

[[toc]]

## Quick-start

DFDS has chosen to use the [JSON API 1.1 standard](https://jsonapi.org/format/1.1/#errors) as a basis for style, with a number of clarifications and opinionated choices.

### Format

Use OpenAPI 3 as API definition format. OpenAPI 2 is also fine if your tooling is not OpenAPI 3 ready. Always use git to keep the definition versioned. The git version is always the authoritative version.

### Versioning

All APIs must be versioned. The version must in the header, and be a simple positive whole number prefixed with `v`. Higher numbers are newer versions. You must bump the version if there are any breaking changes between releases.

A breaking change is defined as removing or renaming any fields or resources, or changing the meaning of resource or field.

```yaml
openapi: "3.0.0"
info:
  version: 2
  title: DFDS Petstore
servers:
  - url: http://petstore.dfds.cloud/v2
paths:
 /catsanddogs:
```

### Naming convention

It is much easier to read and understand an API if the resources and parameters are clearly and concisely named. A few pointers:

1. Urls are lowercase. Use hyphens (-) to improve readability
1. Resources are nouns (`/users, /ships`) and no verbs (no `/add_shipment`)
    1. Actions upon a resource, if relevant, can be verbs, i.e. `/users/{:id}/activate`
1. Use subresources for relations, i.e. `/users/{:id}/bookings`
1. Do not use compound objects invented for convenience, so avoid e.g. `listOfBookingsWithShipNoAndDestinationAddress`
1. Prefer filter query parameters over `/search` endpoints. Filters work on structured data, search functions on unstructured

### Response body

For a response where a single object is returned, follow this format

```json
{
    "data": { object },
    "meta": { ... }
}
```

To return a list:

```json
{
    "items": [
        { object },
        { object },
        ...
    ],
    "meta": { ... }
}
```

The meta object can contain (varies with type of response)

```json
    "meta": {
        "lastResultToken": "<optional: a black box string>",
        "nextResultToken": "<for lists only, not optional: a black box string>",
        "ETag": "<optional>",
        "lastModified": "<optional, use yyyy-MM-dd'T'HH:mm:ssZ if set>"
    }
```

Other data may be added to the meta block as required.

### HTTP Methods

See https://www.restapitutorial.com/lessons/httpmethods.html. Ensure that GETs have no side-effects. Example:

```yaml
  /catsanddogs:
    get:
      summary: List all best friends
      responses:
        '200': ...
    post:
      summary: Create a best friend(??)
  /catsanddogs/{friendId}:
    get:
      summary: Info for a specific furry friend
      parameters:
        - name: friendId
          in: path
          required: true
    put:
        summary: Change something in your best friend, you monster
    delete:
        summary: Remove the best friend
        responses:
            '403': ...
```

### HTTP status codes and error responses

See https://www.restapitutorial.com/httpstatuscodes.html.

If you are returning a response in the 4xx range, include an error object if possible

```json
{
    "error": {
        "error": "<non-optional: unique camelcase error string>",
        "errorDescription": "<optional: human readable text in english>",
        "field": "<optional: useful for validation errors where the field can be referred to>"
    }
}
```

### Headers

There are many possible headers. Avoid carrying application state in the headers and use `X-` as prefix for custom headers. If you use any of these headers, stick to these formats:

<dl>
    <dt>
        <strong>X-Api-Key</strong>
    </dt>
    <dd>
        Use for simple API authentication/authorization
    </dd>
    <dt>
        <strong>X-CorrelationId</strong>
    </dt>
    <dd>
        Set on inbound request, and then downstream and return responses should be tagged with the same id. Highly useful for logging and debugging. The CorrelationId should be a randomly generated GUID.
    </dd>
</dl>

Other typically seen HTTP headers can be found here https://www.whitehatsec.com/blog/list-of-http-response-headers/

## A full example

:::danger
todo
:::

## Theory

The strength of REST lies in the separation into logical resources (bookings, shipments, users, customers, items, ...) and using (relatively) well defined terms to retrieve (GET), update (PUT, PATCH), create (POST) and delete (DELETE) them, so we get a reasonably standardized form

````text
    GET /users/{:user_id}/addresses/{:address_type}/city?format=UPCASE
````

In this contrived example, this would retrieve the uppercased city part of e.g. the home address of a specific user based on their unique user id. In the real world, it is also necessary to specify a protocol, server address and possibly port number. These parts of the invocation are always excluded.

### REST API Maturity levels
:::danger
todo
:::

### In-depth on pagination

:::danger
todo
:::

### In-depth on versioning

1. REST API services MUST Support URL versioning
    1. They MAY support Header and/or Accepts format additionally provided they follow the form specified above
    1. MUST NOT use subdomains or query parameters solely for this purpose
    1. The service MUST NOT offer further mechanisms to versioning
1. The version MUST always be a positive whole number, digits-only, monotonically increasing, prefixed with the letter `v`, e.g. "`v1`, "`v18`"
    1. It MAY additionally provide a date-based subselection as a date-based format, provided that the format is YYYYMMDD gregorian as that format would not break the parent rule (so `/api/v1/20190618/users/{:id}`)
1. MUST increment upon any breaking changes
    1. MAY chose to increment upon non-breaking but otherwise major changes
1. If a user requests a deprecated version, or does not specify a version at all, the service MAY chose to handle it gracefully
    1. Fallback to oldest still supported version if no version provided. If your service offers v2, v3, and v4, and a consumer issues `GET /api/users{:id}`, the v2 version must be delivered. In this case, the service MUST indicate via return headers which version was returned. If no version is provided, and the oldest supported does not offer the requested operation, it is a standard 404 situation
    1. If a deprecated version is requested, the service MAY issue a HTTP 301 redirect to the oldest stable. It does not need to preserve the requested route
    1. If none of the above two options are suitable, the service MUST return a HTTP 404 response with a suitable error message

### Exemptions

- Application and transport protocols
- API authorization and authentication
- HATEOAS style hyperlinking
- Other-than-HTTP communication paradigms, such as GraphQL and event-driven systems
- Client behaviour restrictions
- GraphQL

