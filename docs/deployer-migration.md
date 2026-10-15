# Deployer isolation migration runbook

## Observed server state (read-only Ansible audit, 2026-10-16)

Host: `uye-home` (`rwrotson@10.0.0.1`). No Postgres container, network or volumes existed before preparation. The running Compose labels match `infra-vault/vault-unseal`, `infra-edge/nginx`, `infra-portainer/portainer`, `infra-runner/controller` and `infra-runner/deployer`. Existing data resources are `infra-vault_vault-data`, `infra-vault_vault-logs`, `infra-edge_certbot_certs`, `infra-portainer_portainer-data`, `vault-net`, `portainer-net`, and the runner networks. The Edge container uses host networking. The runner checkout has local `docker-compose.override.yml` staged and `.env.bak` untracked; do not force-reset it. Its `.env` still contains legacy plugin overlays and Vault token until cutover.

The current Vault unseal, runner controller and old deployer image digests verify under their expected CI identities. The current Edge and Portainer digests and `:latest` tags return `cosign: no signatures found`. **The cutover must fail before stopping the old deployer while either unsigned image is running.** The executor `--check` preflight verifies signed rollback digests for all existing services.

## Publish in stages (the user makes all commits and pushes)

1. Commit and merge the Edge and Portainer **CI workflow changes only**. Their old plugin descriptors must remain active on the server for this transitional release. Wait for validation, SHA image build, signature and `:latest` promotion. Verify both GHCR packages are public.
2. Let the existing deployer update Edge and Portainer to those signed digests. Through read-only Ansible, verify the running image `RepoDigests` and exact cosign identities. Confirm Edge and Portainer health. If either remains unsigned, stop here; do not use `deployer-cutover.yml`.
3. Rerun `playbooks/deployer-prepare.yml` to install the current read-only audit command, then run `playbooks/deployer-freeze.yml` through Ansible. It checks the exact signatures of the **running** Edge and Portainer images, then stops the old deployer. Keep its proxy temporarily but do not publish the new runner image before this freeze. Otherwise the old deployer can self-update the new poller under its old secret-bearing Compose spec.
4. Commit and merge the remaining Edge/Portainer overlay and descriptor retirement, Vault changes, infra-runner changes and infra-ansible changes. The user owns each commit/push. Wait for all CI and signed promotions. The old deployer remains stopped during this publication. Publish the new infra-postgres SHA pair before its separate deployment.
5. Rerun `playbooks/deployer-prepare.yml` from the Ansible control machine. It installs root-owned executor code, allowlist, Compose specs, snapshots, secret files and systemd units with timer stopped. It never recreates running containers. The first run on 2026-10-16 succeeded; the second had `changed=0`. After code changes, rerun before cutover to update the installed executor.
6. Run `deployer-cutover.yml` through Ansible. Its preflight requires every new candidate and every still-running rollback image to be signed, checks current projects/networks/volumes, and fast-forwards the dirty runner checkout without forcing local changes. Then it transfers Vault, Edge, Portainer and runner sequentially, retires old overlays/proxy, and enables the timer. A failure leaves the timer disabled and does not restore the old proxy automatically. The intentionally stopped old poller is replaced under the new trusted manifest; it is never used as a rollback image.
7. Confirm timer, executor status and every managed container image ID/health. Inspect the poller: no Docker socket, proxy network, `DOCKER_HOST`, plugin checkout, server `.env`, Vault token or App key. Probe Docker API from poller; it must fail. Introduce an unsigned candidate in a safe test to prove it does not mutate a service.
8. Only then run the new `playbooks/postgres.yml`. It verifies both signed images before creating resources, provisions tenants without Docker access and requires a `complete` initial backup. Verify tenant isolation, a restore on a test database and metrics/alerts.

## Failure handling

If cutover fails, stop `uye-deploy.timer`, leave current containers and volumes intact, repair the executor and rerun preflight. The old deployer proxy is not restarted automatically. A failed post-update health check restores the previous **signed** digest and reports an error. Operator rollback to another digest requires an approved signed image, an Ansible procedure and a backup first.

## Remaining host risk

The controller and Portainer proxies still permit Docker `CONTAINERS+POST`, and `ghrunner` still belongs to the Docker group. They can reach host-level Docker rights despite API path filtering. This project isolates the poller only; isolate these remaining paths separately.
