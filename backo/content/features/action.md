+++
title = "Action"
description = "Custom Actions with Access Control."
weight = 15
+++

## Overview

Backo provides an action mechanism that extends standard CRUD operations with custom business operations.

While CRUD operations focus on manipulating data, actions represent explicit business processes such as confirming an order, approving an invoice, publishing a document or closing a project.

Actions are registered through `register_action()` and integrated into Backo's automatically generated RESTful API.

## Action Definition and Permissions

An action combines a Python business function with authorization rules and an API interface.

Backo distinguishes between two complementary permission mechanisms:

| Parameter | Description |
|---|---|
| `can_see/exist` | Determines whether the action make sens. |
| `can_execute` | Determines whether the current user is authorized to execute the action. |

Both permissions can depend on the authenticated user, their roles and the state of the entity.

This distinction allows an action to be visible but temporarily unavailable for execution. Visibility alone should never be considered an execution authorization.

## Example: Invoice Approval

Consider an invoice management application where accountants and managers can see the approval action, but only accountants can execute it on pending invoices.

### Data Model

```python
from backo import Item, String, Float, current_user

invoice_item = Item({
    "reference": String(),
    "amount": Float(min=0),
    "status": String(
        enum=["draft", "pending", "approved"],
        default="draft",
        can_modify=False
    ),
    "approved_by": String(can_modify=False)
})
```

The protected fields prevent standard client-side update operations from bypassing the approval workflow.

### Action Permissions

```python
def approval_exist(right_name, o):
    return (
        o.status == "pending"
    )

def can_execute_approval(right_name, o):
    return (
        current_user.has_role("accountant")
    )
```

### Business Logic

```python
def approve_invoice(o):
    o.status = "approved"
    o.approved_by = current_user.login
    o.save()
```

### Action Registration

The following illustrative registration associates the business function with its visibility and execution policies:

```python
from backo import Collection, Action

invoices = Collection( "invoices", invoice_item, my_db_connector )

invoices.register_action(
    "approve",
    Action(
        approve_invoice,
        exist=approval_exist,
        can_execute=can_execute_approval
    )
)
```

The approval operation becomes a named action associated with the invoice collection.


## Calling an Action through the RESTful API

Backo automatically generates a dedicated endpoint for each registered action.

```http
POST /<app_name>/<collection>/_actions/<action_name>/<_id>
```

For example, approving invoice with _id = `INV001` with an authenticated user:

```bash
curl -X POST \
  'http://localhost/accounting/invoices/_actions/approve/INV001' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{
    "comment": "Invoice checked and approved."
  }'
```

Backo validates the input parameters, evaluates execution permissions and invokes the associated business function.

After successful execution, the invoice contains the updated business information:

```json
{
    "_id": "INV001",
    "reference": "2026-001",
    "amount": 1500.0,
    "status": "approved",
    "approved_by": "alice",
    "approval_comment": "Invoice checked and approved."
}
```

## Dynamic Action Availability

| User role | Invoice status | Exist | Executable |
|---|---|---|---|
| Accountant | Pending | Yes | Yes |
| Accountant | Draft | No | No |
| Accountant | Approved | No | No |
| Manager | Pending | Yes | No |
| Employee | Pending | Yes | No |

This mechanism is particularly useful for frontend applications that dynamically display available actions and disable operations that cannot currently be executed.


## Benefits

- **Business-oriented APIs:** expose meaningful operations beyond CRUD.
- **Dynamic visibility:** adapt available actions to user profiles.
- **Fine-grained authorization:** independently control visibility and execution.
- **Context-dependent permissions:** evaluate business conditions before execution.
- **Controlled workflows:** protect business state transitions.
- **Centralized business logic:** define operations and permissions directly on the backend.

By combining `register_action()`, `can_see` and execution permissions, Backo provides a declarative approach to implementing secure, dynamic and business-oriented application workflows.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}