# Wolfgang.Etl Framework Documentation

[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/Chris-Wolfgang/ETL-Documentation/badge)](https://scorecard.dev/viewer/?uri=github.com/Chris-Wolfgang/ETL-Documentation)

Comprehensive guides, architecture reference, and cookbook for the [Wolfgang.Etl](https://github.com/Chris-Wolfgang) framework.

**Live site:** [https://Chris-Wolfgang.github.io/ETL-Documentation/](https://Chris-Wolfgang.github.io/ETL-Documentation/)

## Libraries Covered

| Library | Description | Repo |
|---------|-------------|------|
| [Wolfgang.Etl.Abstractions](https://www.nuget.org/packages/Wolfgang.Etl.Abstractions) | Base classes and interfaces | [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) |
| [Wolfgang.Etl.ErrorPolicies](https://www.nuget.org/packages/Wolfgang.Etl.ErrorPolicies) | Ready-made item-error policies (skip, log, dead-letter) | [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) |
| [Wolfgang.Etl.FixedWidth](https://www.nuget.org/packages/Wolfgang.Etl.FixedWidth) | Fixed-width (text and binary) extractor/loader | [ETL-FixedWidth](https://github.com/Chris-Wolfgang/ETL-FixedWidth) |
| [Wolfgang.Etl.Csv](https://www.nuget.org/packages/Wolfgang.Etl.Csv) | CSV extractor/loader | [ETL-Csv](https://github.com/Chris-Wolfgang/ETL-Csv) |
| [Wolfgang.Etl.Json](https://www.nuget.org/packages/Wolfgang.Etl.Json) | JSON extractor/loader | [ETL-Json](https://github.com/Chris-Wolfgang/ETL-Json) |
| [Wolfgang.Etl.Xml](https://www.nuget.org/packages/Wolfgang.Etl.Xml) | XML extractor/loader | [ETL-Xml](https://github.com/Chris-Wolfgang/ETL-Xml) |
| [Wolfgang.Etl.DbClient](https://www.nuget.org/packages/Wolfgang.Etl.DbClient) | Database extractor/loader (ADO.NET) | [Etl-DbClient](https://github.com/Chris-Wolfgang/Etl-DbClient) |
| [Wolfgang.Etl.SqlBulkCopy](https://www.nuget.org/packages/Wolfgang.Etl.SqlBulkCopy) | SQL Server bulk-copy loader | [ETL-SqlBulkCopy](https://github.com/Chris-Wolfgang/ETL-SqlBulkCopy) |
| [Wolfgang.Etl.Transformers](https://www.nuget.org/packages/Wolfgang.Etl.Transformers) | Generic, composable transformers | [ETL-Transformers](https://github.com/Chris-Wolfgang/ETL-Transformers) |
| [Wolfgang.Etl.TestKit](https://www.nuget.org/packages/Wolfgang.Etl.TestKit) | Test doubles | [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) |
| [Wolfgang.Etl.TestKit.Xunit](https://www.nuget.org/packages/Wolfgang.Etl.TestKit.Xunit) | xUnit contract test base classes | [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) |

## Local Development

### Prerequisites

- Python 3.x
- pip

### Setup

```bash
pip install -r requirements.txt
```

`requirements.txt` is a hash-locked closure generated from `requirements.in`, so `pip` installs exactly the
pinned versions. To change a version, edit `requirements.in` and regenerate:

```bash
pip install pip-tools
pip-compile --generate-hashes --strip-extras --output-file=requirements.txt requirements.in
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
