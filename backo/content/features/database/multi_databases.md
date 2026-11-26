+++
title = "Multi databases"
description = "Multi-Database Architecture."
weight = 2
+++


## Overview

Backo provides a native multi-database architecture, allowing a single backend application to interact with multiple storage systems simultaneously.

Instead of defining a global database connection, Backo associates each `Collection` with its own `DBHandler`. Collections remain independent of their physical storage while sharing a common data modeling, validation, authorization and RESTful API layer.

This approach enables polyglot persistence: selecting the most appropriate storage technology for each business entity.

## Architecture

{% <devlab.diagram id="mda" label="Multi-Database Architecture" description="Multi-Database Architecture."> %}
flowchart TD
    API[Unified RESTful API] --> B[Backoffice]
    B --> U[Users Collection]
    B --> P[Projects Collection]
    B --> T[Tasks Collection]
    U --> SQL[(PostgreSQL)]
    P --> MONGO[(MongoDB)]
    T --> YAML[YAML File]
{% </devlab.diagram> %}

Relationships are defined between collections, regardless of their underlying storage technology.

## Example: Project Management Application

Consider a project management application where users are stored in PostgreSQL, projects in MongoDB and tasks in a YAML file.

```python
from flask import Flask
from backo import (
    Backoffice, Collection, Item,
    String, Bool, Ref, RefsList
)
from backo.db import (
    DBPostgresConnector,
    DBMongoConnector,
    DBYmlConnector
)

users = Collection(
    "users",
    Item({
        "name": String(require=True),
        "email": String(),
        "projects": RefsList(
            coll="projects",
            field="$.owner"
        )
    }),
    DBPostgresConnector(
        "postgresql://user:password@localhost:5432/company"
    )
)

projects = Collection(
    "projects",
    Item({
        "name": String(require=True),
        "owner": Ref(
            coll="users",
            field="$.projects"
        ),
        "tasks": RefsList(
            coll="tasks",
            field="$.project"
        )
    }),
    DBMongoConnector(
        "mongodb://localhost:27017/company",
        "projects"
    )
)

tasks = Collection(
    "tasks",
    Item({
        "title": String(require=True),
        "completed": Bool(default=False),
        "project": Ref(
            coll="projects",
            field="$.tasks"
        )
    }),
    DBYmlConnector("/data/tasks.yml")
)

backoffice = Backoffice("company")

backoffice.register_collection(users)
backoffice.register_collection(projects)
backoffice.register_collection(tasks)

app = Flask(__name__)
backoffice.build_routes(app)
```

## Cross-Database Relationships

Despite being stored in three different storage systems, users, projects and tasks belong to the same logical data model.

Backo manages their relationships through `Ref` and `RefsList`, which define bidirectional associations between collections. These relationships are handled by Backo rather than by database-native foreign keys.

Deletion and reference population behavior can also be configured through `DeleteStrategy` and `FillStrategy`.

From an API consumer's perspective, the underlying storage technology is transparent.

## Unified Data Access

Backo provides a common interface for manipulating entities across databases. Its `SFilter` mechanism offers a consistent filtering syntax, translated into native queries when supported by the connector.

Backo also supports existing database integration through `Transformer` and the extension of remote data sources through `DBExtenderConnector`.

## Benefits

- **Polyglot persistence:** combine SQL, NoSQL, files and external APIs.
- **Cross-database relationships:** connect entities independently of their storage.
- **Legacy integration:** reuse existing databases within a unified application.
- **Storage independence:** keep business logic separate from persistence technologies.
- **Unified RESTful API:** expose heterogeneous data through a consistent interface.

Backo turns independent storage systems into a unified application-level data model, without requiring all business entities to share the same physical database.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}