+++
title = "Conditional"
description = "Fields or sel of fields which exists only in some cases"
weight = 10
+++

## Conditional Fields with `exists=`

Backo, through Stricto, provides a declarative mechanism for defining conditional fields using the `exists=` parameter. This mechanism allows the structure of an object to dynamically adapt to its content and business context.

Unlike optional fields, which may remain undefined, conditional fields are designed to exist only when a specific condition is satisfied.

The condition is expressed as a Python function that evaluates the state of the object and returns a boolean value.

## Example: Customer Data Model

A customer can be either an individual or a company. Company-specific information should only exist when the customer is a company.

```python
from backo import Item, String

customer = Item({
    "name": String(required=True),
    "type": String(
        enum=["individual", "company"],
        default="individual"
    ),
    "company_name": String(
        required=True,
        exists=lambda o: o.type == "company"
    ),
    "vat_number": String(
        exists=lambda o: o.type == "company"
    )
    "surname": String(
        exists=lambda o: o.type == "individual"
    )
})
```

When the customer type is `individual`, company-specific fields are excluded from the active data structure. When the customer type becomes `company`, these fields become applicable, together with their associated validation rules.

## Benefits

- **Dynamic data structures:** objects adapt to their current state.
- **Conditional validation:** constraints apply only when the corresponding fields exist.
- **Declarative business rules:** structural conditions are defined directly within the data model.
- **Data consistency:** irrelevant attributes are excluded from the active object structure.
- **Reduced boilerplate:** conditional structures do not require separate models or manual field management.

Combined with Backo's ORM and automatic RESTful API generation, `exists=` enables developers to describe dynamic business entities through a single declarative model.

Rather than defining a static schema for every possible business scenario, developers can express how the structure of an entity evolves according to its own data.


{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% <devlab.card title="Stricto documentation" icon="external-link" tag="External" href="https://stricto.readthedocs.io/en/latest/"> %}
Full Stricto documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}