+++
title = "Files management"
description = "Files and storage of files."
weight = 20
+++

## File Management with Backo

Backo integrates file management directly into its data model. Files can be declared, validated, stored and manipulated like any other field, without implementing dedicated upload or download controllers.

## Storage Strategies

Backo provides two file types:

| Type | Description |
|---|---|
| `BlobFile()` | Embeds binary content directly into the data structure. Suitable for small files. |
| `File()` | Delegates binary storage to a dedicated `FileConnector`, independently of the collection's database. |

Files support standard field options, MIME type restrictions, maximum size validation and automatic metadata management, including filename, content type, size and unique identifier.

Backo also supports two storage locations: a working connector for file processing and an optional permanent storage connector.

## Example: Author Profile Picture

An author model can include a mandatory picture stored on the filesystem:

```python
author = Item({
    "name": String(require=True),
    "surname": String(),
    "picture": File(
        require=True,
        mime_types=["image/jpeg", "image/png"],
        max_size=2 * 1024 * 1024,
        work=FileSystemConnector(path="/data/pictures")
    )
})
```

Replacing `File()` with `BlobFile()` allows the picture to be embedded directly into the stored entity.

## RESTful API Integration

Backo supports file transfers through JSON using text or Base64 encoding, and through multipart requests combining structured data and binary content.

**Create an author with a picture:**

```bash
curl -X POST \
  -F '_json={"name":"Victor","surname":"Hugo"}' \
  -F picture=@portrait.png \
  http://localhost/myApp/authors/
```

**Retrieve the picture:**

```http
GET /myApp/authors/<id>/picture
```

File handling is available across standard CRUD operations and custom actions.

By combining declarative file modeling, configurable storage, validation and automatic RESTful API integration, Backo provides a unified approach to managing structured data and binary content.