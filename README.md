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
