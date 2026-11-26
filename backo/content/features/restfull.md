+++
title = "RESTFul"
description = "Automatic RESTful API Generation."
weight = 2
+++

## Overview

Backo is an open-source Python framework that automatically generates RESTful APIs from declarative data models. By combining its database-agnostic ORM with Flask, Backo provides a unified approach to backend development, reducing the amount of repetitive code required to expose, manipulate, validate and secure application data.

The fundamental principle is to define the data model once and use it as the foundation for the entire API.

## Model-Driven API Generation

Backo relies on three main components:

- `Item`: defines the structure, types, constraints and behavior of an entity.
- `Collection`: associates a data model with a persistence mechanism.
- `Backoffice`: centralizes collections and generates the corresponding HTTP routes.

Once the collections are registered, Backo generates the API routes through its `build_routes()` method and integrates them into a Flask application.

```python
from flask import Flask
from backo import Backoffice

app = Flask(__name__)

backoffice = Backoffice("my_application")
backoffice.register_collection(books)
backoffice.build_routes(app)

app.run()
```

## Standard Operations

Backo exposes resources through standard HTTP methods, covering the complete CRUD lifecycle.

| HTTP Method | Operation | Purpose |
|---|---|---|
| `GET` | Read | Retrieve resources |
| `POST` | Create | Create new resources |
| `PATCH` | Update | Partially modify resources |
| `DELETE` | Delete | Remove resources |

Beyond CRUD operations, Backo supports advanced filtering, sorting, pagination, dedicated validation routes and multipart file handling.

## Request Processing Architecture


{% <devlab.diagram id="rpa" label="Request Processing Architecture" description="The request process."> %}
sequenceDiagram
    participant client as Web client
    participant backo as Backoffice
    participant db@{ "type" : "collections" } as Database    
    client ->>backo: http request (GET/POST/UPDATE/DELETE)
    activate backo
    backo -->>db: Get / insert / update / delete
    db -->> backo: Data
    backo -->>db: Get / insert / update / delete
    db -->> backo: Data
    backo->>client  : http response (Datas)
    deactivate backo
{% </devlab.diagram> %}


{% <devlab.diagram id="rpa" label="Request Processing Architecture" description="The request process."> %}
flowchart LR
    A[HTTP Client] --> B[Flask]
    B --> C[Backo API Routes]
    C --> D[Security and Validation]
    D --> E[Collection and Item]
    E --> F[Database Connector]
    F --> G[(Database)]
{% </devlab.diagram> %}

Backo connects HTTP requests to its ORM, centralizing the processing of incoming data, access control, business constraints, persistence and error handling.

This architecture keeps the API independent of the underlying database technology.

## Validation and Security

The API directly benefits from the rules defined in the data model. Developers can describe field constraints, conditional attributes, computed values and access permissions using Python.

Backo provides role-based access control (RBAC) and granular permissions at object and field levels. Authentication remains the application's responsibility, with JWT-based integration into Backo.

This approach ensures that API behavior is closely aligned with the application's business rules.

## Automatic OpenAPI Documentation

Backo automatically produces an OpenAPI 3.1 specification based on the application definition.

This specification facilitates API discovery, interactive documentation, client generation and integration with external applications.

The API contract is derived from the Python code rather than maintained as a separate configuration.
