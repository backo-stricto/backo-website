+++
title = "Computed"
description = "Compute fields from others with set=."
weight = 10
+++

## A way to compute values

Backo, through Stricto, provides a declarative mechanism for defining computed fields using the `set=` parameter. Instead of receiving a value directly, a computed field derives its value from a Python function that has access to the root object.

Computed fields are automatically recalculated when other attributes change. They are read-only by design, ensuring consistency between source attributes and derived values.

## Basic Example

```python
from stricto import Dict, String, Float, Int

product = Dict({
    "name": String(),
    "price": Float(default=0.0),
    "quantity": Int(default=0),
    "total": Float(
        default=0.0,
        set=lambda o: o.price * o.quantity
    )
})

product.price = 25.0
product.quantity = 4

print(product.total)  # 100.0

product.quantity = 8

print(product.total)  # 200.0
```

No explicit recalculation is required. Stricto automatically maintains the consistency of the computed value.

## Dependency Management

By default, computed fields react to changes throughout the object. Developers can optimize this behavior by explicitly declaring dependencies using JSONPath expressions.

```python
"total": Float(
    default=0.0,
    set=(
        lambda o: o.price * o.quantity,
        ["$.price", "$.quantity"]
    )
)
```

In this example, modifying the product name does not trigger a recalculation of the total.

Computed fields can also depend on other computed fields, allowing developers to build chains of calculations.

## Benefits

- **Automatic consistency:** derived values remain synchronized with their source attributes.
- **Declarative business logic:** calculations are defined directly within the data model.
- **Performance optimization:** explicit dependencies prevent unnecessary computations.
- **Reduced boilerplate:** no manual synchronization or recalculation methods are required.

By integrating computed fields into its data model, Backo provides a reactive approach to backend development, making derived data an integral part of the application model rather than a separate processing layer.


{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% <devlab.card title="Stricto documentation" icon="external-link" tag="External" href="https://stricto.readthedocs.io/en/latest/"> %}
Full Stricto documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}