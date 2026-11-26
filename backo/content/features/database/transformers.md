+++
title = "Transformers"
description = "Data transformation for adaptation to an existing database."
weight = 2
+++


## Overview

Backo provides a `Transformer` mechanism to decouple the logical application data model from the physical representation of data in a database or an external REST API.

When integrating an existing information system, developers often encounter data structures, naming conventions and formats that do not correspond to their application requirements. Transformers provide an adaptation layer without requiring modifications to the original data source.

The application can therefore define its own data model independently of the constraints imposed by existing storage systems.

## Transformation Architecture

Transformers operate at the persistence layer, between the Backo data model and the underlying storage system.

{% <devlab.diagram id="rest" label="Transformers" description="Object transformations to adapt to a database."> %}
flowchart TD
    A[RESTful API] <--> B[Item / Collection]
    B <--> C[DBHandler]
    C <--> T[Transformers]
    T <--> D[(Existing Database)]
{% </devlab.diagram> %}

Transformations can operate in two directions:

- **Database to Backo:** convert stored data into the application's logical representation.
- **Backo to database:** adapt application data to the representation expected by the storage system.

This separation keeps transformation logic outside business logic and API controllers.

## Available Transformers

| Transformer | Description |
|---|---|
| `Transformer` | Base abstraction for implementing custom transformations. |
| `RenameTransformer` | Maps attribute names or paths between the application and the database. |
| `IgnoreTransformer` | Excludes selected attributes from the transformation process. |
| Custom transformers | Implement specific transformations according to business requirements. |

Custom transformations can address differences in naming conventions, nested structures, data formats and value representations.

## Example: Integrating a Legacy Customer Database

Consider an existing customer database containing the following document:

```json
{
    "_id": "CUST001",
    "CUST_FIRSTNAME": "Victor",
    "CUST_LASTNAME": "Hugo",
    "CUST_EMAIL": "victor@example.com",
    "CUST_ACTIVE": true,
    "LEGACY_INTERNAL_CODE": "X874"
}
```

The Backo application defines a more natural and consistent data model:

```python
from backo import Item, String, Bool

customer = Item({
    "firstname": String(),
    "lastname": String(),
    "email": String(),
    "active": Bool()
})
```

The transformation layer establishes the following mapping using renaming and exclusion rules:

| Database attribute | Backo attribute |
|---|---|
| `CUST_FIRSTNAME` | `firstname` |
| `CUST_LASTNAME` | `lastname` |
| `CUST_EMAIL` | `email` |
| `CUST_ACTIVE` | `active` |
| `LEGACY_INTERNAL_CODE` | Ignored |

The resulting entity exposed through Backo's RESTful API is:

```json
{
    "_id": "CUST001",
    "firstname": "Victor",
    "lastname": "Hugo",
    "email": "victor@example.com",
    "active": true
}
```

The original database structure remains unchanged.

## Filter Transformation

Transformers are not limited to data representation. They also participate in the adaptation of `SFilter` expressions.

For example, an application-level filtering expression such as:

```text
$.lastname == "Hugo" AND $.active == true
```

must target the corresponding physical database attributes.

For MongoDB, the equivalent native query would be:

```json
{
    "CUST_LASTNAME": "Hugo",
    "CUST_ACTIVE": true
}
```

The conceptual transformation pipeline is:

{% <devlab.diagram id="transformation" label="Transformators" description="Transformation on database access."> %}
flowchart LR
    A[SFilter] --> B[Filter Transformation]
    B --> C[Native Query Translation]
    C --> D[(Database)]
{% </devlab.diagram> %}

This allows filtering to remain independent of the physical data structure while preserving native database query execution whenever the transformation and connector support it.

## Benefits

- **Legacy integration:** use existing databases without redesigning their schemas.
- **Model independence:** define application-oriented structures and naming conventions.
- **Bidirectional adaptation:** translate data between logical and physical representations.
- **Query transformation:** adapt filtering expressions to underlying database attributes.
- **Interoperability:** normalize data originating from heterogeneous storage systems and external APIs.
- **Extensibility:** implement custom transformations for complex integration requirements.

Backo's transformers extend the ORM abstraction beyond database technology independence. They also provide independence from physical data structures, making Backo particularly suitable for integrating, modernizing and extending existing information systems.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}