# Resource Type Schemas

* **Author**: `Ryan Nowak` (`@rynowak`)

## Overview

The user-defined-types feature of Radius enables users to extend Radius by defining their own resource types. These user-defined-types are first-class in Radius and provide the same features and integrations that built-in resource types provide.

The challenge for users is to define schemas for these resource types. For resource type authors, the schema is the way define a contract to support their users. The schema of a resource type is its API. It defines what properties developers are allowed to set, and what data is provided to applications. 

This is a point of differentiation and customization. For example an organization might choose to use t-shirt sizes to describe the storage capacity of a database (eg: `S` `M` `L`...). Or they might define their own vocubularly for fault domains (eg: `zonal`, `regional`, etc).

From a technical point of view defining schemas is complicated. OpenAPI (and JSON-Schema) is the state of the art, but it's a difficult tool to wield. It provides many constructs that are not good API design. As a result, every project that leverages OpenAPI tends to define its own supported subset of its functionality. 

We've arrived at the purpose of this document. This will be a technically-focused description of the supported feature set for the schema of a resource type in Radius.

## Terms and definitions

| Term                 | Meaning                                                                                                                                                                                                                                                                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenAPI              | A definition language for describing HTTP APIs. Includes both their HTTP semantics and the structure of payloads and URLs. For the purposes of this document we are focused on the OpenAPI features that describe payloads of requests and responses in JSON (JSON-Schema). For the purposes of this document we are focused on OpenAPI v3. |
| Schema / JSON-Schema | A JSON format for describing the structure of a JSON document. This design document may use the terms `Schema`, `OpenAPI Schema` and `JSON-Schema` interchangably. For our purposes they refer to the same thing.                                                                                                                           |

**This design document is highly technical and assumes some familiarity with Go (the programming language), OpenAPI v3, JSON-Schema, and Kubernetes CRDs.**

Readers are asked to digest the following links ahead of reviewing. 

- [Examples](https://spec.openapis.org/oas/v3.0.3.html#schema-object-examples)
- [Kubernetes CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)


*Optionally:*

- [JSON-Schema Spec](https://json-schema.org/draft/2020-12/json-schema-validation#name-a-vocabulary-for-structural)
- [OpenAPI v3 Spec](https://spec.openapis.org/oas/v3.1.0.html#schema-object)

## Objectives

> **Feature Spec:** [User Defined Types](https://github.com/radius-project/design-notes/pull/74)

> **Issue Reference:** [User-Defined Types](https://github.com/radius-project/radius/issues/6688)

### Goals

This document defines the subset of schemas we will support in Radius for user-defined resource types. JSON-Schema can be used to define very complex schemas that do not reflect good API design practices. As such, every project that leverages it defines their supported subset.

As principles, the following guide our decision-making and what we're optimizing for.

- Start small: We'll enable a minimal set of features, and expand what's allowed based on need. This gives users good feedback up front and 
- Simplicity and predicatabilty over terseness: There's a tendancy to count keystrokes when doing design work. This is a bad habit given powerful editing tools like Bicep and AI assistants. We should optimize for smaller design space to keep users focused on decisions valuable to them.
- Align with universal programming constructs: To make things easy for users we'll align with concepts that exist in every programming language. This helps eliminate bad choices and ensures maximum compatibility for consuming the Radius APIs regardless of the client.
  
### Non goals

- Supporting existing schemas (brownfield): We assume that resource types are designed to work with Radius, rather than the opposite. It's unrealistic that Radius will be compatible with existing APIs that were designed for other systems. Radius resources need to look like ARM/UCP resources and follow our rules.
- Supporting the full breadth of OpenAPI/JSON-Schema: OpenAPI/JSON-Schema need to support bad designs so that they can describe existing APIs. We don't have this requirement.

### User scenarios (optional)

See [User Defined Types](https://github.com/radius-project/design-notes/pull/74) feature spec.


## User Experience (if applicable)

Users author a manifest like the following to defind a user-defined resource type.

**Sample Input:**

```yaml
name: MyCompany.Resources
types:
  postgresDatabases:
    apiVersions:
      '2025-01-01-preview':
        schema: 
          type: object
          properties:
            size:
              type: string
              description: The size of database to provision
              enum:
              - S
              - M
              - L
              - XL
            status:
              type: object
              readOnly: true
              properties: 
                binding:
                  type: 'object'
                  properties:
                    hostname: 
                      type: string
                    username:
                      type: string
                    secret:
                      type: string
                      schema:
                        type: 'object'
                        properties:
                          password:
                            type: 'string'
                recipe: 
                  $ref: 'https://radapp.io/schemas/v1#RecipeStatus'
                  
          required:
          - size

    capabilities: ["SupportsRecipes"]
```
**Sample Output:**

Users can use this schema to generate a Bicep extension providing strongly-typed editor support. Here's an example of Bicep code that matches this API definition.

```bicep
extension radius
extension mycompany // generated from the schema

resource webapp 'Applications.Core/containers@2024-01-01' = {
  name: 'sample-webapp'
  properties: {
    image: '...'
    env: {
      // Bicep editor has completion for these 
      DB_HOSTNAME: { fromValue: { value: db.properties.binding.hostname } }
      DB_USERNAME: { fromValue: { value: db.properties.binding.username } }
      DB_HOSTNAME: {
        fromSecret: {
          secret: db.properties.binding.secret 
          key: 'password'
        }
      }
    }
  }
}

resource db 'MyCompany.Resources/postgresDatabases@2025-01-01-preview' = {
  name: 'sample-db'
  properties: {
    size: 'L' // Bicep editor can validate this field
  }
}
```

## Design

### High Level Design

N/A - this is more of a spec.

### Architecture Diagram

N/A - this is more of a spec.

### Detailed Design

When the user provides us a schema, they are defining the `properties` of a resource type. There is no support for defining custom fields outside of `properties`.

Example:

```bicep
resource db 'MyCompany.Resources/postgresDatabases@2025-01-01-preview' = {
  name: 'sample-db' 
  properties: {
    size: 'L' // Matches the `schema` element from the UDT manifest
  }

  // Not allowed to define custom fields here
}
```

#### Structure

The schema for `properties` (eg: the `schema` element in the manifest) must be `object`. All other limitations for `object` apply here (continue reading).

In general, all places where a `type` is supported, a type is explicitly required. That includes:

- each property of an `object`
- `additionalProperties` ("maps")
- `item` (arrays)

The user should define their custom fields within `properties`. The set of rules applied to each construct depend on the property type, which will be addressed in the following sections.

#### References

Schemas are allowed to reference reusable built-in schemas of Radius using the `$ref` construct of OpenAPI. 

The example above uses `$ref: 'https://radapp.io/schemas/v1#RecipeStatus'` to reference the schema for a "recipe status". This aids with consistency and reduces boilerplate for users. 

The exact URL and set of types that can be referenced is TBD.

Schemas are not allowed to use `$ref` to reference other than what we provide in Radius. This reduces concept count and simplifies our tooling. We can reconsider this based on feedback. This also prevents the definition of recursive types.
 
#### Scalars (string/number)

Schemas will support every construct OpenAPI provides for a scalar field. That includes all of the modifiers like `enum` and `format` and also all of the validation attributes like `minLength`.

#### Arrays

Arrays must specify a type for `item`.

#### "Maps"

OpenAPI does not directly include an equivalent of the `map` type from Go. However, this pattern is still possible, and is desirable when a collection of data:

- Has a clear "name"
- Order does not matter

To apply this pattern:

- The `type` must be `object`
- The `additionalProperties` field must be set to a type.
- No other property declarations are allowed.

#### Objectives

Objects may not use the following constructs that support polymorphism:

- `allOf`
- `anyOf`
- `oneOf`
- `not`

This is a point in time decision to limit complexity, and we'll likely get feedback that polymorphism is important. 

Objects may not set both `additionalProperties` and define their own properties.


### API design (if applicable)

N/A - this is more of a spec.

### CLI Design (if applicable)

N/A - this is more of a spec.

### Implementation Details

N/A - this is more of a spec.


### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

## Security

<!--
Describe any changes to the existing security model of Radius or security 
challenges of the features. For each challenge describe the security threat 
and its mitigation with this design. 

Examples include:
- Authentication 
- Storing secrets and credentials
- Using cryptography

If this feature has no new challenges or changes to the security model
then describe how the feature will use existing security features of Radius.
-->

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

## Monitoring and Logging

<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Development plan

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

## Open Questions

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible. 
-->

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->