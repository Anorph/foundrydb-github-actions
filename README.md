# FoundryDB GitHub Actions

Reusable GitHub Actions (composite) for integrating [FoundryDB](https://foundrydb.com) into your CI/CD pipelines. Provision ephemeral databases for integration tests, trigger nightly backups, and retrieve connection strings without any custom scripting.

## Supported databases

- PostgreSQL
- MySQL
- MongoDB
- Valkey
- Kafka

## Quick start

### 1. Add secrets to your repository

In your repository go to **Settings > Secrets and variables > Actions** and add:

| Secret | Value |
|--------|-------|
| `FOUNDRYDB_USERNAME` | Your FoundryDB username |
| `FOUNDRYDB_PASSWORD` | Your FoundryDB password |

For production backup workflows also add:

| Secret | Value |
|--------|-------|
| `PROD_SERVICE_ID` | The ID of your production database service |

### 2. Use the actions

All actions live under `anorph/foundrydb-github-actions/actions/<name>@v1`. They require no installation beyond what GitHub Actions provides natively (curl and jq are available on all `ubuntu-latest` runners).

---

## Actions reference

### `setup`

Downloads the `fdb` CLI and sets `FOUNDRYDB_API_URL`, `FOUNDRYDB_USERNAME`, and `FOUNDRYDB_PASSWORD` environment variables for subsequent steps.

```yaml
- uses: anorph/foundrydb-github-actions/actions/setup@v1
  with:
    username: ${{ secrets.FOUNDRYDB_USERNAME }}
    password: ${{ secrets.FOUNDRYDB_PASSWORD }}
    # api-url: https://api.foundrydb.com  (optional, this is the default)
    # version: latest                     (optional)
```

### `create-service`

Creates a managed database service and waits up to 15 minutes for it to reach `running` status.

```yaml
- uses: anorph/foundrydb-github-actions/actions/create-service@v1
  id: db
  with:
    username: ${{ secrets.FOUNDRYDB_USERNAME }}
    password: ${{ secrets.FOUNDRYDB_PASSWORD }}
    name: my-test-db
    database-type: postgresql   # postgresql | mysql | mongodb | valkey | kafka
    version: "17"
    # plan: tier-2              (optional, default: tier-2)
    # storage-gb: "50"          (optional, default: 50)
    # zone: se-sto1             (optional, default: se-sto1)
```

Outputs: `service-id`, `service-name`, `host`, `port`, `status`.

Also writes the service ID to `.foundrydb-service-id` so the `delete-service` action can find it automatically.

### `delete-service`

Deletes a service by ID or reads the ID from `.foundrydb-service-id`. Safe to run in `if: always()` cleanup steps.

```yaml
- uses: anorph/foundrydb-github-actions/actions/delete-service@v1
  if: always()
  with:
    username: ${{ secrets.FOUNDRYDB_USERNAME }}
    password: ${{ secrets.FOUNDRYDB_PASSWORD }}
    service-id: ${{ steps.db.outputs.service-id }}  # or omit to use .foundrydb-service-id
```

### `get-connection-string`

Retrieves credentials for a database user and builds a connection string. The password and the full connection string are masked via `::add-mask::` and will not appear in logs.

```yaml
- uses: anorph/foundrydb-github-actions/actions/get-connection-string@v1
  id: conn
  with:
    username: ${{ secrets.FOUNDRYDB_USERNAME }}
    password: ${{ secrets.FOUNDRYDB_PASSWORD }}
    service-id: ${{ steps.db.outputs.service-id }}
    # db-username: app_user   (optional, default: app_user)
    # format: url             (optional, url or env, default: url)
```

Output: `connection-string` (masked).

### `trigger-backup`

Triggers an on-demand backup for a service.

```yaml
- uses: anorph/foundrydb-github-actions/actions/trigger-backup@v1
  id: backup
  with:
    username: ${{ secrets.FOUNDRYDB_USERNAME }}
    password: ${{ secrets.FOUNDRYDB_PASSWORD }}
    service-id: ${{ secrets.PROD_SERVICE_ID }}
```

Output: `backup-id`.

---

## Example workflows

### Ephemeral database per pull request

```yaml
name: PR Integration Tests
on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: anorph/foundrydb-github-actions/actions/setup@v1
        with:
          username: ${{ secrets.FOUNDRYDB_USERNAME }}
          password: ${{ secrets.FOUNDRYDB_PASSWORD }}

      - uses: anorph/foundrydb-github-actions/actions/create-service@v1
        id: db
        with:
          username: ${{ secrets.FOUNDRYDB_USERNAME }}
          password: ${{ secrets.FOUNDRYDB_PASSWORD }}
          name: pr-${{ github.event.pull_request.number }}-db
          database-type: postgresql
          version: "17"

      - uses: anorph/foundrydb-github-actions/actions/get-connection-string@v1
        id: conn
        with:
          username: ${{ secrets.FOUNDRYDB_USERNAME }}
          password: ${{ secrets.FOUNDRYDB_PASSWORD }}
          service-id: ${{ steps.db.outputs.service-id }}

      - name: Run tests
        env:
          DATABASE_URL: ${{ steps.conn.outputs.connection-string }}
        run: npm test

      - uses: anorph/foundrydb-github-actions/actions/delete-service@v1
        if: always()
        with:
          username: ${{ secrets.FOUNDRYDB_USERNAME }}
          password: ${{ secrets.FOUNDRYDB_PASSWORD }}
          service-id: ${{ steps.db.outputs.service-id }}
```

Full example: [`examples/pr-ephemeral-db.yml`](examples/pr-ephemeral-db.yml)

### Nightly backup

```yaml
name: Nightly Backup
on:
  schedule:
    - cron: '0 2 * * *'

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - uses: anorph/foundrydb-github-actions/actions/trigger-backup@v1
        with:
          username: ${{ secrets.FOUNDRYDB_USERNAME }}
          password: ${{ secrets.FOUNDRYDB_PASSWORD }}
          service-id: ${{ secrets.PROD_SERVICE_ID }}
```

Full example: [`examples/nightly-backup.yml`](examples/nightly-backup.yml)

---

## Notes

- All actions target `ubuntu-latest` runners and require only `curl` and `jq`, both of which are pre-installed.
- The `create-service` action polls every 15 seconds with a 15-minute timeout. For larger clusters (MongoDB replica sets, Kafka) provisioning may take longer; adjust the timeout in the action if needed.
- Connection strings are masked using `::add-mask::` before they appear in any output step.
- The `.foundrydb-service-id` file written by `create-service` allows `delete-service` to clean up even if the step that created the service fails before setting its output variable.

## License

Apache 2.0. Copyright 2026 Anorph. See [LICENSE](LICENSE).
