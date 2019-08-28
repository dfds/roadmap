# DFDS REST API Standards
This is the DFDS API Standards document.

To avoid having to read a bricks' worth of text just to build an API, this document is divided into two main sections

1. A 'quick-start' list of the main points that DFDS is aligning on, including a full example
2. A slightly more text-heavy theory sections along with references to further reading

## Quick-start
If you are new to REST APIs there is a good tutorial at (https://www.restapitutorial.com).

DFDS has chosen to use the [JSON API 1.1 standard](https://jsonapi.org/format/1.1/#errors) as a **basis** for style, with a number of clarifications and opinionated choices. These are all listed below.

### Format

Use OpenAPI 3 as API definition format. OpenAPI 2 is also fine if your tooling is not OpenAPI 3 ready. Always use git to keep the definition versioned. The git version is always the authoritative version.

### Publishing

Services should **not** generate the API definition from code and instead validate the code using the contract. This is to avoid unintentionally breaking the API by for example changing the variable type of a class member.

### Versioning

All APIs must be versioned. The version must be set as the first element in the URL, and be a single positive whole number prefixed with `v`. You must bump the version if there are any breaking changes between releases.

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
2. Resources are nouns (`/users, /ships`) and no verbs (no `/add_shipment`)
    1. Actions upon a resource, if relevant, can be verbs, i.e. `/users/{:id}/activate`
3. Use subresources for relations, i.e. `/users/{:id}/bookings`
4. Do not use compound objects invented for convenience, so avoid e.g. `listOfBookingsWithShipNoAndDestinationAddress`
5. Prefer filter query parameters over `/search` endpoints. Filters work on structured data, search functions on unstructured

### Response body

For a response where a single object is returned, follow this format

```json
{
    "data": { },
    "meta": { }
}
```

To return a list:

```json
{
    "items": [
        { },
        { },
        ...
    ],
    "meta": { ... }
}
```

All keys are camel case English, in the proper form (ends with 's' when plural). 

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

### Idempotency

Services should act in an idempotent way, such that multiple call to the same resource returns same result.
See https://www.restapitutorial.com/lessons/idempotency.html for more information.

## A full example

```yaml
openapi: "3.0.0"
info:
  version: 2.0.0
  title: Swagger Petstore
  description: A sample API that uses a petstore as an example to demonstrate features in the OpenAPI 3.0 specification
servers:
  - url: http://petstore.dfds.cloud/v2
paths:
  /catsanddogs:
    get:
      summary: List all best friends
      operationId: findPets
      parameters:
        - name: tags
          in: query
          description: tags to filter by
          required: false
          style: form
          schema:
            type: array
            items:
              type: string
        - name: limit
          in: query
          description: maximum number of results to return
          required: false
          schema:
            type: integer
            format: int32
        - name: X-Api-Key
          in: header
          schema:
            type: string
          required: true
      responses:
        '200':
          description: pet response
          content:
            application/json:
              schema:
                properties:
                  items:
                    type: array
                    items:
                      $ref: '#/components/schemas/Pet'
                  meta:
                    $ref: '#/components/schemas/Metadata'
        default:
          description: unexpected error
          content:
            application/json:
              schema:
                properties:
                  error:
                    $ref: '#/components/schemas/Error'
    post:
      summary: Create a best friend(??)
      operationId: addPet
      parameters:
        - name: X-Api-Key
          in: header
          schema:
            type: string
          required: true
      requestBody:
        description: Pet to add to the store
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NewPet'
      responses:
        '200':
          description: pet response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Pet'
        default:
          description: unexpected error
          content:
            application/json:
              schema:
                properties:
                  error:
                    $ref: '#/components/schemas/Error'
  /catsanddogs/{friendId}:
    get:
      summary: Info for a specific furry friend
      operationId: findPetById
      parameters:
        - name: friendId
          in: path
          description: ID of pet to fetch
          required: true
          schema:
            type: integer
            format: int64
        - name: X-Api-Key
          in: header
          schema:
            type: string
          required: true
      responses:
        '200':
          description: pet response
          content:
            application/json:
              schema:
                properties:
                  data:
                    $ref: '#/components/schemas/Pet'
                  meta:
                    $ref: '#/components/schemas/Metadata'
        '404':
          description: Furry friend not found
          content:
            application/json:
              schema:
                properties:
                  error:
                    $ref: '#/components/schemas/Error'
        default:
          description: unexpected error
          content:
            application/json:
              schema:
                properties:
                  error:
                    $ref: '#/components/schemas/Error'
    delete:
      summary: Remove the best friend, you monster
      operationId: deletePet
      parameters:
        - name: friendId
          in: path
          description: ID of pet to delete
          required: true
          schema:
            type: integer
            format: int64
        - name: X-Api-Key
          in: header
          schema:
            type: string
          required: true
      responses:
        '204':
          description: pet deleted
        default:
          description: unexpected error
          content:
            application/json:
              schema:
                properties:
                  error:
                    $ref: '#/components/schemas/Error'
  /catsanddogs/{friendId}/veterinaryappointments:
    get:
      summary: List veterinary appointments for that particular furry friend
      operationId: findVetAppointmentsForPet
      parameters: 
      - name: friendId
        in: path
        description: Id of furry friend
        required: true
        schema:
          type: integer
          format: int64
      - name: X-Api-Key
          in: header
          schema:
            type: string
          required: true
      responses:
        '200':
          description: pet response
          content:
            application/json:
              schema:
                properties:
                  items:
                    type: array
                    items:
                      $ref: '#/components/schemas/Appointment'
                  meta:
                    $ref: '#/components/schemas/Metadata' 
components:
  schemas:
    Appointment:
      type: object
      properties:
        location:
          type: string
        date:
          type: string
    Pet:
      allOf:
        - $ref: '#/components/schemas/NewPet'
        - type: object
          required:
          - id
          properties:
            id:
              type: integer
              format: int64
    NewPet:
      type: object
      required:
        - name  
      properties:
        name:
          type: string
        tag:
          type: string
    Error:
      type: object
      required:
        - error
      properties:
        error:
          type: string
        errorDescription:
          type: string
        field:
          type: string
    Metadata:
      type: object
      properties:
        lastResultToken:
          type: string
        nextResultToken:
          type: string
          description: 'For lists only, no optional: A black box string'
        ETag:
          type: string
        lastModified:
          type: string
          description: 'optional, use yyyy-MM-ddTHH:mm:ssZ if set'
```

## Theory

The strength of REST lies in the separation into logical resources (bookings, shipments, users, customers, items, ...) and using (relatively) well defined terms to retrieve (GET), update (PUT, PATCH), create (POST) and delete (DELETE) them, so we get a reasonably standardized form

````text
    GET /users/{:user_id}/addresses/{:address_type}/city?format=UPCASE
````

In this contrived example, this would retrieve the uppercased city part of e.g. the home address of a specific user based on their unique user id. In the real world, it is also necessary to specify a protocol, server address and possibly port number. These parts of the invocation are always excluded.

### REST API Maturity levels

Going towards a RESTful interface is a journey where value can be derived even if the API does not feature all possible features and properties. For example, for some APIs it can be beneficial if the API itself is machine-readable and -consumable. Somewhat simplified, this is called a "hypermedia" interface.

An often cited source on *levels of maturity* for APIs is Leonard Richardsons and Martin Fowlers' [maturity model](https://martinfowler.com/articles/richardsonMaturityModel.html) which is a worthwhile read.

For DFDS purposes, we want externally consumable APIs to be level 2 or better, meaning that guidance on resources and HTTP methods should be followed, but hypermedia controls and other newer features are optional as long as there is no requirement for the clients for understand such features.

### In-depth on pagination

Using a "black box" token to control pagination allows underlying APIs some freedom in choosing a pagination solution that fits their needs, without being locked into the choice permanently (or requiring a version bump). We generally do not recommend using a page-number/offset approach because when the underlying collection changes, the results are not determistic . For simple cases, a page-number/offset implementation can be sufficient and in these cases, the paging string can simply be encoded using a two-way transformation function and given to the user.


### In-depth on versioning

1. REST API services MUST Support URL versioning
    1. They MAY support Header and/or Accepts format additionally provided they follow the form specified above
    2. MUST NOT use subdomains or query parameters solely for this purpose
    3. The service MUST NOT offer further mechanisms to versioning
2. The version MUST always be a positive whole number, digits-only, monotonically increasing, prefixed with the letter `v`, e.g. "`v1`, "`v18`"
    1. It MAY additionally provide a date-based subselection as a date-based format, provided that the format is YYYYMMDD gregorian as that format would not break the parent rule (so `/api/v1/20190618/users/{:id}`)
3. MUST increment upon any breaking changes
    1. MAY chose to increment upon non-breaking but otherwise major changes
4. If a user requests a deprecated version, or does not specify a version at all, the service MAY chose to handle it gracefully
    1. Fallback to oldest still supported version if no version provided. If your service offers v2, v3, and v4, and a consumer issues `GET /api/users{:id}`, the v2 version must be delivered. In this case, the service MUST indicate via return headers which version was returned. If no version is provided, and the oldest supported does not offer the requested operation, it is a standard 404 situation
    2. If a deprecated version is requested, the service MAY issue a HTTP 301 redirect to the oldest stable. It does not need to preserve the requested route
    3. If none of the above two options are suitable, the service MUST return a HTTP 404 response with a suitable error message

### Exemptions

This version of the standard has deliberately chosen not to cover these topics. They may be included in future revisions.

- Application and transport protocols
- API authorization and authentication
- HATEOAS style hyperlinking
- Other-than-HTTP communication paradigms, such as event-driven systems
- Client behaviour restrictions
- GraphQL

### Links and further reading

As we stated very early on, we have based our standards on the JSON API style of REST interfaces. It is, however, very long and goes into a great depth on very specific topics. No doubt being specific is a plus when designing interfaces but still the JSON API is a large time commitment. We recommend further reading in this order:

Start off with https://www.restapitutorial.com/. It gives a fairly good overview of the best and most important points when designing RESTful APIs.

If you have specific questions about a narrow topic, read the JSON:API 1.1 guide at https://jsonapi.org/format/1.1/. Unless **directly countermanded** by the document you are currently reading, it has many good points and patterns to follow.
