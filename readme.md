This is a POC for Optimistic Locking.

It is mainly a generated app, to explore various scenarios described in [the opt_locking_readme](./api/system/opt_locking/readme.md).

Please click the link to see that file.

> Note: you cannot run this under Codespaces; depends on a patched version of SAFRS.

> Suggestion: open this in GitHub using "Project View" (Shift + ".")


# Installation

&nbsp;

## Existing ALS

Using existing ALS/safrs, this should run OptLocking tests (see the readme), and `behave run` tests.  To install, setup the virtual env:

1. [shared venv](https://apilogicserver.github.io/Docs/Project-Env/#shared-venv), or

2. local venv - python3 -m venv venv; . venv/bin/activate; python3 -m pip install ApiLogicServer

&nbsp;

## safrs==3.1.0rc2 (WIP)

rc2 safrs requires the following (caution - still very brittle):

1. Create a local venv:

`python3 -m venv venv; . venv/bin/activate`

2. Install ApiLogicServer

`pip install ApiLogicServer`

3. Update the local venv

`python3 -m pip install -r requirements.txt`

4. Get the rc2 safrs

`python -m pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple  safrs==3.1.0rc2`

5. Install yaml (a surprise)

`sudo pip install pyyaml`

But, fails in safrs_init: ModuleNotFoundError: No module named 'flask_swagger_ui'

6. Install flask_swagger_ui

`sudo pip install flask_swagger_ui`

&nbsp;

### Failing in DB Bind

But, fails when running with security:

`sqlalchemy.exc.UnboundExecutionError: Bind key 'authentication' is not in 'SQLALCHEMY_BINDS' config.`

Thomas concurs that binds need recoding.

previously:
```python
session = db.create_scoped_session(options={"bind": connection, "binds": {}}
```

currently:
```python
from sqlalchemy.orm import scoped_session
from sqlalchemy.orm import sessionmaker
session_factory = sessionmaker(bind=connection)
session = scoped_session(session_factory)
```

See [this gist](https://github.com/thomaxxl/safrs-example/blob/414aae69719db4fa544a086ae694f82047ae772e/tests/conftest.py#L69).

To be continued.
