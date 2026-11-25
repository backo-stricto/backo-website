+++
title = "ORM"
description = "Object-Relational Mapping."
weight = 1
+++

## Overview

Backo is an open-source Python framework that provides a storage-independent Object-Relational Mapping (ORM) layer combined with low-code backend development capabilities.

Unlike traditional ORMs, which primarily target SQL databases, Backo introduces a relational abstraction over different storage technologies, including NoSQL databases and file-based storage. It decouples the logical data model from its physical representation.

## Declarative Data Modeling

Backo relies on the Stricto library to describe data structures declaratively. Each entity is defined through an `Item`, which specifies its attributes, types, constraints and default values.

An `Item` represents both a data model and an active persistent object. It provides methods to manipulate its content and manage its persistence lifecycle.

A `Collection` associates an entity definition with a storage connector, establishing the mapping between the application data model and its persistence layer.

## Relationship Management

One of Backo's fundamental ORM characteristics is its ability to express relationships independently of the underlying storage engine.

Relationships are defined using two dedicated field types:

- `Ref`: a reference to a single entity.
- `RefsList`: a reference to multiple entities.

Their combination allows developers to define standard relational cardinalities:

| Relationship | Definition |
|---|---|
| One-to-one | `Ref` ↔ `Ref` |
| One-to-many | `Ref` ↔ `RefsList` |
| Many-to-many | `RefsList` ↔ `RefsList` |

Backo also supports bidirectional references, automatic reference resolution and configurable deletion policies. Related entities can be preserved, unlinked or deleted according to the declared relationship constraints.

This brings relational modeling capabilities to storage engines that do not necessarily provide them natively.

## Persistence Abstraction

Backo separates data manipulation from storage implementation through database connectors.

Entities expose a unified persistence interface, including operations such as `create()`, `load()`, `save()`, `reload()`, `delete()` and `select()`.

The application manipulates entities without having to directly implement database-specific operations.

```text
          Application
               |
          Backo ORM
               |
    Items / Relationships
               |
          Collections
               |
      Database Connectors
          /         \
     Database 1    Database 2
```

This architecture makes the logical data model independent of the physical storage technology.

## Beyond Traditional ORM

Backo extends the responsibilities of a conventional ORM by integrating additional backend capabilities:

- Automatic REST API generation.
- Declarative data validation.
- Custom business actions.
- Named selections and filtering.
- Fine-grained access control.
- File and binary object management.

The data model becomes the central definition of the backend application, reducing duplication between persistence, validation, authorization and API layers.
