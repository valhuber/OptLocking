This is a POC for Optimistic Locking.

It is mainly a generated app, then altered to explore a bug described in [the opt_locking_readme](./api/system/opt_locking/readme.md).

Please click the link to see that file.

> Note: you cannot run this under Codespaces; depends on a patched version of SAFRS.

> Suggestion: open this in GitHub using "Project View" (Shift + ".")

## Alert - upgrading to safrs==3.1.0rc2 (WIP)

With GA safrs, config was to use a global (GA) venv, which seemed to work.

rc2 safrs requires the following:

1. Create a local venv:

python3 -m venv venv; . venv/bin/activate; python3 -m pip install -r requirements.txt

2. Get the rc2 safrs

python -m pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple  safrs==3.1.0rc2