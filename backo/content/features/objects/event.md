+++
title = "Events"
description = "Manage events on fields and trigg your own events."
weight = 10
+++

## Event Management with Backo

Backo provides an event-driven mechanism that allows developers to associate business logic with data manipulation and object lifecycle operations.

Based on Stricto's event system, the `on=` parameter enables any object or field to register callbacks for predefined or custom events.

## Lifecycle Events

Backo automatically triggers events during the lifecycle of an `Item`.

| Event | Trigger |
|---|---|
| `created` | After object creation |
| `loaded` | After loading an object |
| `before_save` | Before saving an object |
| `saved` | After saving an object |
| `before_delete` | Before deleting an object |
| `change` | When an object's value changes (inherited from Stricto) |

## Example: Order Lifecycle

```python
from backo import Item, String, Float

def log_event(event_name, root, me, **kwargs):
    print(f"Order {root.reference}: {event_name}")

order = Item({
    "reference": String(required=True),
    "amount": Float(min=0),
    "status": String(default="pending")
}, on=[
    ("created", log_event),
    ("saved", log_event),
    ("before_delete", log_event)
])
```

Each callback receives the event name, the root object and the object on which the listener is registered.

## Custom Events and Selective Listening

Developers can define their own events and explicitly trigger them using `trigg()`. Listeners can also target specific attributes using JSONPath expressions.

```python
order = Item({
    "status": String(default="pending")
}, on=[
    ("change", log_status, "$.status"),
    ("confirmed", notify_customer)
])

order.status = "confirmed"
order.trigg("confirmed")
```

Here, `log_status` and `notify_customer` represent application-specific callbacks. The first reacts to status changes, while the second listens for a custom business event.

## Benefits

- **Reactive business logic:** automatically respond to data changes.
- **Lifecycle hooks:** execute operations around persistence events.
- **Custom workflows:** implement business-specific events.
- **Selective listening:** react only to relevant attribute changes.
- **Separation of concerns:** decouple business behavior from API controllers.

By integrating events directly into its data model, Backo enables reactive and extensible backend development without requiring a separate event management infrastructure.


{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% <devlab.card title="Stricto documentation" icon="external-link" tag="External" href="https://stricto.readthedocs.io/en/latest/"> %}
Full Stricto documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}