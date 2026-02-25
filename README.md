# Slicer documentation

## README for docs site maintenance

This file is only for working on the documentation site.

## Development setup

Install documentation dependencies:

```bash
pip install -r requirements.txt
```

Run the docs site locally:

```bash
mkdocs serve
```

Or run in Docker:

```bash
docker run --rm -it -p 8000:8000 -v `pwd`:/docs squidfunk/mkdocs-material:latest
```

Docs are available at `http://127.0.0.1:8000`.

## Contributing

See the [contribution guide](./CONTRIBUTING.md).

All commits must be signed-off with `git commit --sign-off`.
