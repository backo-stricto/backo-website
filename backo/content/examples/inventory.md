+++
title = "Equipment Inventory"
description = "An IT department needs to track its equipment, including laptops, monitors and other devices. Staff must be able to register equipment, find available devices, and assign or return them."
weight = 1
+++


## Installation

```bash
pip install backo
```

## Implementation

Create an `app.py` file:

```python
from flask import Flask
from backo import Backoffice, Collection, Item, String, Action
from backo.db import DBYmlConnector


# Define the equipment data model.
equipment_item = Item({
    "name": String(require=True),
    "serial_number": String(require=True),
    "category": String(enum=["laptop", "monitor", "other"]),
    "status": String(
        enum=["available", "assigned", "maintenance"],
        default="available",
    ),
    "assigned_to": String(default=""),
})

# Store equipment in a local YAML file.
equipment = Collection(
    "equipment",
    equipment_item,
    DBYmlConnector("equipment.yml"),
)


# Business logic: assign an available device.
def assign(action: Action, device: Item, **kwargs) -> None:
    device.assigned_to = action.employee
    device.status = "assigned"
    device.save(**kwargs)


def is_available(right_name: str, device: Item) -> bool:
    return device.status == "available"


equipment.register_action(
    "assign",
    Action(
        {"employee": String(require=True)},
        assign,
        exists=is_available,
        can_execute=is_available,
    ),
)


# Business logic: return an assigned device.
def return_device(action: Action, device: Item, **kwargs) -> None:
    device.assigned_to = ""
    device.status = "available"
    device.save(**kwargs)


def is_assigned(right_name: str, device: Item) -> bool:
    return device.status == "assigned"


equipment.register_action(
    "return",
    Action(
        {},
        return_device,
        exists=is_assigned,
        can_execute=is_assigned,
    ),
)

# Register the collection and generate the REST API.
backoffice = Backoffice("inventory")
backoffice.register_collection(equipment)

app = Flask(__name__)
backoffice.build_routes(app)

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5000)
```

## Running the application

```bash
python app.py
```

The API is available locally under `/inventory`.

## Example workflow

### 1. Register a new laptop

```bash
curl -X POST http://127.0.0.1:5000/inventory/equipment/ \
  -H 'Content-Type: application/json' \
  -d '{"name":"ThinkPad T14","serial_number":"IT-001","category":"laptop"}'
```

The response contains the newly created item and its `_id`. Its initial status is `available`.

Copy the returned identifier into a shell variable:

```bash
ID='<returned _id>'
BASE='http://127.0.0.1:5000/inventory'
```

### 2. Find available equipment

```bash
curl "$BASE/equipment/?status=available"
```

### 3. Assign the laptop to an employee

```bash
curl -X POST "$BASE/equipment/_actions/assign/$ID" \
  -H 'Content-Type: application/json' \
  -d '{"employee":"Alice Martin"}'
```

The action updates the employee and status together. Its execution predicate allows assignment only when the device is available.

### 4. Retrieve the updated device

```bash
curl "$BASE/equipment/$ID"
```

Illustrative response (additional metadata may be present):

```json
{
  "_id": "<generated id>",
  "name": "ThinkPad T14",
  "serial_number": "IT-001",
  "category": "laptop",
  "status": "assigned",
  "assigned_to": "Alice Martin"
}
```

### 5. Return the device

```bash
curl -X POST "$BASE/equipment/_actions/return/$ID" \
  -H 'Content-Type: application/json' \
  -d '{}'
```

The device becomes available again and its employee assignment is cleared.

### 6. Retrieve the OpenAPI specification

```bash
curl "$BASE/_openapi"
```

## Generated API endpoints

| Method | Endpoint (relative to `/inventory`) | Purpose |
| --- | --- | --- |
| POST | `/equipment/` | Register equipment |
| GET | `/equipment/` | List and filter equipment |
| GET | `/equipment/{id}` | Retrieve a device |
| PUT | `/equipment/{id}` | Update a device |
| DELETE | `/equipment/{id}` | Remove a device |
| POST | `/equipment/_actions/assign/{id}` | Assign a device |
| POST | `/equipment/_actions/return/{id}` | Return a device |
| GET | `/_openapi` | Retrieve the API specification |

## Why Backo?

This example requires no handwritten CRUD controllers. The data model defines allowed values, the connector handles persistence, and custom actions express the equipment lifecycle directly in Python.

## Production considerations

This is a local demonstration, not a production-ready inventory system. Generic CRUD updates can bypass the action workflow; restrict direct writes to lifecycle fields and enforce the associated invariants before deployment. Add authentication, role-based permissions, unique serial-number enforcement and concurrency protection. YAML storage is intended here for simplicity; consider a transactional database for concurrent access.

The example targets the documented Backo 0.4.1 API. It has not been integration-tested against a running server.

## References

- [Backo documentation](https://backo.readthedocs.io/en/latest/index.html)
- [Quickstart](https://backo.readthedocs.io/en/latest/Quickstart.html)
- [Database handlers](https://backo.readthedocs.io/en/latest/Database.html)
- [Actions](https://backo.readthedocs.io/en/latest/Action.html)
- [Routing](https://backo.readthedocs.io/en/latest/Routing.html)
- [Backo source code](https://github.com/backo-stricto/backo)