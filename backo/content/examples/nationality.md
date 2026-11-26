+++
title = "Nationality Management"
description = "An application stores people locally while retrieving country reference data from an external API. A person has a primary nationality and optional additional nationalities. Countries are read-only."
weight = 1
+++


## Installation

Python 3.12 or later is required. From the root of the supplied Backo checkout:

```bash
pip install -e ".[mongo]"
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

import sys
from flask import Flask

sys.path.insert(1, "../../../stricto")
sys.path.insert(1, "../../../backo")

from collections_set import countries, people
from backo import Backoffice, current_user, log_system

log_system.add_handler(log_system.set_streamhandler())


# set the flask application route
flask = Flask("nationality")


myapp = Backoffice("nationality")
myapp.register_collection(countries)
myapp.register_collection(people)

# To check if you have some syntax issue
myapp.check_syntax()

myapp.build_routes(flask, "")


# ------------------------------------
# Initialisation
# ------------------------------------
current_user.standalone = True

people.drop()

current_user.standalone = False


if __name__ == "__main__":
    flask.run(host="0.0.0.0", port=5000)
```

### `__init__.py`

```python
"""backoffice as a module (used by tests.py)"""

from .backoffice import flask
```

### `collections_set/__init__.py`

```python
"""Module for all collections"""

from .countries import countries
from .people import people
```

### `collections_set/countries.py`

```python
"""Books module"""

# pylint: disable=unused-argument

from backo import (
    Dict,
    String,
    Collection,
    Item,
    log_system,
)
from backo.meta import NoneMetaDataHandler
from .db_country_connector import MyDBRestfullConnector

log = log_system.get_or_create_logger("countries")


# ------------------------------------------------
# ITEM
# ------------------------------------------------


# --------------------
# Description of "what is a country"
# --------------------
item = Item(
    {
        # "name": Dict({"common": String(), "official": String()}),
        "name": String(),
        "cca2": String(),
        "cca3": String(),
        "flags": Dict({"png": String(), "svg": String(), "alt": String()}),
    },
    meta_data_handler=NoneMetaDataHandler(),
)


# ------------------------------------------------
# COLLECTION
# ------------------------------------------------
connector = MyDBRestfullConnector()

countries = Collection(
    "countries",
    item,
    connector,
    can_create=False,
    can_modify=False,
    can_delete=False,
)


# ------------------------------------------------
# ACTIONS
# ------------------------------------------------

# ------------------------------------------------
# SELECTIONS
# ------------------------------------------------
```

### `collections_set/db_country_connector.py`

```python
"""
Module using DBRestFullConnector to connect to restcountries.com API and retrieve country data.

Note: we assume that the signature of some inherited methods differ from the mother class
"""

# pylint: disable=logging-fstring-interpolation,arguments-differ
from backo import log_system, DBError, SFilter
from backo.db import DBRestFullConnector

log = log_system.get_or_create_logger("wget")


class MyDBRestfullConnector(
    DBRestFullConnector
):  # pylint: disable=too-many-instance-attributes
    """An example of a rest API connector"""

    def __init__(self, **kwargs):
        """constructor"""
        DBRestFullConnector.__init__(
            self,
            "nationality",
            host="www.apicountries.com",
            tls=True,
            prefix="countries",
            **kwargs,
        )

    def generate_id(self, o: dict) -> str:  # pylint: disable=unused-argument
        """
        The function to generate an Id.

        :param o: The object given (json format)
        :type o: dict
        :return: an Id
        :rtype: str

        """
        return o["alpha3Code"]

    def _clean_data(self, o: dict) -> dict:
        """clean data from unwanted informations

        :param o: the object
        :type o: dict
        """

        n = {}
        n["_id"] = self.generate_id(o)
        n["name"] = o["name"]
        n["flags"] = o["flags"]
        n["cca2"] = o["alpha2Code"]
        n["cca3"] = o["alpha3Code"]
        return n

    def drop(self):  # pylint: disable=unused-argument
        raise DBError("MyDBRestfullConnector doenst implement drop() method")

    def create(self, o: dict) -> str:  # pylint: disable=unused-argument
        raise DBError("MyDBRestfullConnector doenst implement create() method")

    def save(self, _id: str, o: dict):  # pylint: disable=unused-argument
        raise DBError("MyDBRestfullConnector doenst implement save() method")

    def delete_by_id(self, _id: str):  # pylint: disable=unused-argument
        raise DBError("MyDBRestfullConnector doenst implement delete_by_id() method")

    def get_by_id(self, _id: str) -> dict:
        """See :func:`DBConnector.get_by_id`"""
        return self._internal_get_by_id(
            _id,
            endpoint="alpha",
            query_options={"fields": "name,flags,alpha2Code,alpha3Code"},
        )

    def select(
        self,
        select_filter: SFilter = None,
        projection=list[str],
        page_size: int = 0,
        num_of_element_to_skip: int = 0,
        sort_object: list[str] = None,
    ) -> list:
        """See :func:`DBConnector.select`

        Params ``select_filter`` and ``projection`` are not used

        """
        log.debug(
            "select(%r, %r).sort(%r).skip(%r).limit(%r)",
            select_filter,
            projection,
            sort_object,
            num_of_element_to_skip,
            page_size,
        )

        query_options = {}

        query_options["offset"] = num_of_element_to_skip
        query_options["limit"] = page_size

        return self._internal_select(
            select_filter,
            projection,
            page_size,
            num_of_element_to_skip,
            sort_object,
            query_options=query_options,
        )
```

### `collections_set/people.py`

```python
"""Users module"""

# pylint: disable=unused-argument

from backo import (
    String,
    Collection,
    Item,
    log_system,
    Ref,
    RefsList,
)
from backo.db import DBMongoConnector
from backo.meta import NoneMetaDataHandler

log = log_system.get_or_create_logger("user")


# ------------------------------------------------
# ITEM
# ------------------------------------------------


# ------------------------------------
# Description of the Item
#
# The item is the object in the collection
# ------------------------------------
item = Item(
    {
        "name": String(require=True, default=""),
        "surname": String(require=True, default=""),
        "first_nationality": Ref(coll="countries"),
        "other_nationalities": RefsList(coll="countries"),
    },
    meta_data_handler=NoneMetaDataHandler(),
)

# ------------------------------------------------
# COLLECTION
# ------------------------------------------------

# First define the connector  = where to store datas
connector = DBMongoConnector("mongodb://localhost:27017/country", "Users")


#
# The collection creation
#
# Association of Item, collector, rights
#
people = Collection(
    "people",
    item,
    connector,
)


# ------------------------------------------------
# SELECTIONS
# ------------------------------------------------
```

### `tests.py`

```python
"""
backoffice tests : The main application test
"""

import unittest
import json
import sys

sys.path.insert(1, "../../../backo")
sys.path.insert(1, "../../../stricto")
sys.path.insert(1, "../")


from nationality import flask


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

    def test_get_a_country(self):
        """
        try to get a country
        """
        response = self.client.get("/nationality/countries/can")
        self.assertEqual(response.status_code, 200)
        results = json.loads(response.data)
        self.assertEqual(results["name"], "Canada")
        self.assertEqual(results["_id"], "CAN")

    def test_get_a_false_country(self):
        """
        try to get a wrong country
        """
        response = self.client.get("/nationality/countries/trumpland")
        self.assertEqual(response.status_code, 404)

    def test_select_country(self):
        """
        try to select country
        """
        response = self.client.get("/nationality/countries?name=France")
        self.assertEqual(response.status_code, 200)
        results = json.loads(response.data)
        self.assertEqual(results["total"], None)
        self.assertEqual(len(results["result"]), 1)

    def test_create_person(self):
        """
        try to create
        """
        response = self.client.post(
            "/nationality/people",
            json={
                "name": "bourne",
                "surname": "jason",
                "first_nationality": "USA",
                "other_nationalities": ["FRA"],
            },
        )
        self.assertEqual(response.status_code, 200)
        d = json.loads(response.data)
        jason_id = d["_id"]

        # delete the user

        response = self.client.delete(f"/nationality/people/{jason_id}")
        self.assertEqual(response.status_code, 200)
```

## Running the application

Start a disposable MongoDB instance on localhost:27017 and ensure outbound HTTPS access to www.apicountries.com. Startup drops the local people collection.

```bash
cd examples/nationality
python backoffice.py
```

The supplied tests can be run separately with `python -m unittest tests.py`.

## Example workflow

### 1. Retrieve a country

```bash
BASE=http://localhost:5000/nationality
curl "$BASE/countries/can"
curl "$BASE/countries?name=France"
```

The supplied tests expect Canada to have `_id` equal to `CAN`. The connector maps external alpha codes to local fields.

### 2. Register a person

```bash
curl -H "Content-Type: application/json" -d '{"name":"bourne","surname":"jason","first_nationality":"USA","other_nationalities":["FRA"]}' "$BASE/people"
```

### 3. Retrieve and remove the person

```bash
ID="<returned _id>"
curl "$BASE/people/$ID"
curl -X DELETE "$BASE/people/$ID"
curl "$BASE/_openapi"
```

## Generated API endpoints

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/nationality/people` | List items |
| POST | `/nationality/people` | Create an item |
| GET | `/nationality/people/{id}` | Retrieve an item |
| PUT | `/nationality/people/{id}` | Update an item |
| DELETE | `/nationality/people/{id}` | Delete an item |
| GET | `/nationality/countries` | List countries |
| GET | `/nationality/countries/{id}` | Retrieve a country |
| GET | `/nationality/_openapi` | API specification |

## Why Backo?

`Ref` and `RefsList` link MongoDB records to an external read-only collection through a custom connector.

## Production considerations

Remove startup deletion, protect personal data, and add authentication, caching and timeout/error handling. The actual connector targets apicountries.com, despite its module docstring mentioning restcountries.com. External API availability and filtering behavior were not verified.

The original source is preserved. Python syntax was checked, but the services and workflows have not been integration-tested.

## References

- [Original example](https://github.com/backo-stricto/backo/tree/main/examples/nationality)
- [Backo documentation](https://backo.readthedocs.io/en/latest/index.html)
- [Backo source code](https://github.com/backo-stricto/backo)