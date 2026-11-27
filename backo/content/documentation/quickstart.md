+++
title = "Quickstart"
description = "Build your first Backo application with `backo_init`."
weight = 1


+++


## Installation

You need Python 3.12 or later and a terminal. Create an empty project directory and a virtual environment:

```bash
mkdir book_catalog
cd book_catalog
python3 -m venv .venv
source .venv/bin/activate
pip install backo
```

On Windows PowerShell, activate the environment with `.venv\Scripts\Activate.ps1`. The generated YAML storage path may also need adapting on Windows.

Optional database dependencies can be installed with `pip install 'backo[mongo]'` or `pip install 'backo[postgres]'`.

## Step 1 - Generate the application

Run the interactive initializer:

```bash
backo_init
```

Enter the following values when prompted:

| Prompt | Answer |
| --- | --- |
| Name of the application | `book_catalog` |
| Add a new collection | Yes |
| Name of the collection | `books` |
| Add a new field | Yes |
| Name of the field | `title` |
| Type of the field | `String` |
| Add a new field | Yes |
| Name of the field | `author` |
| Type of the field | `String` |
| Add a new field | Yes |
| Name of the field | `pages` |
| Type of the field | `Int` |
| Add another field | No |
| Add another collection | No |
| Is this configuration ok? | Yes |

The initializer displays the configuration before generating files. The prompt labels above are abbreviated for readability.

## Step 2 - Explore the generated project

```text
book_catalog/
├── backoffice.py
├── main.py
├── migrate.py
└── collections_set/
    ├── __init__.py
    └── books.py
```

| File | Purpose |
| --- | --- |
| `backoffice.py` | Creates the `Backoffice`, registers collections and checks their syntax. |
| `main.py` | Creates the Flask application, implements demo JWT authentication and builds API routes. |
| `migrate.py` | Provides a command-line scaffold for checking and migrating existing data. |
| `collections_set/books.py` | Defines the book model, database connector and collection permissions. |
| `collections_set/__init__.py` | Exports the generated collections. |

## Step 3 - Understand the implementation

The following excerpts show the essential parts of the generated code, with logging and permission callbacks omitted for clarity. Keep the actual generated files for this tutorial.

### Define the collection

In `collections_set/books.py`:

```python
from backo import Collection, Item, String, Int
from backo.db import DBYmlDirConnector

books_item = Item({
    "title": String(),
    "author": String(),
    "pages": Int(),
})

connector = DBYmlDirConnector("/tmp/book_catalog/books")

books = Collection("books", books_item, connector)
```

`Item` describes the data model, `Collection` associates it with a storage connector, and `DBYmlDirConnector` stores each item in a separate YAML file. The generated permission callbacks allow creation, modification and deletion by default.

### Register the collection

The essential code in `backoffice.py` is:

```python
from backo import Backoffice
from collections_set import books

my_backoffice = Backoffice("book_catalog")
my_backoffice.register_collection(books)
my_backoffice.check_syntax()
```

### Generate REST routes

The generated `main.py` creates a Flask application and registers the routes using:

```python
my_backoffice.build_routes(flask, "", check_user_token)
```

The `check_user_token` callback validates the JWT stored in the `jwt_token` cookie and populates Backo's `current_user` object. You do not need to implement CRUD routes manually.

## Step 4 - Run the application

For a local tutorial, change the generated `flask.run` host from `0.0.0.0` to `127.0.0.1` in `main.py`:

```python
if __name__ == "__main__":
    flask.run(host="127.0.0.1", port=5000)
```

Start the server:

```bash
python main.py
```

The API is now available under `http://127.0.0.1:5000/book_catalog`.

## Step 5 - Example workflow

Open another terminal in the project directory. The following commands use a POSIX-compatible shell and curl.

### Log in

```bash
BASE=http://127.0.0.1:5000
API="$BASE/book_catalog"

curl -c cookies.txt -X POST "$BASE/login" \
  -H 'Content-Type: application/json' \
  -d '{"login":"demo","password":"demo"}'
```

The generated login route returns `{"login":"ok"}` and sets a JWT cookie. It does **not** verify credentials: this is a development scaffold, not real authentication.

### Create a book

```bash
curl -b cookies.txt -X POST "$API/books" \
  -H 'Content-Type: application/json' \
  -d '{"title":"The Hobbit","author":"J.R.R. Tolkien","pages":310}'
```

Copy the `_id` from the returned object and set it in your shell:

```bash
BOOK_ID='<returned _id>'
```

### List and filter books

```bash
curl -b cookies.txt "$API/books"
curl -b cookies.txt --get "$API/books" \
  --data-urlencode 'title=The Hobbit'
```

### Read and update a book

```bash
curl -b cookies.txt "$API/books/$BOOK_ID"

curl -b cookies.txt -X PUT "$API/books/$BOOK_ID" \
  -H 'Content-Type: application/json' \
  -d '{"pages":320}'
```

### Delete a book

```bash
curl -b cookies.txt -X DELETE "$API/books/$BOOK_ID"
```

### Export metadata and OpenAPI

```bash
curl "$API/_meta"
curl "$API/_openapi" -o openapi.json
```

### Log out

```bash
curl -b cookies.txt -c cookies.txt "$BASE/logout"
```

## Generated API endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/login` | Issue a demo JWT cookie. |
| GET | `/logout` | Clear the JWT cookie. |
| GET | `/book_catalog/books` | List or filter books. |
| POST | `/book_catalog/books` | Create a book. |
| GET | `/book_catalog/books/{_id}` | Read a book. |
| PUT | `/book_catalog/books/{_id}` | Update a book. |
| DELETE | `/book_catalog/books/{_id}` | Delete a book. |
| GET | `/book_catalog/_meta` | Export application metadata. |
| GET | `/book_catalog/_openapi` | Export the OpenAPI specification. |

Collection routes require the JWT cookie in the generated application. The metadata and OpenAPI routes are not protected by the collection authentication callback.

## Going further with `backo_init`

```bash
backo_init --help
backo_init --expert
backo_init --template_dir /path/to/templates /path/to/project
```

Expert mode adds prompts for required fields, default values, selections and actions. Selections define field paths; actions generate function stubs for custom business logic. In the supplied 0.4.1 templates, default values entered in the wizard are not rendered into the field definitions, so add them manually when needed. The `Datetime` and `List` types are offered by the wizard but are not imported by the default collection template; add the corresponding imports if you use them.

The optional destination directory must already exist. The template directory option selects custom Jinja templates. Run the initializer in an empty directory: it writes generated files directly and can overwrite existing files.

To check existing data with the generated migration script:

```bash
python migrate.py --collection books --mode dry_run
```

## Why Backo?

- Interactive project scaffolding with `backo_init`.
- Typed data models and validation based on Stricto.
- Automatic REST routing and OpenAPI generation.
- Replaceable database connectors.
- Extensible permissions, actions, selections and migrations.

## Production considerations

Replace the demo login with real credential verification, remove hard-coded secrets, implement authorization rules, secure cookies and use HTTPS. Protect metadata endpoints if necessary. Use durable storage rather than `/tmp`, and deploy Flask behind a production WSGI server. Review generated code before deployment.

## References

- [Backo documentation](https://backo.readthedocs.io/en/latest/index.html)
- [Quickstart](https://backo.readthedocs.io/en/latest/Quickstart.html)
- [Routing](https://backo.readthedocs.io/en/latest/Routing.html)
- [Initializer source](https://github.com/backo-stricto/backo/blob/main/backo/initialisation/init_script.py)
- [Project templates](https://github.com/backo-stricto/backo/tree/main/backo/initialisation/templates)
