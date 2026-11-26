+++
title = "Selection"
description = "Named Selections for custom `select`."
weight = 10
+++

## Overview

Backo provides named selections to expose reusable, business-oriented queries through its RESTful API.

While standard CRUD operations allow clients to retrieve objects using arbitrary filters, named selections enable developers to centralize predefined queries on the server side.

A selection defines which objects should be retrieved, which attributes should be returned and how the results should be sorted.

Typical use cases include dashboards, business reports, administrative interfaces and operational workflows.

## Selection Definition

Backo provides the `Selection` class, which supports the following configuration:

| Parameter | Description |
|---|---|
| `selectors` | List of paths identifying the attributes to return. |
| `filter` | An `SFilter` expression or a function generating a filter. |
| `sort` | Ordered list of sorting expressions. |
| `can_read` | Authorization rule controlling access to the selection. |

Selections are registered at the collection level using `register_selection()`.

The `_id` attribute is automatically included in the results. When the selector list is `None`, complete items are returned.

## Example: Product Inventory Management

Consider an inventory management application that needs to identify products that are out of stock.

```python
from backo import Selection, Collection, SFilter, Operator

products = Collection( "products", product_item, my_db_connector )

out_of_stock = Selection(
    ["$.name", "$.category", "$.price", "$.stock"],
    filter=SFilter("$.stock", Operator.EQ, 0),
    sort=["$.category", "$.name"]
)

products.register_selection(
    "out_of_stock",
    out_of_stock
)
```

This selection returns a structured list of unavailable products, ordered by category and name.

## Dynamic Selections

A selection can also use a Python function to dynamically generate its filter.

For example, an order management application can expose overdue orders according to the current date.

```python
from datetime import datetime

def overdue_filter():
    return SFilter(
        "$.delivery_date",
        Operator.LT,
        datetime.now()
    )

overdue_orders = Selection(
    ["$.reference", "$.customer.name", "$.delivery_date"],
    filter=overdue_filter,
    sort=["$.delivery_date"]
)

orders.register_selection(
    "overdue_orders",
    overdue_orders
)
```

Dynamic filters allow queries to adapt to the current application context without requiring clients to construct the filtering expression themselves.

## Automatic RESTful API Integration

Backo automatically exposes registered selections through dedicated REST endpoints.

```http
GET /inventory/products/_selections/out_of_stock
```

An example response:

```json
{
    "result": [
        ["P001", "Server", "hardware", 1200, 0],
        ["P002", "Router", "network", 250, 0]
    ],
    "_skip": 0,
    "_page": 10
}
```

## Query String and Additional Filtering

Named selections can be combined with query string parameters, allowing API consumers to refine a predefined selection without defining a new server-side query.

For example, a client may request only hardware products that are out of stock, with pagination and a total count.

```http
GET /inventory/products/_selections/out_of_stock?category=hardware&_page=20&_skip=0&_total=1
```

| Query parameter | Purpose |
|---|---|
| `category=hardware` | Restricts results to hardware products. |
| `_page=20` | Limits the page to 20 results. |
| `_skip=0` | Starts from the first matching result. |
| `_total=1` | Requests the total number of matching items. |

The additional filtering criteria are combined with the predefined selection filter. The resulting expression is logically equivalent to:

```text
stock == 0 AND category == "hardware"
```

For a relational database, the equivalent query would be:

```sql
SELECT id, name, category, price, stock
FROM products
WHERE stock = 0
  AND category = 'hardware'
ORDER BY category, name
LIMIT 20 OFFSET 0;
```

The selection therefore defines the business scope of the query, while the query string provides additional flexibility to API consumers.

This mechanism is particularly useful for dynamic tables, searchable dashboards and paginated administrative interfaces.

## Database Independence

Selections rely on Backo's database abstraction layer. Their filtering expressions can be translated into native database queries by compatible connectors, allowing the same selection definition to operate across different storage technologies.

Selectors can also target nested attributes and referenced entities, making it possible to produce business-oriented representations of connected data.

## Benefits

- **Reusable business queries:** define selection logic once on the server.
- **Reduced network traffic:** return only relevant attributes.
- **Dynamic filtering:** combine predefined filters with query string parameters.
- **Pagination:** efficiently retrieve large collections.
- **Centralized authorization:** control access through selection-specific permissions.
- **Database independence:** maintain consistent queries across storage systems.
- **Frontend simplification:** expose data structures suitable for tables, dashboards and reports.

Named selections transform generic data retrieval into business-oriented API resources, combining server-side business logic with flexible client-side filtering.
{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}