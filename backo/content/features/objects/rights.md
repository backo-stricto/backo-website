+++
title = "Access Control"
description = "Field-Level Access Control"
weight = 10
+++

## Overview

Backo provides fine-grained access control directly within its declarative data model. Rather than implementing authorization exclusively at the API or collection level, developers can define individual permissions for each field of an entity.

This approach allows a single data model to serve multiple categories of users while dynamically adapting their ability to access and manipulate information.

## Field Permissions

Backo provides two independent authorization parameters:

| Parameter | Description | Default |
|---|---|---|
| `can_read=` | Determines whether a field is accessible to the current user. | `True` |
| `can_modify=` | Determines whether the current user can change a field. | `True` |

Permissions can be static or dynamically evaluated through Python functions.

A field can therefore be readable but not modifiable, restricted to specific users, or protected according to the state of its containing object.

## Dynamic Authorization

Authorization functions follow a common signature:

```python
def check_permission(right_name: str, o: Item) -> bool:
    ...
```

The function receives the requested permission and the current item.

It can also access `current_user`, which represents the authenticated user and provides:

- `current_user._id`: the user identifier.
- `current_user.login`: the user login.
- `current_user.roles`: the user's assigned roles.
- `current_user.has_role(role)`: a role membership check.

This allows developers to implement both Role-Based Access Control (RBAC) and context-dependent authorization based on object ownership, business state or other application-specific criteria.

## Example: Employee Management

```python
from backo import Item, String, Float, current_user

def is_hr(right_name, o):
    return current_user.has_role("HR")

def is_owner_or_hr(right_name, o):
    return (
        current_user._id == o.user_id
        or current_user.has_role("HR")
    )

def can_modify_salary(right_name, o):
    return (
        current_user.has_role("HR")
        and o.status == "active"
    )

employee = Item({
    "user_id": String(
        can_modify=False
    ),
    "name": String(
        required=True,
        can_modify=is_hr
    ),
    "email": String(
        can_read=is_owner_or_hr,
        can_modify=is_owner_or_hr
    ),
    "salary": Float(
        min=0,
        can_read=is_owner_or_hr,
        can_modify=can_modify_salary
    ),
    "status": String(
        enum=["active", "inactive"],
        default="active",
        can_modify=is_hr
    ),
    "internal_notes": String(
        can_read=is_hr,
        can_modify=is_hr
    )
})
```

This model implements different authorization policies within a single entity.

| Field | Other employees | Owner | HR |
|---|---|---|---|
| Name | Read | Read | Read / Write |
| Email | No access | Read / Write | Read / Write |
| Salary | No access | Read | Read / Conditional write |
| Status | Read | Read | Read / Write |
| Internal notes | No access | No access | Read / Write |

Salary modifications are additionally restricted to active employees, illustrating how authorization can depend on both user roles and business data.

## RESTful API Integration

Field permissions complement collection-level authorization and are integrated into Backo's model-driven RESTful API architecture.

Authentication remains the responsibility of the application, which must identify the connected user and populate `current_user`. Backo then uses this context to evaluate authorization functions.

Security policies are defined within the data model rather than duplicated across individual API controllers.

## Benefits

- **Fine-grained authorization:** independent permissions for every attribute.
- **Dynamic security:** permissions adapt to users, roles and business context.
- **Centralized policies:** authorization rules are maintained alongside data definitions.
- **Reusable models:** multiple user profiles can interact with the same entity definition.
- **Reduced boilerplate:** field-level security does not require dedicated controllers.

By integrating field-level authorization into its declarative architecture, Backo makes security an integral part of the data model rather than an additional application layer.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% <devlab.card title="Stricto documentation" icon="external-link" tag="External" href="https://stricto.readthedocs.io/en/latest/"> %}
Full Stricto documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}