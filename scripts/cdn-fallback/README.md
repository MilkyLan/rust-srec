# Douyin CDN fallback image

The [GitHub Actions workflow](../../.github/workflows/cdn-fallback.yml) follows stable `rust-srec-vX.Y.Z` releases from `hua0512/rust-srec`. Each image uses the source pinned by the corresponding release tag, carries the `X.Y.Z-cdn-fallback` tag, and targets `linux/amd64`. The upstream `main` branch and `dev` image may include changes made after that tag, even when the program reports the same version number.

The scheduled workflow checks once a day and skips versions already present in the registry. It also runs on manual dispatch and on pushes to the fork's `main` branch that touch the workflow or files in this directory.

## Recording behavior

For a Douyin room, the recorder prefers eligible streams outside the CDN blacklist. When every eligible stream is blacklisted, it tries those streams using the ordinary ranking and emits a warning. Both the built-in extractor and Streamlink use this selection rule. Other platforms continue to enforce the blacklist strictly.

## Install with Docker Compose

Back up the database and Compose file before replacing an existing recorder. If the current recorder uses a development image, check the database compatibility described below first. Set the backend service's `image` to the published fallback tag for the desired release. The Compose setting below uses upstream `v0.8.0`.

```yaml
services:
  rust-srec:
    image: ghcr.io/milkylan/rust-srec:0.8.0-cdn-fallback
```

Keep the service's existing volumes and environment settings. From the Compose project directory, pull the backend image, recreate the backend service, and check its status.

```sh
docker compose pull rust-srec
docker compose up -d --no-deps rust-srec
docker compose ps rust-srec
```

Check the liveness endpoint from inside the backend container. The response should include `"status":"alive"`.

```sh
docker compose exec -T rust-srec sh -c \
  'curl -fsS "http://127.0.0.1:${API_PORT:-12555}/api/health/live"'
```

## Database compatibility when switching from a development image

At startup, SQLx reads the database's `_sqlx_migrations` table and compares the recorded migration versions with the migrations embedded in the recorder. A migration recorded in the database and absent from the image stops startup with an error such as `migration 20261108120000 was previously applied but is missing in the resolved migrations`. An image that includes the same applied migrations can pass this check.

Migration [`20261108120000`](https://github.com/hua0512/rust-srec/blob/4691f860/rust-srec/migrations/20261108120000_disable_cancelled_streamers.sql) landed after the `v0.8.0` release. It converts `CANCELLED` streamers to `DISABLED` and clears their error and retry state. The migration updates rows without changing the schema.

For this specific migration, the `0.8.0-cdn-fallback` image was verified against a copy of an affected database after removing only the corresponding row from `_sqlx_migrations`. That edit leaves the streamer data changes in place. To use this workaround, first stop the recorder, make a consistent SQLite backup, and repeat the startup check on a copy before editing the deployment database. Keep the backup so the original migration history can be restored if needed. A future image that includes the migration will consider it pending and run it again. Other migration IDs require their own compatibility check.

## Build and publish

The workflow checks the release tag against the Cargo version and runs the focused regression test before publishing. If a new release changes the patched code and the patch no longer applies, the workflow stops. Update `douyin.patch` against that upstream release and push the change to retry.
