# Contributing to the Wolfgang.Etl Framework Documentation

Thank you for your interest in improving the **Wolfgang.Etl** documentation site! Contributions are welcome.

## How Can You Contribute?

- Reporting errors in the docs (wrong signatures, stale versions, broken links)
- Suggesting new guides or cookbook recipes
- Submitting pull requests that fix or extend a page
- Improving code samples so they compile against the current packages

**Please note:** before writing a large change, open an issue and get agreement on it first. Pull requests are much more likely to be merged when the change has been agreed.

Bugs in the libraries themselves belong in the library's own repository (for example
[ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) or
[ETL-Json](https://github.com/Chris-Wolfgang/ETL-Json)), not here.

---

## Getting Started

1. **Fork the repository** and clone it locally.
2. Install the site generator (Python 3 and pip required):

   ```bash
   pip install -r requirements.txt
   ```

3. Preview the site while you edit:

   ```bash
   mkdocs serve
   ```

   Then open <http://localhost:8000>.

4. Create a branch for your change, make it, and open a pull request against `main`.

---

## Build Before You Push

The pull-request check runs a strict build. Run the same command locally; it must finish with no warnings:

```bash
mkdocs build --strict
```

`--strict` turns every warning into an error, including broken internal links and anchors.

---

## Writing Guidelines

- Pages are Markdown under `docs/`; add every new page to the `nav` in `mkdocs.yml`.
- Use relative links between pages (`../guides/index.md`), never absolute site URLs.
- Code samples should compile against the **current** published packages. Check the type's
  `PublicAPI.Shipped.txt` in its source repository when in doubt about a constructor or generic arity.
- Follow the C# formatting used throughout the site: Allman braces and multi-line argument lists with
  the opening parenthesis on its own line.
- Describe extractors, transformers and loaders in E-T-L flow order.

---

## Pull Requests

- Keep each pull request focused on one change.
- Reference the related issue in the description.
- Changes to workflow files under `.github/workflows/` go in a pull request of their own.
- Respond to review feedback in a timely manner.

---

## Code of Conduct

Please be respectful and considerate in all interactions. See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for our community guidelines.

---

Thank you for contributing!
