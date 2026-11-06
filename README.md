# NADA Soda

Example Naisjob for periodically running Soda v4 data quality contracts against BigQuery and posting failures to Slack.
Also serves as a smoketest for the full [nada-soda](https://github.com/navikt/nada-soda) + [nada-soda-service](https://github.com/navikt/nada-soda-service) setup.

## Images

Use `europe-north1-docker.pkg.dev/nais-management-233d/nada/nada-soda-contracts:<tag>` in the Naisjob. See [.nais/dev/soda/](https://github.com/navikt/dp-nada-soda/tree/main/.nais/dev/soda) for an example.

The image is also mirrored to GHCR as `ghcr.io/navikt/nada-soda/soda-contracts`. Dependabot tracks this image through `Dockerfile.dummy`, and the image-update workflow copies its tag into the dev and prod Naisjob manifests.

See [navikt/nada-soda](https://github.com/navikt/nada-soda) for available tags.

## Prerequisites

The Docker image requires:

- A **data source config** for connecting to BigQuery
- One or more **check files** describing the tests to run
- A Slack channel for data quality alerts

All of these must be provided via environment variables as described in [Environment variables](#environment-variables).

### Data source config

The config defines one data source per BigQuery project. The dataset is specified in the contract file:

```yaml
type: bigquery
name: <datasource_name>
connection:
  use_context_auth: true
  project_id: <gcp-project-id>
```

See [soda-config.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda/soda-config.yaml) for a complete example.

### Contract files

Each contract file covers exactly **one table**. The table is identified by a fully-qualified `dataset` path:

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

See [soda-checks.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda/soda-checks.yaml) for a complete example. Full reference at [docs.soda.io/reference/contract-language-reference](https://docs.soda.io/reference/contract-language-reference).

## Environment variables

### Required

- `SODA_CONFIG`: Path to the data source config file
- `SODA_CHECKS_FOLDER`: Path to the folder containing check files
- `SLACK_CHANNEL`: Slack channel for data quality alerts (e.g. `#my-team-alerts`)

### Optional

- `NOTIFY_OK_SCAN_STATUS`: Set to `"true"` to also send Slack notifications when all checks pass

## Deploy to Nais

Start from the example naisjobs and adapt for your team:

- [.nais/dev/soda/naisjob.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/dev/soda/naisjob.yaml)
- [.nais/prod/soda/naisjob.yaml](https://github.com/navikt/dp-nada-soda/blob/main/.nais/prod/soda/naisjob.yaml)

Update the team name, job name and Slack channel for your setup.

### Mounting config and check files

The config and check files must be mounted into the container. In this example they are deployed as Kubernetes ConfigMaps alongside the Naisjob. The ConfigMaps are then mounted via `filesFrom` in the naisjob spec, making the files available under `/var/run/configmaps/<configmap-name>/` in the container.

The `SODA_CONFIG` and `SODA_CHECKS_FOLDER` environment variables are then set to point to these paths. See the full example in [.nais/dev/soda/](https://github.com/navikt/dp-nada-soda/tree/main/.nais/dev/soda).

The Kubernetes resource names retain the `contracts` suffix so the same deployed job and ConfigMaps are updated rather than replaced.

### IAM permissions

The naisjob service account needs the following project-level IAM roles to run the checks:

- `roles/bigquery.dataViewer` to read table data
- `roles/bigquery.jobUser` to run BigQuery jobs

Set the correct `project_id` under `gcp.permissions` in the naisjob spec.

### Job failure alerts

The deploy workflow includes the `PrometheusRule` in `.nais/dev/alert.yaml` or `.nais/prod/alert.yaml` alongside the job and its ConfigMaps. The rule selects failed Jobs owned by the `soda-contracts` CronJob in the `nada` namespace, using `kube_job_failed` and `kube_job_owner`.

The alert fires after the failure condition has been present for two minutes. Because failed Jobs are retained, a failed Job can keep the alert active after a later run succeeds. The rule does not detect missing scheduled runs.
