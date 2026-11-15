# GitHub Grimoire Action

[![GitHub repo](https://img.shields.io/badge/GitHub-guibranco%2Fgithub--grimoire--action-purple.svg?style=plastic&logo=github)](https://github.com/guibranco/github-grimoire-action "shields.io")
[![GitHub last commit](https://img.shields.io/github/last-commit/guibranco/github-grimoire-action?color=purple&logo=github&style=plastic&label=Last%20commit)](https://github.com/guibranco/github-grimoire-action "shields.io")
[![GitHub license](https://img.shields.io/github/license/guibranco/github-grimoire-action?color=purple&logo=github&style=plastic&label=License)](https://github.com/guibranco/github-grimoire-action "shields.io")

![CI](https://github.com/guibranco/github-grimoire-action/actions/workflows/ci.yml/badge.svg)

🔮 :octocat: A GitHub Action that loads secrets and configuration from a self-hosted [Grimoire](https://github.com/guibranco/grimoire-api) vault into your workflow, as masked environment variables, step outputs or a dotenv file.

Keep one Grimoire API key in GitHub instead of copying every secret into every repository.

---

## Usage

```yml
- name: Load secrets from Grimoire
  uses: guibranco/github-grimoire-action@v1.0.0
  with:
    url: https://api.grimoire.example.com
    api-key: ${{ secrets.GRIMOIRE_API_KEY }}
    environment: production
    secrets: |
      database-password
      smtp-password=MAIL_PASSWORD

- name: Deploy
  run: ./deploy.sh   # DATABASE_PASSWORD, MAIL_PASSWORD and every configuration entry are in the environment
```

Values are the **effective** ones for the application that owns the API key: its own value, or a global value when the application inherits globals.

Every secret value is masked in the logs (each line of a multi-line value). The API key is masked too.

---

## Inputs

| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `url` | Grimoire API base URL | **Yes** | |
| `api-key` | Application API key. Store it as a GitHub secret | **Yes** | |
| `environment` | Grimoire environment slug, e.g. `production` | **Yes** | |
| `secrets` | Secrets to load, newline or comma separated. `name` or `name=ENV_NAME` | No | `""` |
| `configurations` | `all`, `none`, or a list of keys. `Prefix:*` matches a section. `key=ENV_NAME` to rename | No | `all` |
| `prefix` | Prefix for every generated variable name (not for names given with `=ENV_NAME`) | No | `""` |
| `export-env` | Export the values as environment variables for the next steps | No | `true` |
| `set-outputs` | Also set one step output per value, named like the variable | No | `true` |
| `env-file` | Write the values to this dotenv file (mode `600`). See [Dotenv file](#dotenv-file) | No | `""` |
| `mask-configurations` | Mask configuration values too (secrets are always masked) | No | `false` |
| `fail-on-missing` | Fail when a requested secret or configuration key has no value | No | `true` |
| `timeout` | Timeout of each request, in seconds | No | `30` |
| `retries` | Retries after a network error or a 5xx response | No | `2` |

## Outputs

| Output | Description |
| --- | --- |
| `loaded` | Number of values loaded |
| `variables` | JSON array of the variable names loaded (no values) |
| `missing` | JSON array of the requested names that had no value |
| `<VARIABLE>` | One output per value when `set-outputs` is `true`, e.g. `steps.grimoire.outputs.DATABASE_PASSWORD` |

---

## Variable names

Unless you choose a name with `=ENV_NAME`, names are built like the [Pancake](https://github.com/guibranco/pancake) PHP client and .NET configuration do:

| Grimoire | Variable |
| --- | --- |
| secret `database-password` | `DATABASE_PASSWORD` |
| configuration `Feature:DarkMode` | `FEATURE__DARKMODE` |
| configuration `ConnectionStrings:Main` | `CONNECTIONSTRINGS__MAIN` |

`:` becomes `__`, so a .NET app run in the job reads them back as `Feature:DarkMode` from `IConfiguration`.

The action refuses names that would overwrite `PATH`, `HOME`, `CI`, `NODE_OPTIONS` or `GITHUB_*` / `RUNNER_*` / `ACTIONS_*` variables. When a secret and a configuration entry map to the same name, the secret wins (with a warning).

## Examples

### Only some configuration, with a prefix

```yml
- uses: guibranco/github-grimoire-action@v1.0.0
  with:
    url: https://api.grimoire.example.com
    api-key: ${{ secrets.GRIMOIRE_API_KEY }}
    environment: staging
    configurations: |
      Database:*
      Feature:NewDashboard=NEW_DASHBOARD
    prefix: APP_
```

### Use a value as an output

```yml
- id: grimoire
  uses: guibranco/github-grimoire-action@v1.0.0
  with:
    url: https://api.grimoire.example.com
    api-key: ${{ secrets.GRIMOIRE_API_KEY }}
    environment: production
    secrets: ftp-password
    configurations: none
    export-env: false

- uses: SamKirkland/FTP-Deploy-Action@v4
  with:
    server: ftp.example.com
    username: deploy
    password: ${{ steps.grimoire.outputs.FTP_PASSWORD }}
```

### Dotenv file

For PHP apps and Docker images that read a `.env` file:

```yml
- uses: guibranco/github-grimoire-action@v1.0.0
  with:
    url: https://api.grimoire.example.com
    api-key: ${{ secrets.GRIMOIRE_API_KEY }}
    environment: production
    secrets: database-password
    env-file: deploy/.env
    export-env: false
```

The file holds secret values in plain text (double-quoted, escaped). It is created with mode `600`. Ship it to the server and delete it, and **never upload it as an artifact**.

## Missing values

A secret that does not exist, or has no active version in the environment (disabled, expired, not yet valid), fails the step by default, before anything is exported. Set `fail-on-missing: false` to only warn; the names are listed in the `missing` output.

## Roadmap

- Load every secret without listing names, once Grimoire has a bulk export endpoint ([grimoire-api#74](https://github.com/guibranco/grimoire-api/issues/74)).
- Authenticate with GitHub OIDC instead of a stored API key ([grimoire-api#75](https://github.com/guibranco/grimoire-api/issues/75)).

## Development

```bash
node --test tests/*.test.js               # unit tests, no dependencies
node tests/mock-grimoire.js 8089          # mock Grimoire API used by the CI workflow
```
