# PolyHermes operator runbook

This runbook covers the Docker Compose production stack in this repository. The local instance used for validation is bound to loopback only and must remain a no-wallet, no-copy-relationship test system until board approval.

## Architecture

The `wrbug/polyhermes` application image contains the React/Vite frontend served by Nginx and the Kotlin 1.9 / Spring Boot 3.2 backend. Nginx serves the UI and proxies `/api` and `/ws` to the backend (port 8000 inside the container). Spring Data JPA persists application state in MySQL 8; Flyway applies schema migrations at startup. Compose connects both services on a private bridge network and persists MySQL data in the named `mysql-data` volume. Real-time order/position events use the backend WebSocket endpoint. The browser talks to the same-origin Nginx endpoint.

## Configuration (.env)

Compose reads `.env` from the working directory next to `docker-compose.prod.yml`. Keep it outside version control, restrict it to its owner (`chmod 600 .env`), and back it up securely. Settings present in the production Compose file or its example:

| Variable | Purpose | Local secure example |
| --- | --- | --- |
| `DB_URL` | JDBC URL used by the app to reach the Compose database | `jdbc:mysql://mysql:3306/polyhermes?...` |
| `DB_USERNAME` | MySQL application login | `root` (change for hardened deployments) |
| `DB_PASSWORD` | MySQL root password and app database password | Unique generated secret; required |
| `TZ` | Container timezone | `UTC` or your operations timezone |
| `SPRING_PROFILES_ACTIVE` | Spring profile | `prod` |
| `SERVER_PORT` | Host port for app port 80 | `18081` (Compose binds to loopback) |
| `MYSQL_PORT` | Host port for MySQL port 3306 | `13307`; omit/disable host publication if not needed |
| `JWT_SECRET` | JWT signing secret | `openssl rand -hex 64` |
| `ADMIN_RESET_PASSWORD_KEY` | Authorization key for admin password reset | `openssl rand -hex 32` |
| `CRYPTO_SECRET_KEY` | Application key for encryption of stored secrets, as documented upstream | `openssl rand -hex 32`; verify image passes this setting before relying on encryption |
| `LOG_LEVEL_ROOT` | Root logger verbosity | `WARN` |
| `LOG_LEVEL_APP` | Application logger verbosity | `INFO` |
| `ALLOW_PRERELEASE` | Controls whether prereleases may be considered by update logic | `false` |
| `GITHUB_REPO` | Repository used by update/version functionality | `WrBug/PolyHermes` |

The checked-in Compose file currently defaults to the mutable `wrbug/polyhermes:latest` tag and still permits image-level update behavior; set a reviewed immutable release tag in the Compose file before operational use and confirm online self-update is disabled in that image. Do not infer that `ALLOW_PRERELEASE=false` disables updates. The current Compose file also does not pass `CRYPTO_SECRET_KEY` to the app, despite the example documenting it; encryption configuration must be corrected and verified before storing any wallet/API credentials.

## Safe local start and health

Use a dedicated host directory, checkout the reviewed commit, and retain `.env` only in that directory. Generate secrets locally; never paste or commit them:

```sh
umask 077
printf 'DB_PASSWORD=%s\nJWT_SECRET=%s\nADMIN_RESET_PASSWORD_KEY=%s\nCRYPTO_SECRET_KEY=%s\n' \\
  "$(openssl rand -hex 32)" "$(openssl rand -hex 64)" \\
  "$(openssl rand -hex 32)" "$(openssl rand -hex 32)"
```

Put those values in `.env`, set `SERVER_PORT=18081` and `MYSQL_PORT=13307`, then validate and start:

```sh
docker compose -f docker-compose.prod.yml config --quiet
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs --tail=200 app
curl --fail http://127.0.0.1:18081/api/system/health
```

The backend health endpoint is `/api/system/health`. Review app logs for successful Flyway migration startup and a healthy MySQL dependency. Do not import a private key, add a funded account, or enable any copy relationship in validation. The UI admin bootstrap/login credential should be obtained through the application's documented first-run flow and rotated immediately; never record it in this runbook.

## Accounts, leaders, templates and relationships

Only after explicit operational approval, create/import an account in Account Management; private keys are sensitive and must be protected by a verified encryption key and backups. Add the leader's public wallet address in Leader Management. Create a copy template and set its sizing mode and limits. Review daily loss limit, maximum order count, and price tolerance before associating account + leader + template in Copy Trading Configuration. Leave the relationship disabled until a human has reviewed the account, leader, template, market permissions and risk limits; enablement can place real orders. This local validation intentionally does not create accounts or relationships.

## Risk controls

Configure the template's daily loss limit to cap permitted daily losses, its order limit to restrict order count, and its price tolerance to bound the allowed difference from the leader's price. These are application controls, not guarantees: verify units, reset windows, boundary behavior and failure handling in the UI/code before live use. Use conservative limits and retain human monitoring. No setting makes live trading risk-free.

## Update and rollback

Never use a floating tag for an approved deployment. Change the app image to an explicit reviewed release tag/digest, inspect release notes, back up the database, then pull and recreate the app with Compose. Preserve the prior image reference for rollback. Database migrations may not be reversible; restore a matching pre-update database backup before rolling back across a schema migration. Keep online self-update disabled so the deployed image remains pinned.

## Backup and restore

Stop writes or briefly stop the stack for a consistent logical dump. Store dumps encrypted with restrictive permissions outside the repository; keep the encryption key separately. Example:

```sh
umask 077
docker compose -f docker-compose.prod.yml exec -T mysql sh -c 'exec mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction polyhermes' > polyhermes-$(date +%F).sql
```

Restore into a compatible, empty MySQL database after verifying the backup and image/schema compatibility:

```sh
cat polyhermes-backup.sql | docker compose -f docker-compose.prod.yml exec -T mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD" polyhermes'
```

Test restoration periodically in an isolated environment. Back up `.env` separately and securely; losing `CRYPTO_SECRET_KEY` may make encrypted values unrecoverable.

## Network and security notes

Compose should publish app and MySQL ports on `127.0.0.1` by default. Do not change these bindings to `0.0.0.0`; do not expose either port publicly. The application image and MySQL image are external supply-chain inputs; pin reviewed versions/digests, scan them, and apply updates deliberately. Treat logs, database dumps, backups, and the `.env` as sensitive.