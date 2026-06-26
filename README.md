# NADA Soda

Example Naisjob for periodically running data quality checks against BigQuery and posting failures to Slack.
Also serves as a smoketest for the full [nada-soda](https://github.com/navikt/nada-soda) + [nada-soda-service](https://github.com/navikt/nada-soda-service) setup.

## Images

Two image variants are available — choose based on which version of Soda you use:

| GAR image (use in naisjob) | Soda version | Example |
|----------------------------|-------------|---------|
| `europe-north1-docker.pkg.dev/nais-management-233d/nada/nada-soda:<tag>` | v3 (SodaCL) | [.nais/dev/soda/](https://github.com/navikt/dp-nada-soda/tree/main/.nais/dev/soda) |
| `europe-north1-docker.pkg.dev/nais-management-233d/nada/nada-soda-contracts:<tag>` | v4 (Contracts) | [.nais/dev/soda-contracts/](https://github.com/navikt/dp-nada-soda/tree/main/.nais/dev/soda-contracts) |

The images are also mirrored to GHCR (`ghcr.io/navikt/nada-soda/soda` and `ghcr.io/navikt/nada-soda/soda-contracts`), which is used by Dependabot for automated tag bumping via `Dockerfile.dummy` and `Dockerfile.dummy-contracts`.

See [navikt/nada-soda](https://github.com/navikt/nada-soda) for available tags and migration information from v3 to v4.

## Prerequisites

The Docker image requires:

- A **data source config** for connecting to BigQuery
- One or more **check files** describing the tests to run
- A Slack channel for data quality alerts

All of these must be provided via environment variables as described in [Environment variables](#environment-variables).

### v3 (SodaCL) — data source config

The v3 config defines one data source per BigQuery dataset:

```yaml
data_source <datasource_name>:
  type: bigquery
  project_id: <gcp-project-id>
  dataset: <bq-dataset>
```

See [soda-config.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda/soda-config.yaml) for a complete example.

> Note: The check file name (without `.yaml`) must match the data source name in the config, e.g. `dataproducts.yaml` must match `data_source dataproducts:`.

### v3 (SodaCL) — check files

One check file can cover multiple tables in the same dataset. Each table is identified by a `checks for <table>:` block:

```yaml
checks for <table_name>:
  - missing_count(column) = 0
  - missing_percent(column) < 50 %

  # Custom metric using expression (like a CTE)
  - my_metric >= 1:
      my_metric expression: count(distinct id)
      name: "Check that we have at least one row"

  # Custom metric using raw SQL
  - my_sql_metric >= 1:
      my_sql_metric query: |
        SELECT COUNT(distinct id) FROM my_table
      name: "Same check using SQL"
```

See [soda-checks.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda/soda-checks.yaml) for a complete example. Full reference at [docs.soda.io/soda-cl](https://docs.soda.io/soda-cl/soda-cl-overview.html).

### v4 (Contracts) — data source config

The v4 config defines one data source per BigQuery project (the dataset is specified in the contract file instead):

```yaml
type: bigquery
name: <datasource_name>
connection:
  use_context_auth: true
  project_id: <gcp-project-id>
```

See [soda-config.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda-contracts/soda-config.yaml) for a complete example.

### v4 (Contracts) — check files

In v4, each check file covers exactly **one table**. The table is identified by a fully-qualified `dataset` path:

```
dataset: <datasource_name>/<gcp-project-id>/<bq-dataset>/<table>
```

Checks can be defined at the dataset level (e.g. row count) or per column:

```yaml
dataset: dataproducts/my-project/my-dataset/my-table

checks:
  - row_count:
      threshold:
        must_be_greater_than_or_equal_to: 1

columns:
  - name: id
    checks:
      - missing:
  - name: type
    checks:
      - missing:
          threshold:
            metric: percent
            must_be_less_than: 50
```

See [soda-checks.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda-contracts/soda-checks.yaml) for a complete example. Full reference at [docs.soda.io/reference/contract-language-reference](https://docs.soda.io/reference/contract-language-reference).

## Environment variables

### Required

- `SODA_CONFIG`: Path to the data source config file
- `SODA_CHECKS_FOLDER`: Path to the folder containing check files
- `SLACK_CHANNEL`: Slack channel for data quality alerts (e.g. `#my-team-alerts`)

### Optional

- `NOTIFY_OK_SCAN_STATUS`: Set to `"true"` to also send Slack notifications when all checks pass

## Deploy to Nais

Start from the example naisjobs and adapt for your team:

- v3: [.nais/dev/soda/naisjob.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda/naisjob.yaml)
- v4: [.nais/dev/soda-contracts/naisjob.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda-contracts/naisjob.yaml)

Update the team name, job name and Slack channel for your setup.

### Mounting config and check files

The config and check files must be mounted into the container. In this example they are deployed as Kubernetes ConfigMaps alongside the Naisjob. The ConfigMaps are then mounted via `filesFrom` in the naisjob spec, making the files available under `/var/run/configmaps/<configmap-name>/` in the container.

The `SODA_CONFIG` and `SODA_CHECKS_FOLDER` environment variables are then set to point to these paths. See the full example in [.nais/dev/soda/](https://github.com/navikt/dp-nada-soda/tree/main/.nais/dev/soda) or [.nais/dev/soda-contracts/](https://github.com/navikt/dp-nada-soda/tree/main/.nais/dev/soda-contracts).

### IAM permissions

The naisjob service account needs the following project-level IAM roles to run the checks:

- `roles/bigquery.dataViewer` — to read table data
- `roles/bigquery.jobUser` — to run BigQuery jobs

For v3 only, `roles/bigquery.readSessionUser` is also required for the BigQuery Storage Read API.

Set the correct `project_id` under `gcp.permissions` in the naisjob spec.

