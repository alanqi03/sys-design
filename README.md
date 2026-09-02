# System Design

A system-design textbook built with Jupyter Book 2 and MyST Markdown.

## Local development

Jupyter Book 2 supports several installation methods. With Python and a virtual
environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Start the live-preview server:

```bash
jupyter book start
```

Then open the local URL printed by the command (normally
`http://localhost:3000`). To create a static HTML build and treat warnings as
errors, run:

```bash
jupyter book build --html --strict
```

The `version: 1` field in `myst.yml` is the current MyST configuration schema;
the installed `jupyter-book` package is constrained to major version 2.

