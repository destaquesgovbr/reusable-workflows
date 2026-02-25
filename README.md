# Reusable Workflows

Reusable GitHub Actions and Workflows for the destaquesgovbr organization.

## Available Actions

### [fetch-typesense-config](./actions/fetch-typesense-config)

Fetches Typesense configuration from GCP Secret Manager. Handles authentication via Workload Identity Federation and outputs host, port, protocol, and API key for use in workflows.

```yaml
- uses: destaquesgovbr/reusable-workflows/actions/fetch-typesense-config@v1
  with:
    workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
    service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

## Available Workflows

### [composer-deploy-dags](./.github/workflows/composer-deploy-dags.yml)

Reusable workflow that validates and deploys Airflow DAGs to Cloud Composer. Handles GCP authentication (Workload Identity), bucket discovery, `gsutil rsync`, and post-deploy verification.

```yaml
jobs:
  deploy:
    uses: destaquesgovbr/reusable-workflows/.github/workflows/composer-deploy-dags.yml@v1
    with:
      dags_local_path: dags
      dags_bucket_subdir: scraper
```

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `dags_local_path` | yes | — | Local path to DAGs directory |
| `dags_bucket_subdir` | yes | — | Subdirectory in Composer bucket |
| `python_version` | no | `3.11` | Python version for validation |
| `airflow_version` | no | `3.0.1` | Airflow version for validation |
| `airflow_extra_packages` | no | `apache-airflow-providers-postgres` | Extra pip packages |
| `rsync_exclude` | no | `""` | Regex exclusion pattern for gsutil rsync |
| `check_imports` | no | `false` | Test Python imports beyond syntax |

#### Jobs

1. **validate-dags** — Installs Airflow, runs `py_compile` on all DAG files, optionally checks imports
2. **deploy-dags** — Authenticates to GCP, discovers Composer bucket, syncs DAGs via `gsutil rsync -r -d`, verifies deployment, waits for Airflow to parse

## Versioning

This repository uses semantic versioning:

- `@v1` - Latest version of major version 1 (recommended)
- `@v1.0.0` - Specific version
- `@main` - Latest development (not recommended for production)

## Contributing

1. Create a feature branch
2. Make changes
3. Test locally if possible
4. Create a PR
5. After merge, create a new tag if needed

## License

AGPL-3.0 - see [LICENSE](LICENSE) for details.
