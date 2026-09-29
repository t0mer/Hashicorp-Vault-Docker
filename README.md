# Hashicorp-Vault-Docker

A minimal Dockerfile that packages [HashiCorp Vault](https://www.vaultproject.io/) 1.7.2 on Alpine Linux, with a single-node configuration that stores data on the local file system and enables the web UI.

> **Status:** unmaintained (last changed in June 2021). Vault 1.7.2 and Alpine 3.7 are both long out of support. For new deployments, use the official [`hashicorp/vault`](https://hub.docker.com/r/hashicorp/vault) image instead. A 2021 build of this repository is on Docker Hub as [`techblog/hashicorp-vault:latest`](https://hub.docker.com/r/techblog/hashicorp-vault) (`linux/amd64` only) and is not updated.

## Contents

| File | Description |
|------|-------------|
| [`Dockerfile`](Dockerfile) | Downloads the Vault 1.7.2 `linux_amd64` binary into `/vault` on `alpine:3.7` and exposes port 8200. |
| [`vault-config.json`](vault-config.json) | Vault server configuration, copied to `/vault/config/vault-config.json`. |

## Configuration

`vault-config.json`:

| Setting | Value | Meaning |
|---------|-------|---------|
| `backend.file.path` | `vault/data` | File storage backend. The path is relative to the working directory, which is `/` in the image, so data is written to `/vault/data`. |
| `listener.tcp.address` | `0.0.0.0:8200` | Listen on all interfaces, port 8200. |
| `listener.tcp.tls_disable` | `1` | TLS is **disabled**. |
| `ui` | `true` | Enables the Vault web UI. |

Current Vault versions call the storage section `storage` instead of `backend`.

## Usage

Build the image:

```bash
docker build -t vault-local .
```

The image's entrypoint is `vault` with no default command, so you must pass the server arguments yourself. Vault also needs the `IPC_LOCK` capability to lock memory:

```bash
docker run -d --name vault \
  --cap-add IPC_LOCK \
  -p 8200:8200 \
  -v vault-data:/vault/data \
  vault-local server -config=/vault/config/vault-config.json
```

Then initialize and unseal Vault, either from the web UI at `http://localhost:8200/ui` or with the CLI. Because TLS is disabled, point the CLI at plain HTTP (it defaults to `https://`):

```bash
docker exec -e VAULT_ADDR=http://127.0.0.1:8200 vault vault operator init
docker exec -e VAULT_ADDR=http://127.0.0.1:8200 vault vault operator unseal
```

## Known issues

* The image only runs on `linux/amd64`, because the Dockerfile downloads the amd64 Vault binary.
* The `PATH` line in the Dockerfile (`ENV PATH="PATH=$PATH:$PWD/vault"`) produces a malformed value; `vault` is still found because `/vault` ends up on the path.
* Without the `server` arguments shown above, the container just prints Vault's help and exits.

## Security notes

* TLS is disabled, so tokens and secrets travel in clear text. Only use this configuration for local testing, or put Vault behind a TLS-terminating proxy.
* Vault 1.7.2 has known vulnerabilities that were fixed in later releases, for example [CVE-2021-41802](https://nvd.nist.gov/vuln/detail/CVE-2021-41802) (fixed in 1.7.5).

## License

This repository has no license file. HashiCorp Vault 1.7.2 is licensed under the MPL-2.0; newer Vault releases use the Business Source License 1.1.
