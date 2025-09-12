Actuated documentation
=====================

Pre-reqs

You'll need Python and pip installed, then run:

```bash
pip install -r requirements.txt
```

Local testing:

```bash
mkdocs serve
```

Local testing with docker:

```bash
docker run --rm -it -p 8000:8000 -v `pwd`:/docs squidfunk/mkdocs-material:latest
```

Access the site at http://127.0.0.1:8000

## Contributing

See the [contribution guide](./CONTRIBUTING.md)

All commits must be signed-off with the CLI using `git commit --sign-off`
