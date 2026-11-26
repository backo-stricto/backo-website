+++
title = "Authentication"
description = "Authentication and `current_user`."
weight = 15
+++

## Overview

Backo separates authentication from authorization. Authentication establishes the identity of the connected user, while authorization determines which resources, fields and business operations that user can access.

{% <devlab.callout type="info" title="You have to code"> %}
Backo does not impose a specific authentication mechanism. The application is responsible for authenticating users and populating `current_user`.
{% </devlab.callout> %}

JWT (JSON Web Token) provides a convenient solution for implementing authentication in a stateless RESTful API.

## Authentication Workflow

{% <devlab.diagram id="auth" label="Authentication process" description="Authentication steps process."> %}
flowchart LR
    login[ /login ]
    route[At each route]
    setjwt[ set jwt ]
    decjwt[ decode jwt ]
    set[ current_user.set ]
    logout[ /logout ]
    dropjwt[drop jwt]
    login --> setjwt
    route --> decjwt
    decjwt --> set
    logout --> dropjwt
{% </devlab.diagram> %}

## Generating the JWT Token

Here is an example of a login endpoint that verifies user credentials and generates a signed token containing the user identifier with an expiration date.

```python
import os
import jwt
from datetime import datetime, timezone, timedelta
from flask import Flask, request, jsonify

app = Flask(__name__)
JWT_SECRET = os.environ["JWT_SECRET"]

@app.post("/login")
def login():
    data = request.get_json()

    user = load_from_db_by_login(data["login"])

    if not user or not user.check_password(data["password"]):
        return jsonify({"message": "Invalid credentials"}), 401

    token = jwt.encode(
        {
            "user_id": str(user._id),
            "exp": datetime.now(timezone.utc)
                   + timedelta(hours=1)
        },
        JWT_SECRET,
        algorithm="HS256"
    )

    return jsonify({"token": token})
```

At this stage, `current_user` is not used. The authentication endpoint simply establishes the user's identity and issues a token.

## Token Verification and User Context

For each protected API request, an authentication decorator verifies the JWT signature and expiration date, retrieves the corresponding user and initializes Backo's user context.

```python
from functools import wraps
from backo import current_user

def check_user_token(f):
    @wraps(f)
    def decorated(*args, **kwargs):

        authorization = request.headers.get(
            "Authorization", ""
        )

        if not authorization.startswith("Bearer "):
            return jsonify({"message": "Missing token"}), 401

        token = authorization[7:]

        try:
            data = jwt.decode(
                token,
                JWT_SECRET,
                algorithms=["HS256"],
                options={"require": ["exp", "user_id"]}
            )
        except jwt.InvalidTokenError:
            return jsonify({"message": "Invalid token"}), 401

        user = load_from_db_by_id(data["user_id"])

        if not user:
            return jsonify({"message": "Unknown user"}), 401

        current_user.set({
            "_id": str(user._id),
            "login": user.login,
            "roles": list(user.roles)
        })

        return f(*args, **kwargs)

    return decorated
```

Loading roles from the database on each request allows authorization changes to take effect independently of the token's expiration date.

## Integration with Backo

The authentication decorator is passed to Backo when generating the RESTful API routes.

```python
backoffice.build_routes(app, "", check_user_token)
```

Protected routes can therefore benefit from a common authentication mechanism without requiring individual authentication controllers.

## The `current_user` Object

`current_user` is an extensible Stricto object representing the authenticated user.

| Attribute or method | Description |
|---|---|
| `_id` | Unique user identifier. |
| `login` | User login. |
| `roles` | List of assigned roles. |
| `has_role(role)` | Checks role membership. |
| `set(data)` | Populates the user context. |
| `reset()` | Allows customization of the underlying user object type. |


## Calling the RESTful API

**Authenticate and retrieve a token:**

```bash
TOKEN=$(curl -s -X POST \
  http://localhost/login \
  -H 'Content-Type: application/json' \
  -d '{"login":"alice","password":"example"}' \
  | jq -r '.token')
```

**Access a protected resource:**

```bash
curl -X GET \
  http://localhost/myApp/documents/DOC001 \
  -H "Authorization: Bearer ${TOKEN}"
```

## Conclusion

{% <devlab.callout type="tip" title="Fast feedback"> %}
By separating authentication from authorization, Backo allows developers to integrate their own identity management systems while providing centralized, declarative access control.
{% </devlab.callout> %}


The combination of JWT verification, `current_user.set()`, automatic route protection and dynamic authorization provides a flexible foundation for secure backend applications.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}