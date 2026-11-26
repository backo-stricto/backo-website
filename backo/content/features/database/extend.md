+++
title = "Extended Databases"
description = "Extending Data Sources with `DBExtenderConnector`."
weight = 10
+++

## Overview

Backo provides `DBExtenderConnector`, a database connector designed to enrich an existing data source with additional application-specific attributes without modifying its original structure.

Unlike a conventional database connector, which interacts with a single storage system, `DBExtenderConnector` combines two database handlers to expose their data through a single logical collection.

This mechanism is particularly useful when integrating external REST APIs, legacy databases or shared information systems whose data structures cannot be modified.

## Architecture

The connector combines two complementary data sources:

- **Primary data source:** provides the original entity and its existing attributes.
- **Extension data source:** stores additional attributes required by the application.

Backo combines both representations into a single logical object, using the entity identifier to associate the corresponding records.

{% <devlab.diagram id="mda" label="Extended Database" description="Extend Database Model and Architecture."> %}
flowchart TD
    API[RESTful API] --> C[Collection]
    C --> E[DBExtenderConnector]

    E --> P[Primary DBHandler]
    E --> X[Extension DBHandler]

    P --> R[External REST API]
    X --> M[(MongoDB)]
{% </devlab.diagram> %}

Both data sources can rely on different storage technologies.

## Example: Enriching a Country Database

Consider an application that retrieves country information from an external REST API. The application needs to associate additional commercial information with each country without modifying the external service.

### Primary data source — External REST API

```json
{
    "_id": "FRA",
    "name": "France",
    "capital": "Paris",
    "region": "Europe"
}
```

### Extension data source — MongoDB

```json
{
    "_id": "FRA",
    "market_priority": "high",
    "annual_target": 250000,
    "commercial_notes": "Strategic European market"
}
```

### Unified Backo Data Model

```python
from backo import Item, String, Float

country = Item({
    "name": String(),
    "capital": String(),
    "region": String(),
    "market_priority": String(
        enum=["low", "medium", "high"],
        default="medium"
    ),
    "annual_target": Float(min=0),
    "commercial_notes": String()
})
```

The collection uses a `DBExtenderConnector` combining a `DBRestFullConnector` for geographical information and a `DBMongoConnector` for commercial information.

### Resulting Logical Entity

Backo exposes the combined data as a single resource:

```json
{
    "_id": "FRA",
    "name": "France",
    "capital": "Paris",
    "region": "Europe",
    "market_priority": "high",
    "annual_target": 250000,
    "commercial_notes": "Strategic European market"
}
```

The application can manipulate this unified representation without having to manually retrieve and merge records from both systems.

## Filtering and RESTful API Integration

The extended collection benefits from Backo's unified data access mechanisms, including filtering across the combined data model.

For example, an application can select countries located in Europe whose commercial priority is high, even though these attributes originate from different storage systems.

The resulting collection can also be exposed through Backo's automatically generated RESTful API, making the underlying data distribution transparent to API consumers.

## Benefits

- **Non-invasive integration:** enrich existing data without modifying its original schema.
- **Data ownership:** maintain application-specific information in a dedicated database.
- **Storage independence:** combine SQL, NoSQL, files and external REST APIs.
- **Unified data modeling:** manipulate distributed attributes through a single `Item`.
- **Legacy modernization:** extend existing information systems without restructuring them.
- **Reduced boilerplate:** avoid implementing dedicated data aggregation mechanisms.

`DBExtenderConnector` transforms existing data sources into extensible application resources, allowing Backo to build new business capabilities on top of existing information systems.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}