# Chameleon Operator Docs

This repo uses `mkdocs` to build a static website of docs.

The intended content is resources for Chameleon Openstack Site operators, and assisting with user tickets.


## Building these Docs

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs build
```

## Making a Change

Just edit markdown directly, and make a PR.
To get a live preview while editing, run `mkdocs serve`
