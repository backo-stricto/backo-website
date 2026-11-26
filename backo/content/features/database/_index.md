+++
title = "Databases"
description = "Database Management and Integration."
template = "docs.html"
page_template = "doc-page.html"
sort_by = "weight"
weight = 10
+++

## Database-Agnostic Architecture

Backo provides a unified persistence layer that decouples application data models from their underlying storage systems.

Each `Collection` has its own `DBHandler`, independently of other collections. This allows a single application to combine relational databases, document databases, file-based storage, in-memory databases and external REST APIs.

The `DBHandler` acts as middleware between Backo and the storage system, exposing a common interface for creating, retrieving, updating, deleting and selecting objects.

## Available Database Connectors

Backo provides several built-in database connectors:

| Connector | Storage system |
|---|---|
| `DBMongoConnector` | MongoDB |
| `DBSqlite3Connector` | SQLite3 |
| `DBPostgresConnector` | PostgreSQL |
| `DBValkeyConnector` | Valkey / Redis |
| `DBYmlConnector` | Single YAML file |
| `DBYmlDirConnector` | Directory containing one YAML file per item |
| `DBRestFullConnector` | External REST API |
| `DBExtenderConnector` | Combination of two database handlers |

This architecture makes storage selection a collection-level decision rather than an application-wide constraint.

## Multi-Database Architecture

{% <devlab.diagram id="mda" label="Multi-Database Architecture" description="Multi-Database Architecture."> %}
flowchart TD
    subgraph application
        flask@{ shape: div-rect, label: "flask" }
        subgraph backo
            backoffice@{ shape: div-rect, label: "backoffice" }
            coll1[collection 1]
            coll2[collection 2]
            coll3[collection 3]
            DBHandler1[ DBHandler ]
            DBHandler2[ DBHandler ]
            DBHandler3( DBRestFull )
        end
        DB1[(Database 1)]
    end
    client1[ client ] -. GET\nPOST\nPATCH\nDELETE .-> flask
    client2[ client ] -. GET\nPOST\nPATCH\nDELETE .-> flask
    flask <==> backoffice
    backoffice <==>|"get_by_id()\ncreate()\nselect()\nupdate()"| coll1
    backoffice <==> coll2
    backoffice <==> coll3
    coll1 <==>|"get()\ncreate()\nselect()\nsave()"| DBHandler1
    DBHandler1 <-->|"SELECT * FROM WHERE"| DB1
    coll2 <==> DBHandler2
    DBHandler2 <-->|"coll.find().sort()"| DB2[(Database 2)]
    coll3 <==> DBHandler3
    DBHandler3 <-..->|GET\nPOST\nPATCH\nDELETE| DB3[external API]
{% </devlab.diagram> %}

Relationships between collections are expressed through `Ref` and `RefsList`, independently of their respective storage systems. Backo manages relational consistency at the application level, including configurable deletion and reference policies.

## Database Queries and Schema Management

Backo uses `SFilter` to express storage-independent filtering operations.

MongoDB, SQLite and PostgreSQL connectors translate these expressions into native database queries. For connectors without native filtering support, such as YAML and Valkey, filtering is performed by Backo itself.

SQL connectors also provide a `check_structure()` method to identify discrepancies between the expected data model and the physical database schema, and optionally apply structural changes.

## Integration with Existing Information Systems

Backo can integrate existing databases without requiring their structures to match the application's logical model.

The `Transformer` mechanism provides an adaptation layer between both representations:

- `RenameTransformer`: maps different field names or paths.
- `IgnoreTransformer`: excludes unwanted attributes.
- Custom transformers: implement application-specific data transformations.

Transformations can also be reflected in query generation, allowing applications to use logical field names while targeting existing database structures.

## Extending External Data Sources

The `DBExtenderConnector` allows a collection to enrich data originating from an external system with additional locally stored attributes.

For example, an application can retrieve country information from an external REST API and add local business attributes stored in MongoDB.

Both sources are exposed as a single logical collection, with combined filtering capabilities, without modifying the original external system.

## Extensibility

Developers can implement custom database connectors by extending `DBHandler` and implementing its persistence contract.

This contract uses Python dictionaries to represent objects and string identifiers to identify them, making it possible to integrate new storage technologies without changing Backo's core architecture.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}