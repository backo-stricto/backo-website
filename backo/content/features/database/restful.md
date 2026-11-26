+++
title = "REST API connector"
description = "Use an external REST API as a local database."
weight = 4
+++

## Overview

Backo provides the `DBRestFullConnector` to integrate external RESTful APIs as native data sources. Instead of directly accessing a database, a collection can retrieve and manipulate resources exposed by a remote HTTP service.

From the application's perspective, an external API becomes another persistence backend, alongside MongoDB, PostgreSQL, SQLite or YAML files.

## Integration Architecture

{% <devlab.diagram id="rest" label="RESTFul API Database" description="RESTFul API Database Architecture."> %}
flowchart LR
    A[Backo REST API] --> B[Collection]
    B --> C[DBRestFullConnector]
    C <-->|HTTP / JSON| D[External REST API]
{% </devlab.diagram> %}

The connector encapsulates communication with the remote service, while the collection provides a unified object model based on Backo and Stricto.

## Example: External Product Catalog

Consider an application that retrieves products from an external inventory management system exposing the following REST endpoints:

```http
GET    /products/
GET    /products/{id}
POST   /products/
PATCH  /products/{id}
DELETE /products/{id}
```

By associating a `products` collection with a `DBRestFullConnector`, Backo can represent these remote resources as application entities.

Additional capabilities can be introduced at the application level:

- **Data modeling and validation:** describe remote entities using Stricto fields and constraints.
- **Data transformation:** adapt remote data structures to the application's logical model.
- **Cross-database relationships:** associate remote resources with locally stored entities using `Ref` and `RefsList`.
- **Access control:** define application-specific authorization policies.
- **Data enrichment:** combine remote resources with local attributes using `DBExtenderConnector`.

The operations available depend on the capabilities and permissions of the external API.

## Benefits

`DBRestFullConnector` facilitates the integration of microservices, legacy applications and third-party APIs without requiring data replication into a local database.

Backo therefore acts as a unifying layer over distributed information systems, treating remote RESTful resources as first-class components of its object model.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}