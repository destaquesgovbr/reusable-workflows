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

### [airflow-deploy-dags](./.github/workflows/airflow-deploy-dags.yml) (`@v3`)

Valida e publica DAGs (e, opcionalmente, um pacote de plugins) do Airflow em um ou mais buckets GCS. Por padrão publica só no bucket do Airflow próprio (VM `destaquesgovbr-airflow`), `gs://destaquesgovbr-airflow`, que o `dag-sync` da VM espelha em `/opt/airflow/{dags,plugins}` a cada 60 s. Substitui o `composer-deploy-dags` (v1), descontinuado desde que o Cloud Composer foi desligado (`infra/_plan/migrar-composer-para-airflow-proprio.md`, Fase 7): não chama `gcloud composer environments describe` e publica em cada bucket de `buckets`.

```yaml
jobs:
  deploy:
    uses: destaquesgovbr/reusable-workflows/.github/workflows/airflow-deploy-dags.yml@v3
    with:
      dags_local_path: dags
      dags_bucket_subdir: scraper
```

O caller precisa de `permissions: id-token: write` (Workload Identity) e o repo precisa estar no binding do pool `github-pool` (`infra/terraform/workload-identity.tf`).

Para cada bucket `b` de `buckets`:

| Origem | Destino |
|--------|---------|
| `plugins_local_path/` (se informado) | `gs://b/plugins/<basename(plugins_local_path)>/` |
| `dags_local_path/` | `gs://b/dags/<dags_bucket_subdir>/` |

#### Inputs

| Input | Obrigatório | Default | Descrição |
|-------|-------------|---------|-----------|
| `dags_local_path` | sim | — | Diretório local das DAGs |
| `dags_bucket_subdir` | sim | — | Subpasta em `dags/`: um nome só, sem `/`; `platform` é reservado (DAG de heartbeat do repo infra) |
| `buckets` | não | `destaquesgovbr-airflow` | Buckets de destino, só o nome (sem `gs://`), separados por espaço ou quebra de linha |
| `python_version` | não | `3.11` | Python da validação |
| `airflow_version` | não | `3.1.8` | Airflow da validação (o da imagem em `infra/airflow/Dockerfile`) |
| `airflow_extra_packages` | não | `apache-airflow-providers-postgres apache-airflow-providers-google apache-airflow-providers-standard` | Pacotes extras da validação |
| `rsync_exclude` | não | `""` | Regex Python de arquivos de DAG a não publicar (ex.: `requirements\.txt$`) |
| `check_imports` | não | `false` | Executa cada DAG para achar erros de import (só avisa) |
| `plugins_local_path` | não | `""` | Pacote de plugins (ex.: `src/data_publishing`); vazio = sem plugins |

O default de `buckets` é só o bucket do Airflow próprio. Até a `v3.0.0` ele incluía também o bucket do Composer (`southamerica-east1-destaque-227e22da-bucket`), da fase em que os dois conviviam; a partir da `v3.1.0` o bucket do Composer saiu, e a tag `v3` aponta para ela. Os callers que fixam `@v3` não mudam. Quem passa `buckets` explicitamente (ex.: o `data-platform`, que repassa a lista do job de plugins) precisa tirar o bucket do Composer da própria lista.

Depois do deploy, um arquivo novo aparece no Airflow em até ~2 min (dag-sync + refresh do bundle) e uma mudança numa DAG existente em até ~4 min (dag-sync + `min_file_process_interval` de 180 s). Ver `infra/airflow/README.md`.

#### Jobs

1. **check-inputs**: recusa `dags_bucket_subdir` vazio, com `/` ou igual a `platform`, nome de bucket inválido (ou com `gs://`) e `rsync_exclude` que não seja regex válida, contenha `;;` ou termine em `;`.
2. **validate-dags**: instala `apache-airflow==<airflow_version>` e os extras com o arquivo de constraints oficial (`constraints-<versão>/constraints-<python>.txt`); roda `py_compile` em todos os `.py` das DAGs e dos plugins, recursivamente (bloqueia o deploy). Com `check_imports`, executa cada arquivo de DAG (os que citam `airflow` e `dag`, como o safe mode do Airflow) com os plugins no `sys.path`; erro de import só gera aviso, como no v1, porque DAGs que leem Variables ou env vars no parse falham sem credenciais do GCP.
3. **deploy-dags**: autentica via Workload Identity e, para cada bucket, roda `gcloud storage rsync --recursive --delete-unmatched-destination-objects` dos plugins e depois das DAGs. Um bucket com erro não impede os outros, mas o job falha. Lista o resultado (e um resumo no job summary) e espera 60 s pelo parse.

#### Exclusões

- `__pycache__/` nunca é publicado.
- `rsync_exclude` vale só para as DAGs e tem a semântica do `-x` do `gsutil rsync` do v1: a regex é testada com `re.match` (ancorada no início) no caminho relativo a `dags_local_path`, na origem e no destino. O que casa não é copiado nem apagado. Para excluir um arquivo em qualquer subpasta, use `(.*/)?nome$`.
- `rsync_exclude` não pode conter `;;`, o separador usado no `--exclude` do `gcloud storage rsync`, nem terminar em `;` (o gcloud corta o `;` final e a regex passaria a excluir mais arquivos).

#### Migrando do `composer-deploy-dags@v1`

Troque só o `uses:`; os inputs são os mesmos. Com o Composer desligado, o v1 não funciona mais.

```diff
-    uses: destaquesgovbr/reusable-workflows/.github/workflows/composer-deploy-dags.yml@v1
+    uses: destaquesgovbr/reusable-workflows/.github/workflows/airflow-deploy-dags.yml@v3
```

### [composer-deploy-dags](./.github/workflows/composer-deploy-dags.yml) (`@v1`, descontinuado)

> **Descontinuado.** O Cloud Composer foi desligado (Fase 7 da migração para o Airflow próprio) e este workflow falha no `gcloud composer environments describe`. Use o [`airflow-deploy-dags@v3`](#airflow-deploy-dags-v3). O arquivo fica no repo só como histórico das tags `v1`.

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

- `@vN` - Latest version of major version N (recommended)
- `@vN.M.P` - Specific version
- `@main` - Latest development (not recommended for production)

Tags maiores em uso: `v1` (`fetch-typesense-config`; o `composer-deploy-dags` está descontinuado), `v2` (`cloud-run-deploy`) e `v3` (`airflow-deploy-dags`).

Depois de mergear uma mudança compatível, crie a tag da versão e mova a tag maior para ela (os callers fixam a maior):

```bash
git checkout main && git pull
git tag v3.1.0
git tag -f v3 v3.1.0
git push origin v3.1.0
git push -f origin v3
```

## Contributing

1. Create a feature branch
2. Make changes
3. Test locally if possible
4. Create a PR
5. After merge, create a new tag if needed

## License

AGPL-3.0 - see [LICENSE](LICENSE) for details.
