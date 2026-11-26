+++
title = "Media Library"
description = "A media library needs to manage users, books and loans. Administrators manage roles, employees register books and record loans, and readers can inspect their own records. The code allows administrators as well as employees to create books, and uses field-level permissions to control borrower visibility."
weight = 1
+++


## Installation

Python 3.12 or later is required. From the root of the supplied Backo checkout:

```bash
pip install backo[mongo]
```

## Implementation

Keep the original directory layout. The following files reproduce the repository source without changes.

### `backoffice.py`

```python
"""
backoffice : The main application
"""

# linter is lost with different modules with the same name
# pylint: disable=no-name-in-module

import json

from random import choice
from string import ascii_uppercase

import sys
from functools import wraps
from datetime import datetime, timezone, timedelta
import jwt
from flask import Flask, request, jsonify, make_response, Response

sys.path.insert(1, "../../../stricto")
sys.path.insert(1, "../../../backo")


from collections_set import books, users
from backo import Backoffice, current_user, log_system, LogLevel, SFilter, Operator

log_system.add_handler(log_system.set_streamhandler())
log_system.setLevel(LogLevel.ERROR)


# set the flask application route
flask = Flask("my_media_library")
flask.secret_key = "super secret key"

JWTOKEN_SECRET_KEY = "".join(choice(ascii_uppercase) for i in range(64))


# --- Set the login route -----
@flask.route("/login", methods=["POST"])
def log_in():
    """check the login and if OK return a jwt token populated with the user data for current_user"""
    d = request.json
    login = d["login"]
    password = d["password"]

    current_user.standalone = True

    # find the user by login in the db
    user = users.select_one(SFilter("$.login", Operator.EQ, login))

    # Fake auth
    if login != password:
        return jsonify({"message": "Invalid login or password"}), 401

    if user is None:
        user = users.create({"login": login, "roles": ["USER"]})
    current_user.standalone = False

    token = jwt.encode(
        {
            "exp": datetime.now(timezone.utc) + timedelta(hours=1),
            "user": {
                "_id": user._id.get_value(),
                "login": user.login.get_value(),
                "roles": user.roles.get_value(),
            },
        },
        JWTOKEN_SECRET_KEY,
        algorithm="HS256",
    )
    response = make_response(json.dumps({"login": "ok"}))
    response.set_cookie("jwt_token", token)
    return response


def check_user_token() -> None | Response:
    """
    Decode the jwt token and set current_user.
    """
    token = request.cookies.get("jwt_token")
    if not token:
        return jsonify({"message": "Token is missing!"}), 401
    try:
        data = jwt.decode(token, JWTOKEN_SECRET_KEY, algorithms=["HS256"])
    except:  # pylint: disable=bare-except
        return jsonify({"message": "Token is invalid!"}), 401
    current_user.set(data["user"])
    return None


def token_required(f):
    """Populate current_user with data found in the jwt token"""

    @wraps(f)
    def decorated(*args, **kwargs):
        error_message = check_user_token()
        if error_message is not None:
            return error_message
        return f(*args, **kwargs)

    return decorated


@flask.route("/ping")
@token_required
def ping():
    """
    just a ping
    """
    response = make_response(
        json.dumps({"pong": True, "current_user": current_user.get_value()})
    )
    return response


@flask.route("/logout")
@token_required
def logout():
    """The logout
    clear the jwt in cookie
    """
    current_user.logout()
    response = make_response(json.dumps({"logout": True}))
    response.delete_cookie("jwt_token")
    return response


myapp = Backoffice("media_library")
myapp.register_collection(books)
myapp.register_collection(users)

# To check if you have some syntax issue
myapp.check_syntax()
myapp.build_routes(flask, "", check_user_token)


# ------------------------------------
# Initialisation
# ------------------------------------
current_user.standalone = True
current_user.roles.append("ADMIN")

users.drop()
books.drop()

admin_user = users.select_one(SFilter("$.login", Operator.EQ, "admin"))
if admin_user is None:
    users.create({"login": "admin", "roles": ["ADMIN", "USER"]})

emp1_user = users.select_one(SFilter("$.login", Operator.EQ, "emp1"))
if emp1_user is None:
    users.create({"login": "emp1", "roles": ["EMPLOYEE", "USER"]})

client1_user = users.select_one(SFilter("$.login", Operator.EQ, "client1"))
if emp1_user is None:
    users.create({"login": "client1", "roles": ["USER"]})

first_book = books.select_one(
    SFilter("$.title", Operator.EQ, "martine mange des yaourth")
)
if first_book is None:
    books.create({"title": "martine mange des yaourth", "pages": 12})


current_user.standalone = False


if __name__ == "__main__":
    flask.run(host="0.0.0.0", port=5000)
```

### `__init__.py`

```python
"""backoffice as a module (used by tests.py)"""

from .backoffice import flask, myapp
```

### `collections_set/__init__.py`

```python
"""Module for all collections"""

from .books import books
from .users import users

__all__ = ["books", "users"]
```

### `collections_set/books.py`

```python
"""Books module"""

# pylint: disable=unused-argument

from datetime import datetime
from backo import (
    Int,
    Datetime,
    Bool,
    Dict,
    String,
    Collection,
    Item,
    current_user,
    log_system,
    Ref,
    Action,
    Selection,
    SFilter,
    Operator,
)
from backo.db import DBMongoConnector

log = log_system.get_or_create_logger("books")


# ------------------------------------------------
# ITEM
# ------------------------------------------------


def set_borrowed(book: Item) -> bool:
    """compute if the book is currently borrowed

    :param book: the current book
    :type book: Item
    :return: borrowed or not
    :rtype: bool
    """
    if book.borrow is None:
        return False
    if book.borrow.return_date == None:  # pylint: disable=singleton-comparison
        return False
    if book.borrow.return_date > datetime.now().replace(microsecond=0):
        return True
    return False


def can_read_borrow_user(right_name: str, book: Item) -> bool:
    """Tel if current_user can read the name of the personne who borrow the book

    :param right_name: The name of the right (here ="read")
    :type right_name: str
    :param book: the book
    :type book: Item
    :return: True if current_user can read
    :rtype: bool
    """
    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True
    # print(f'book borrewed = {book}')
    if book.borrow.user._id == current_user._id:
        return True
    return False


def can_modify_borrow(right_name: str, book: Item) -> bool:
    """Tel if current_user can modify the borrow part of the book

    :param right_name: The name of the right (here ="modify")
    :type right_name: str
    :param book: The book
    :type book: Item
    :return: True if can modify
    :rtype: bool
    """
    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True

    if book.borrowed is False:
        return True

    return False


# --------------------
# Description of "what is a book"
# --------------------
books_item = Item(
    {
        "title": String(require=True, default=""),
        "pages": Int(),
        "borrowed": Bool(set=set_borrowed, views=["save"]),
        "borrow": Dict(
            {
                "user": Ref(
                    coll="users",
                    field="$.rent.books",
                    can_read=can_read_borrow_user,
                ),
                "return_date": Datetime(),
                "date": Datetime(),
            },
            can_modify=can_modify_borrow,
        ),
    }
)


# ------------------------------------------------
# COLLECTION
# ------------------------------------------------
connector = DBMongoConnector("mongodb://localhost:27017/media_library", "Books")


def can_create(right_name: str, book: Item) -> bool:
    """Check if can create a book"""

    log.debug(f"can create a book by {current_user.login} ? ")

    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True
    return False


def can_modify(right_name: str, book: Item) -> bool:
    """Check if can modify a book"""

    log.debug(f"can modify a book by {current_user.login} ? ")

    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True
    return False


def can_delete(right_name: str, book: Item) -> bool:
    """Check if can delete a book"""

    log.debug(f"can delete a book by {current_user.login} ? ")

    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True
    return False


books = Collection(
    "books",
    books_item,
    connector,
    can_create=can_create,
    can_modify=can_modify,
    can_delete=can_delete,
)


# ------------------------------------------------
# ACTIONS
# ------------------------------------------------


def borrow(action: Action, book: Item) -> None:
    """borrow the book"""
    book.borrow.user = action.user_id
    book.borrow.date = datetime.now().replace(microsecond=0)
    book.borrow.return_date = action.return_date
    book.save()


def can_borrow(right_name: str, book: Item) -> bool:
    """Tel if current_user can execute the borrow action

    :param right_name: The name of the right (here="execute")
    :type right_name: str
    :param book: The book
    :type book: Item
    :return: True if he can
    :rtype: bool
    """
    if current_user.has_role("EMPLOYEE"):
        return True
    return False


def can_see_borrow_action(right_name: str, book: Item) -> bool:
    """Tel the borrow action is a nonsense ?

    you can borrow a book which is not borrowed.

    :param right_name: The name of the right (here="see")
    :type right_name: str
    :param book: the book
    :type book: Item
    :return: True if borrowed field is false
    :rtype: bool
    """
    return not book.borrowed


#
# Definition of the action
borrow_action = Action(
    {
        "user_id": String(require=True, default=""),
        "return_date": Datetime(
            require=True, default=datetime.now().replace(microsecond=0)
        ),
    },
    borrow,
    can_execute=can_borrow,
    can_see=can_see_borrow_action,
)

# Add the action to the book collection
books.register_action("borrow", borrow_action)


# ------------------------------------------------
# SELECTIONS
# ------------------------------------------------
borrowed_book_select = Selection(
    ["$.title", "$.borrow.user.login"], filter=SFilter("$.borrowed", Operator.EQ, True)
)
books.register_selection("borrowed", borrowed_book_select)
```

### `collections_set/users.py`

```python
"""Users module"""

# pylint: disable=unused-argument

from backo import (
    String,
    List,
    Dict,
    Collection,
    Item,
    current_user,
    Action,
    log_system,
    RefsList,
)
from backo.db import DBMongoConnector

log = log_system.get_or_create_logger("user")


# ------------------------------------------------
# ITEM
# ------------------------------------------------
def can_modify_login(right_name: str, user: Item) -> bool:
    """Tel if the field "login" of the user can be modified

    :param right_name: The name of the right (here ="modify")
    :type right_name: str
    :param user: the user Item we want to modify login
    :type user: Item
    :return: True if current_user can modify the login
    :rtype: bool
    """
    if current_user.has_role(["ADMIN"]):
        return True
    return False


def can_modify_roles(right_name: str, user: Item) -> bool:
    """Tel if the roles field of the user can be modified

    :param right_name: The name of the right (here ="modify")
    :type right_name: str
    :param user: the user Item we want to modify roles
    :type user: Item
    :return: True if current_user can modify the roles field
    :rtype: bool
    """
    if current_user.has_role(["ADMIN"]):
        return True
    return False


# ------------------------------------
# Description of the Item
#
# The item is the object in the collection
# ------------------------------------
item = Item(
    {
        "login": String(require=True, can_modify=can_modify_login, default=""),
        "email": String(),
        "roles": List(
            String(union=["ADMIN", "EMPLOYEE", "USER"], default="USER"),
            can_modify=can_modify_roles,
        ),
        "rent": Dict({"books": RefsList(coll="books", field="$.borrow.user")}),
    }
)


# ------------------------------------------------
# COLLECTION
# ------------------------------------------------

# First define the connector  = where to store datas
connector = DBMongoConnector(
    connection_string="mongodb://localhost:27017/media_library", collection="Users"
)


def can_create(right_name: str, user: Item) -> bool:
    """Check if current_user can create a user


    :param right_name: The name of the right (here ="create")
    :type right_name: str
    :param user: the user Item we want to create
    :type user: Item
    :return: True if current_user can create this user
    :rtype: bool
    """

    log.debug(f"can create user by {current_user.login} ? ")

    if current_user.has_role("ADMIN"):
        return True

    # In this case, current_user.is_anonymous() is authorized to create a user
    # because anonymous user is used during auto creation in the login attempt
    # (if the user doesn't exist, he is created)
    if current_user.is_anonymous():
        return True
    return False


def can_read(right_name: str, user: Item) -> bool:
    """Check if current_user can read a user

    :param right_name: The name of the right (here ="read")
    :type right_name: str
    :param user: the user Item we want to read
    :type user: Item
    :return: True if current_user can read this user
    :rtype: bool
    """

    log.debug(f"can read user by {current_user.login} ? ")

    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True
    if user._id == current_user._id:
        return True

    # In this case, current_user.is_anonymous() is authorized to read a user
    # because anonymous user is used during auto creation in the login attempt
    # (must check if the user exist in the db before his creation)
    if current_user.is_anonymous():
        return True
    return False


def can_modify(right_name: str, user: Item) -> bool:
    """Check if current_user can modify a user"""

    log.debug(f"can modify user by {current_user.login} ? ")

    if current_user.has_role(["ADMIN", "EMPLOYEE"]):
        return True

    # I can modify myself
    if user._id == current_user._id:
        return True
    return False


def can_delete(right_name: str, user: Item) -> bool:
    """Check if current_user can delete a user"""

    log.debug(f"can delete user by {current_user.login} ? ")

    if current_user.has_role("ADMIN"):
        return True

    # auto_deletion is possible
    if user._id == current_user._id:
        return True

    return False


#
# The collection creation
#
# Association of Item, collector, rights
#
users = Collection(
    "users",
    item,
    connector,
    can_read=can_read,
    can_create=can_create,
    can_modify=can_modify,
    can_delete=can_delete,
)


# ------------------------------------------------
# ACTIONS
#
# Actions are a specific fuction with parameters we apply on an Item
# ------------------------------------------------
def toggle_role(action: Action, user: Item) -> None:
    """Change roles for a user

    :param action: The action object
    :type action: Action
    :param user: The user to apply the action
    :type user: Item
    """
    if action.role in user.roles:
        user.roles.remove(action.role)
    else:
        user.roles.append(action.role)
    # Save into the DB
    user.save()


def can_toggle_role(right_name: str, user: Item) -> bool:
    """check if current_user can execute or see this action

    :param right_name: The name of the right (here ="execute" or "see" )
    :type right_name: str
    :param user: the user on wich we want to see or execute the action
    :type user: Item
    :return: True if current_user can execute or see the action on this user
    :rtype: bool
    """
    if current_user.has_role("ADMIN"):
        return True
    return False


#
# Definition of the action
add_role_action = Action(
    {"role": String(union=["ADMIN", "EMPLOYEE", "USER"], default="USER")},
    toggle_role,
    can_execute=can_toggle_role,
    can_see=can_toggle_role,
)

# Add the action to the users collection
users.register_action("toggle_role", add_role_action)


# ------------------------------------------------
# SELECTIONS
#
# soon
# ------------------------------------------------
```

### `migration.py`

```python
"""
migration sample script


"""

import sys

sys.path.insert(1, "../../../backo")
sys.path.insert(1, "../../../stricto")
sys.path.insert(1, "../")


from media_library import myapp
from backo import log_system

# Set migration level to debug
log_migration = log_system.get_or_create_logger("migration")


# ---------------------------------------------
# Check if a data match the model
# it raise an error if not
# ---------------------------------------------
report = myapp.migrate("books", _id="my_book_id")
report = myapp.migrate("books", _ids=["my_book_id1", "my_book_id2"])
report = myapp.migrate("books")  # All ids


# --------------------------------------------
# Do a changement
# example you must now have a note field with a float value
# --------------------------------------------
def update_with_note(o: dict) -> dict:
    """
    this the function for doing operation on objects before setting them into the Item
    You can do what you want.
    """
    if "note" not in o:
        o["note"] = 10.0
    return o


# Check if OK (dry_run is True by default)
report = myapp.migrate("books", update_with_note, _id="my_book_id")
report = myapp.migrate("books", update_with_note, _ids=["my_book_id1", "my_book_id2"])
report = myapp.migrate("books", update_with_note)  # All ids

# do it for real
report = myapp.migrate("books", update_with_note, _id="my_book_id", dry_run=False)
report = myapp.migrate(
    "books", update_with_note, _ids=["my_book_id1", "my_book_id2"], dry_run=False
)
report = myapp.migrate("books", update_with_note, dry_run=False)  # All ids
```

### `tests.py`

```python
"""
backoffice tests : The main application test
"""

import unittest
import json
import sys
from datetime import datetime, timedelta

sys.path.insert(1, "../../../backo")
sys.path.insert(1, "../../../stricto")
sys.path.insert(1, "../")


from media_library import flask
from backo import log_system, LogLevel

log_system.setLevel(LogLevel.ERROR)


class TestBackoffice(unittest.TestCase):
    """
    Tests for this example.
    """

    def __init__(self, *args, **kwargs):
        """
        init this tests
        """
        super().__init__(*args, **kwargs)
        ctx = flask.app_context()
        ctx.push()
        self.client = flask.test_client()

    def login(self, login):
        """Do a login"""
        return self.client.post("/login", json={"login": login, "password": login})

    def logout(self):
        """Do a logout"""
        return self.client.get("/logout")

    def test_jwt_fail(self):
        """test jwt error"""
        self.logout()
        response = self.client.get("/ping")
        self.assertEqual(response.status_code, 401)

    def test_login_logout(self):
        """
        test login logout
        """
        self.logout()
        self.login("test")
        self.logout()
        response = self.client.get("/ping")
        self.assertEqual(response.status_code, 401)

    def test_emp1_cannot_create_a_user(self):
        """
        test error user creation
        """
        self.logout()
        response = self.login("emp1")
        response = self.client.post("/media_library/users", json={"login": "toto1"})
        self.assertEqual(response.status_code, 403)
        self.logout()

    def test_admin_create_users(self):
        """
        create a user
        """
        self.logout()
        response = self.login("admin")
        response = self.client.post("/media_library/users", json={"login": "toto1"})
        self.assertEqual(response.status_code, 200)
        response = self.client.post("/media_library/users", json={"login": "toto2"})
        self.assertEqual(response.status_code, 200)
        response = self.client.post("/media_library/users", json={"login": "toto3"})
        self.assertEqual(response.status_code, 200)
        response = self.client.post("/media_library/users", json={"login": "toto4"})
        self.assertEqual(response.status_code, 200)
        self.logout()

    def test_emp1_add_books(self):
        """
        test add a book
        """
        self.logout()
        response = self.login("emp1")
        response = self.client.post(
            "/media_library/books",
            json={"title": "martine a la plage 1", "pages": 21},
        )
        self.assertEqual(response.status_code, 200)
        d = json.loads(response.data)
        book_id = d["_id"]
        # self.logout()
        # self.login('test')
        # response=self.client.delete(f"/media_library/books/{book_id}")
        # self.assertEqual(response.status_code, 403)
        # self.logout()
        # self.login('emp1')
        response = self.client.delete(f"/media_library/books/{book_id}")
        self.assertEqual(response.status_code, 200)
        self.logout()

    def test_emp1_borrow_a_book(self):
        """
        Test borrow a book
        """
        self.logout()
        response = self.login("emp1")
        response = self.client.get("/media_library/users?login=client1")
        results = json.loads(response.data)
        self.assertEqual(results["total"], None)
        self.assertEqual(len(results["result"]), 1)
        user = results["result"][0]

        # create a book
        response = self.client.post(
            "/media_library/books",
            json={"title": "martine a la plage 3", "pages": 23},
        )
        self.assertEqual(response.status_code, 200)

        response = self.client.get("/media_library/books?title.$reg=martine.*3")
        results = json.loads(response.data)
        self.assertEqual(results["total"], None)
        self.assertEqual(len(results["result"]), 1)
        book = results["result"][0]
        # response = self.client.post(
        #     f"/media_library/books/_actions/borrow/{book['_id']}",
        #     json={"user_id": user['_id'], "return_date": "test"},
        # )
        # self.assertEqual(response.status_code, 400)
        return_date = datetime.now() + timedelta(days=3)
        response = self.client.post(
            f"/media_library/books/_actions/borrow/{book['_id']}",
            json={"user_id": user["_id"], "return_date": return_date.isoformat()},
        )
        self.assertEqual(response.status_code, 200)
        response = self.client.get(f"/media_library/users/{user['_id']}")
        u = json.loads(response.data)
        self.assertEqual(u["rent"]["books"][0], book["_id"])

        # verify book has borrowed = True
        response = self.client.get(f"/media_library/books/{book['_id']}")
        b = json.loads(response.data)
        self.assertEqual(b["borrowed"], True)

        # Do the selection on borrowed books
        response = self.client.get("/media_library/books/_selections/borrowed")
        self.assertEqual(response.status_code, 200)
        results = json.loads(response.data)
        self.assertEqual(results["total"], None)
        self.assertEqual(len(results["result"]), 1)
        self.assertEqual(results["result"][0][2], "client1")
```

## Running the application

Start a disposable MongoDB instance on localhost:27017. **Warning:** startup drops the Users and Books collections.

```bash
cd examples/media_library
python backoffice.py
```

Run the supplied tests separately with `python -m unittest tests.py` from the same directory; importing the application also resets its data.

## Example workflow

### 1. Create a reader as administrator

```bash
BASE=http://localhost:5000
curl -c admin.cookies -H "Content-Type: application/json" -d '{"login":"admin","password":"admin"}' "$BASE/login"
curl -b admin.cookies -H "Content-Type: application/json" -d '{"login":"reader1"}' "$BASE/media_library/users"
```

### 2. Sign in as an employee and create a book

```bash
curl -c emp.cookies -H "Content-Type: application/json" -d '{"login":"emp1","password":"emp1"}' "$BASE/login"
curl -b emp.cookies -H "Content-Type: application/json" -d '{"title":"Example book","pages":120}' "$BASE/media_library/books"
curl -b emp.cookies "$BASE/media_library/users?login=reader1"
```

### 3. Record a loan

Copy the returned book and reader IDs.

```bash
BOOK_ID="<book id>"
USER_ID="<reader id>"
curl -b emp.cookies -H "Content-Type: application/json" -d "{\"user_id\":\"$USER_ID\",\"return_date\":\"2027-01-15T12:00:00\"}" "$BASE/media_library/books/_actions/borrow/$BOOK_ID"
```

### 4. Inspect the loan and borrowed-book selection

```bash
curl -b emp.cookies "$BASE/media_library/books/$BOOK_ID"
curl -b emp.cookies "$BASE/media_library/users/$USER_ID"
curl -b emp.cookies "$BASE/media_library/books/_selections/borrowed"
curl -b emp.cookies "$BASE/media_library/_openapi"
```

The book has a computed `borrowed` field and the user exposes the related books. The migration script illustrates dry-run and actual migrations; its placeholder IDs must be replaced before use.

## Generated API endpoints

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/media_library/users` | List items |
| POST | `/media_library/users` | Create an item |
| GET | `/media_library/users/{id}` | Retrieve an item |
| PUT | `/media_library/users/{id}` | Update an item |
| DELETE | `/media_library/users/{id}` | Delete an item |
| GET | `/media_library/books` | List items |
| POST | `/media_library/books` | Create an item |
| GET | `/media_library/books/{id}` | Retrieve an item |
| PUT | `/media_library/books/{id}` | Update an item |
| DELETE | `/media_library/books/{id}` | Delete an item |
| POST | `/login` | Demo login |
| GET | `/logout` | Clear login cookie |
| GET | `/ping` | Inspect authenticated user |
| POST | `/media_library/users/_actions/toggle_role/{id}` | Toggle a role |
| POST | `/media_library/books/_actions/borrow/{id}` | Record a loan |
| GET | `/media_library/books/_selections/borrowed` | List borrowed books |
| GET | `/media_library/_openapi` | API specification |

## Why Backo?

Collection and field permissions, computed fields, references, actions and selections keep business logic close to the model.

## Production considerations

Replace fake authentication and hard-coded secrets; secure cookies, refresh permissions after role changes, and remove destructive startup initialization. The original client1 initialization checks `emp1_user` rather than `client1_user`; the workflow creates its own reader instead. Review loan invariants and migration scripts before use.

The original source is preserved. Python syntax was checked, but the services and workflows have not been integration-tested.

## References

- [Original example](https://github.com/backo-stricto/backo/tree/main/examples/media_library)
- [Backo documentation](https://backo.readthedocs.io/en/latest/index.html)
- [Backo source code](https://github.com/backo-stricto/backo)