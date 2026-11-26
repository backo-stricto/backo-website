+++
title = "Model and Constraints"
description = "Fields and Data Constraints."
weight = 1
+++

## Declarative Data Modeling

Backo relies on Stricto, a Python library for defining strongly typed and structured data models. Unlike conventional schema validation libraries, Stricto embeds validation rules directly into data types.

Each field describes not only the nature of its data but also its constraints, default value, transformations and dynamic behavior. Validation is integrated into object manipulation rather than implemented as a separate processing step.

## Field Types

Stricto provides a comprehensive range of elementary and composite types.

| Field type | Description |
|---|---|
| `String()` | Textual value |
| `Int()` | Integer value |
| `Float()` | Floating-point number |
| `Bool()` | Boolean value |
| `Datetime()` | Date and time |
| `Bytes()` | Binary content |
| `Dict({...})` | Structured object with named fields |
| `List(type)` | Collection of typed elements |
| `Tuple((...))` | Fixed-length sequence of typed elements |
| `In([...])` | Union of multiple accepted types |

Composite types can be recursively combined to represent complex hierarchical data structures. Each nested field can define its own validation rules and constraints.

## Declarative Constraints

Constraints are specified directly within field definitions.

| Option | Purpose |
|---|---|
| `required` | Mandatory value |
| `default` | Static or dynamically generated initial value |
| `enum` | Restriction to a predefined enumeration |
| `min` / `max` | Numerical bounds or list cardinality |
| `pattern` | Regular expression validation |
| `patterns` | Multiple regular expression checks |
| `uniq` | Uniqueness of elements within a list |
| `constraint` | Custom validation function |
| `constraints` | Combination of custom validation rules |
| `description` | Field documentation |

Custom constraints allow developers to implement business-specific validation rules using the candidate value, its previous value and the containing object.

This makes it possible to express both individual field constraints and dependencies between multiple fields.

## Dynamic Field Behavior

Beyond static validation, Stricto supports dynamic field processing through several mechanisms:

- **`transform`** — Normalizes or converts incoming values before assignment.
- **`default`** — Supports dynamically generated initial values through callable functions.

These mechanisms allow business rules and data processing logic to be integrated directly into the data model.

## Example: Product Data Model

```python
from backo import Dict, String, Int, Float, List, Bool

product = Dict({
    "name": String(
        required=True,
        transform=lambda o, value, old: value.strip()
    ),
    "reference": String(
        required=True,
        pattern=r"^PRD-[0-9]{6}$"
    ),
    "category": String(
        enum=["hardware", "software", "service"]
    ),
    "price": Float(
        required=True,
        min=0
    ),
    "quantity": Int(
        default=0,
        min=0,
        constraint=lambda o, value, old: value % 2 == 0
    ),
    "tags": List(
        String(),
        default=[],
        uniq=True,
        max=10
    ),
    "available": Bool(default=True),
    "supplier": Dict({
        "name": String(required=True),
        "email": String(
            pattern=r"^[^@]+@[^@]+\.[^@]+$"
        )
    })
})
```

This model illustrates several validation capabilities:

- Product names are automatically normalized.
- Product references must follow a predefined format.
- Categories are restricted to predefined values.
- Prices must be non-negative.
- Quantities must be non-negative and even.
- Tags must be unique and limited to ten elements.
- Supplier information is represented by a nested structured object.

## Integration

By incorporating Stricto's type system into its `Item` models, Backo centralizes data structures, validation rules, transformations and business constraints within a single declarative definition.

These rules become an integral part of the object lifecycle, benefiting both direct Python manipulation and RESTful API operations.

This approach reduces repetitive validation code, improves data consistency and makes the data model a single source of truth for application development.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% <devlab.card title="Stricto documentation" icon="external-link" tag="External" href="https://stricto.readthedocs.io/en/latest/"> %}
Full Stricto documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}