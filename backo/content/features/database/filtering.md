+++
title = "Agnostic filtering"
description = "Database-Agnostic Filtering with `SFilter`."
weight = 3
+++

## Overview

Backo relies on Stricto's `SFilter` mechanism to provide a unified filtering language across heterogeneous database systems.

Instead of writing SQL statements, MongoDB queries or storage-specific filtering functions, developers express selection criteria independently of the underlying database technology.

Backo delegates the translation of these filters to its database connectors, allowing filtering operations to be executed natively by the database whenever possible.

## The `SFilter` Structure

An `SFilter` is composed of three elements:

| Parameter | Description |
|---|---|
| `path` | JSONPath expression identifying the targeted attribute. |
| `operator` | Comparison or logical operator. |
| `value` | Comparison value or nested filtering expression. |

Filters can be recursively combined to express complex selection criteria.

```python
from stricto import SFilter, Operator

my_filter = SFilter(None, Operator.AND, [
    SFilter("$.price", Operator.GT, 100),
    SFilter("$.category", Operator.EQ, "hardware"),
    SFilter("$.stock", Operator.GT, 0)
])
```

This filter selects hardware products whose price exceeds 100 and whose stock is positive.

The expression does not contain any database-specific instructions.

## Database-Specific Translation

Each database connector is responsible for adapting the filtering expression to its underlying storage technology.

{% <devlab.diagram id="mda" label="Database-Specific filtering Translation" description="MDatabase-Specific filtering Translation."> %}
flowchart TD
    A[Application / REST API] --> B[SFilter]
    B --> C[Collection]
    C --> D[DBHandler]
    D --> M[MongoDB Query]
    D --> S[SQLite Query]
    D --> P[PostgreSQL Query]
    D --> F[Python Filtering]
    M --> M1[(MongoDB)]
    S --> S1[(SQLite)]
    P --> P1[(PostgreSQL)]
    F --> F1[YAML / Other Storage]
{% </devlab.diagram> %}

### MongoDB Translation

The previous filter is conceptually equivalent to the following MongoDB query:

```python
{
    "$and": [
        {"price": {"$gt": 100}},
        {"category": {"$eq": "hardware"}},
        {"stock": {"$gt": 0}}
    ]
}
```

MongoDB executes the filtering operation directly on its documents, potentially benefiting from existing database indexes.

### SQL Translation

For a relational database with corresponding columns, the equivalent SQL query would be:

```sql
SELECT *
FROM products
WHERE price > 100
  AND category = 'hardware'
  AND stock > 0;
```

SQLite and PostgreSQL connectors adapt filtering expressions to their respective SQL environments.

### Python-Based Filtering

For storage systems without suitable native query capabilities, filtering is performed directly by backo using the `SFilter.check()` method.

```python
matching_products = [
    product
    for product in products
    if my_filter.check(product)
]
```

This provides a filtering mechanism even for simple storage systems that do not implement a dedicated query language.

## Integration with Transformers

When integrating an existing database, the application's logical field names may differ from their physical storage representation.

For example:

| Logical Backo path | Physical database field |
|---|---|
| `$.price` | `unit_price` |
| `$.category` | `product_type` |
| `$.stock` | `available_quantity` |

Backo's `Transformer` mechanism enables the adaptation of both data structures and filtering expressions.

The application continues to use its logical data model, while the persistence layer targets the corresponding physical database fields.

## Benefits

- **Database independence:** write filtering expressions once, independently of storage technology.
- **Native query execution:** delegate filtering to the database whenever supported.
- **Performance:** reduce unnecessary data transfers and potentially benefit from database indexes.
- **Complex queries:** combine comparison operators and nested logical expressions.
- **Legacy integration:** filter existing databases through transformed logical field names.
- **Multi-database consistency:** provide a common filtering interface across heterogeneous collections.

By introducing `SFilter` as an intermediate query representation, Backo separates the expression of business criteria from their technical execution. Each database connector acts as a query translator, enabling a unified and extensible approach to data selection.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}