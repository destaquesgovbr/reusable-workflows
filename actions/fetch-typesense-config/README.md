# Fetch Typesense Config Action

Fetches Typesense configuration from GCP Secret Manager and outputs connection parameters for use in workflows.

## Prerequisites

1. A GCP Secret Manager secret containing a JSON object with Typesense configuration
2. Workload Identity Federation configured for GitHub Actions
3. Service account with `roles/secretmanager.secretAccessor` on the secret

## Secret Format

The secret must be a JSON object with the following structure:

```json
{
  "host": "34.39.186.38",
  "port": 8108,
  "protocol": "http",
  "searchOnlyApiKey": "your-search-only-api-key"
}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `workload_identity_provider` | GCP Workload Identity Provider | Yes | - |
| `service_account` | GCP Service Account email | Yes | - |
| `project_id` | GCP Project ID | No | `inspire-7-finep` |
| `secret_name` | Secret name in Secret Manager | No | `typesense-config` |

## Outputs

| Output | Description |
|--------|-------------|
| `host` | Typesense server host |
| `port` | Typesense server port |
| `protocol` | Connection protocol (http/https) |
| `api_key` | Search-only API key (masked in logs) |

## Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Required for Workload Identity

    steps:
      - uses: actions/checkout@v4

      - name: Fetch Typesense Config
        id: typesense
        uses: destaquesgovbr/reusable-workflows/actions/fetch-typesense-config@v1
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Use Typesense Config
        run: |
          echo "Host: ${{ steps.typesense.outputs.host }}"
          echo "Port: ${{ steps.typesense.outputs.port }}"
        env:
          TYPESENSE_HOST: ${{ steps.typesense.outputs.host }}
          TYPESENSE_PORT: ${{ steps.typesense.outputs.port }}
          TYPESENSE_API_KEY: ${{ steps.typesense.outputs.api_key }}
```

## Example: Next.js Build

```yaml
- name: Fetch Typesense Config
  id: typesense
  uses: destaquesgovbr/reusable-workflows/actions/fetch-typesense-config@v1
  with:
    workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
    service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

- name: Build
  run: npm run build
  env:
    NEXT_PUBLIC_TYPESENSE_HOST: ${{ steps.typesense.outputs.host }}
    NEXT_PUBLIC_TYPESENSE_PORT: ${{ steps.typesense.outputs.port }}
    NEXT_PUBLIC_TYPESENSE_SEARCH_ONLY_API_KEY: ${{ steps.typesense.outputs.api_key }}
```

## Security

- The API key is automatically masked in GitHub Actions logs using `::add-mask::`
- Uses Workload Identity Federation (no long-lived credentials)
- Requires `id-token: write` permission for OIDC authentication
