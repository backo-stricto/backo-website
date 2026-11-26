+++
title = "Validation"
description = "Interactive Field Validation."
weight = 16
+++

## Interactive Field Validation

Backo automatically exposes a dedicated `_check` endpoint to validate individual fields without persisting the object.

{% <devlab.callout type="info" title="No client validation"> %}
The purpose of this feature is to avoid duplicate the business logic (and code) to the client.
{% </devlab.callout> %}



```http
POST /<app_name>/<collection>/_check
```

The request body contains two parameters:

| Parameter | Description |
|---|---|
| `item` | Complete or partial object containing the data to validate. |
| `path` | Stricto selector identifying the field to validate. |

### Invalid Value

```bash
curl -X POST \
  'http://localhost/inventory/products/_check' \
  -H 'Content-Type: application/json' \
  -d '{
    "item": {"name": 123},
    "path": "$.name"
  }'
```

Response:

```json
{
    "error": "Must be a string"
}
```

### Valid Value

```bash
curl -X POST \
  'http://localhost/inventory/products/_check' \
  -H 'Content-Type: application/json' \
  -d '{
    "item": {"name": "Network Switch"},
    "path": "$.name"
  }'
```

Response:

```json
{
    "error": null
}
```

Both requests return HTTP status `200`, even when a validation error is detected. The `error` attribute indicates whether the selected field satisfies the model's constraints.

Only the targeted field is checked. This allows frontend applications to validate individual form inputs while the user is editing an incomplete object.

This mechanism enables real-time server-side form validation without duplicating business constraints in frontend code or creating temporary database records.

{% <devlab.cards> %} 
{% <devlab.card title="Backo documentation" icon="external-link" tag="External" href="https://backo.readthedocs.io/en/latest/"> %}
Full Backo documentation on readthedocs.
{% </devlab.card> %}
{% </devlab.cards> %}