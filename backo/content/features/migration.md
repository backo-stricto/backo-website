+++
title = "Migration"
description = "A minimal migration tool."
weight = 20
+++

## Overview

Backo provides a built-in data migration mechanism to manage the evolution of application data models. When an `Item` definition changes, previously stored data may no longer comply with the new model.

Such changes may include adding mandatory fields, renaming attributes, modifying data types, restructuring objects or introducing new business constraints.

Backo addresses this problem through the `migrate()` method of the `Backoffice`, independently of the database technology associated with the collection.

## Migration Process

A migration is defined through a Python function that receives a dictionary representing a stored record and returns its updated representation.

{% <devlab.diagram id="migration" label="Migration" description="Migrate datas."> %}
flowchart LR
    A[(Existing Database)] --> B[Raw Dictionary]
    B --> C[Migration Function]
    C --> D[Updated Dictionary]
    D --> E{Migration Strategy}
    E --> F[Migration Report]
    E --> G[(Updated Database)]
{% </devlab.diagram> %}

Working directly with dictionaries makes it possible to manipulate legacy data before dealing with the requirements of the new model.

## Example: Product Model Evolution

Consider a product collection whose data model must evolve to introduce a mandatory currency, rename the price attribute and remove an obsolete field.

### Existing data

```json
{
    "_id": "PRD001",
    "name": "Server",
    "price": 1200,
    "obsolete_code": "SRV-OLD"
}
```

### Migration script

```python
from backo import MigrationStrategy
from myapp import backoffice

def migrate_product(o: dict) -> dict:
    if "currency" not in o:
        o["currency"] = "EUR"

    if "price" in o:
        o["unit_price"] = float(o.pop("price"))

    o.pop("obsolete_code", None)

    return o

# Preview changes without writing to the database
report = backoffice.migrate(
    "products",
    migrate_product,
    strategy=MigrationStrategy.DRY_RUN
)

# Apply changes
report = backoffice.migrate(
    "products",
    migrate_product,
    strategy=MigrationStrategy.EXECUTE
)
```

### Migrated data

```json
{
    "_id": "PRD001",
    "name": "Server",
    "unit_price": 1200.0,
    "currency": "EUR"
}
```

The transformation is idempotent: running it repeatedly does not introduce additional changes.

## Migration Strategies

| Strategy | Behavior |
|---|---|
| `MigrationStrategy.DRY_RUN` | Previews modifications without persisting them. Default strategy. |
| `MigrationStrategy.EXECUTE` | Persists modified records. |
| `MigrationStrategy.FORCE` | Rewrites records even when their content remains unchanged. |

Individual records can also be targeted through `_id` or `_ids`, allowing progressive migration and controlled testing.

## Migration Reports

Each migration produces a structured report containing:

- `report.nochanges._ids`: identifiers of unaffected records.
- `report.changes._ids`: identifiers of modified records.
- `report.changes.total`: number of modifications.
- `report.changes.diff`: detailed differences represented using DeepDiff.

These reports help developers review changes before committing them and document the results of a migration.

## Important Considerations

Migration operates directly through database handlers. It bypasses normal item-level processing, authorization and relationship management. Metadata is not refreshed automatically.

Migration scripts must therefore explicitly preserve business and relational consistency whenever necessary.

## Conclusion

Backo makes data migration an integral part of application evolution. By combining programmable transformations, dry-run execution, selective migration and detailed reporting, it enables developers to adapt existing data to evolving business models without implementing database-specific migration scripts.

The migration logic remains focused on the application's data structure rather than its underlying storage technology.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}