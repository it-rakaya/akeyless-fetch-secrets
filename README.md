# Load Secrets from Akeyless

A composite GitHub Action that authenticates with [Akeyless](https://www.akeyless.io/) and fetches a secret, either exporting its key-value pairs as environment variables (`GITHUB_ENV`) or writing the raw secret to a file.

## Features

- Authenticates to Akeyless using an Access ID / Access Key pair
- Fetches a secret by path
- Two output modes:
  - **Environment variables** (default) — secret must be a JSON key-value object; each key becomes an env var
  - **File output** — writes the raw secret value to a specified file path (any format)
- Automatically masks secret values in logs (including multiline values)
- Cleans up temporary token/secret variables after use

## Inputs

| Name | Description | Required | Default |
|---|---|---|---|
| `access-id` | Akeyless Access ID | Yes | — |
| `access-key` | Akeyless Access Key | Yes | — |
| `secret-path` | Path of the secret in Akeyless | Yes | — |
| `output-file` | If set, writes the raw secret to this file instead of exporting env vars | No | `""` |

## Usage

### Export secret as environment variables

The secret at `secret-path` must be a JSON object (e.g. `{"DB_USER": "admin", "DB_PASS": "s3cr3t"}`). Each key is exported as an environment variable for subsequent steps.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Load secrets
        uses: ./.github/actions/load-secrets-akeyless
        with:
          access-id: ${{ secrets.AKEYLESS_ACCESS_ID }}
          access-key: ${{ secrets.AKEYLESS_ACCESS_KEY }}
          secret-path: /prod/myapp/db-credentials

      - name: Use secrets
        run: echo "Connecting as $DB_USER"
```

### Write secret to a file

Useful for non key-value secrets (e.g. certificates, private keys, raw config files).

```yaml
- name: Load secret to file
  uses: ./.github/actions/load-secrets-akeyless
  with:
    access-id: ${{ secrets.AKEYLESS_ACCESS_ID }}
    access-key: ${{ secrets.AKEYLESS_ACCESS_KEY }}
    secret-path: /prod/myapp/tls-cert
    output-file: ./certs/tls.pem
```

## How it works

1. **Authenticate** — Calls Akeyless's `/auth` endpoint with the provided Access ID/Key (`api_key` access type) to obtain a short-lived token. The token is masked in logs and stored in `GITHUB_ENV`.
2. **Fetch and export** — Calls `/get-secret-value` for the given `secret-path`.
   - If `output-file` is set, the raw secret value is written to that file (permissions set to `600`) and the step exits.
   - Otherwise, the secret is parsed as a JSON object and each key/value pair is masked and appended to `GITHUB_ENV`, making it available to later steps as `$KEY`.
3. **Cleanup** — Clears the token and secret values from the environment at the end of the job.

## Requirements

- `curl` and `jq` must be available on the runner (present by default on `ubuntu-latest`).
- The Akeyless credentials used must have read access to the target `secret-path`.

## Security notes

- Secret values and the auth token are masked via `::add-mask::`, but masking only prevents accidental display in logs — treat any exported secrets with the same care as native GitHub Actions secrets.
- When using `output-file`, ensure the output path is not committed or uploaded as a build artifact.
- Use repository or organization-level `secrets` (not plain text) for `access-id` and `access-key`.

## License

This project is licensed under the MIT License.

```
MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```