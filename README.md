# Wolfgang.Etl Framework Documentation

Comprehensive guides, architecture reference, and cookbook for the [Wolfgang.Etl](https://github.com/Chris-Wolfgang) framework.

**Live site:** [https://Chris-Wolfgang.github.io/ETL-Documentation/](https://Chris-Wolfgang.github.io/ETL-Documentation/)

## Libraries Covered

| Library | Description | Repo |
|---------|-------------|------|
| [Wolfgang.Etl.Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) | Base classes and interfaces | [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) |
| [Wolfgang.Etl.FixedWidth](https://github.com/Chris-Wolfgang/ETL-FixedWidth) | Fixed-width file extractor/loader | [ETL-FixedWidth](https://github.com/Chris-Wolfgang/ETL-FixedWidth) |
| [Wolfgang.Etl.DbClient](https://github.com/Chris-Wolfgang/ETL-DbClient) | Database extractor/loader | [ETL-DbClient](https://github.com/Chris-Wolfgang/ETL-DbClient) |
| [Wolfgang.Etl.Xml](https://github.com/Chris-Wolfgang/ETL-Xml) | XML extractor/loader | [ETL-Xml](https://github.com/Chris-Wolfgang/ETL-Xml) |
| [Wolfgang.Etl.Json](https://github.com/Chris-Wolfgang/ETL-Json) | JSON extractor/loader | [ETL-Json](https://github.com/Chris-Wolfgang/ETL-Json) |
| [Wolfgang.Etl.TestKit](https://github.com/Chris-Wolfgang/ETL-TestKit) | Test doubles and contract tests | [ETL-TestKit](https://github.com/Chris-Wolfgang/ETL-TestKit) |

## Local Development

### Prerequisites

- Python 3.x
- pip

### Setup

```bash
pip install -r requirements.txt
```

### Serve locally

```bash
mkdocs serve
```

Open [http://localhost:8000](http://localhost:8000) in your browser.

### Build

```bash
mkdocs build
```

The site is generated in the `site/` directory.

## Deployment

Documentation is automatically deployed to GitHub Pages when changes are pushed to `main`.
