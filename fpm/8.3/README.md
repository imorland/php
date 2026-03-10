## PHP 8.3 FPM Images

PHP 8.3 images built on `php:8.3-fpm` (rolling, not pinned to a Debian release).
Apache runs in **mpm_event** mode and proxies to PHP-FPM over a Unix socket for better concurrency than the traditional mod_php / mpm_prefork setup.

OPcache is tuned for [Flarum](https://flarum.org/) — large composer autoloader, many files, JIT enabled.

---

### Images

| Image | Tag | Dockerfile | Description |
|-------|-----|------------|-------------|
| `ianmgg/php83fpm` | `latest` | `Dockerfile.apache` | Production web image. Apache mpm_event + PHP-FPM. OPcache on, JIT enabled. |
| `ianmgg/php83fpm` | `dev` | `Dockerfile.apache.dev` | Development web image. Extends `latest`. Adds Xdebug, OPcache disabled. |
| `ianmgg/php83fpm` | `cli` | `Dockerfile.cli` | CLI image for queue workers and websocket servers (Horizon, Reverb). OPcache enabled for long-running processes. |
| `ianmgg/php83fpm` | `cli-dev` | `Dockerfile.cli.dev` | Development CLI image. Extends `cli`. Adds Xdebug, OPcache disabled. |

---

### Building locally

All commands are run from the **repository root** (the build context must be the repo root so `COPY fpm/8.3/...` paths resolve correctly).

```bash
# Production web image
docker buildx build -f fpm/8.3/Dockerfile.apache -t ianmgg/php83fpm:latest .

# Development web image (requires latest to be built or available on Docker Hub first)
docker buildx build -f fpm/8.3/Dockerfile.apache.dev -t ianmgg/php83fpm:dev .

# CLI image
docker buildx build -f fpm/8.3/Dockerfile.cli -t ianmgg/php83fpm:cli .

# Development CLI image (requires cli to be built or available on Docker Hub first)
docker buildx build -f fpm/8.3/Dockerfile.cli.dev -t ianmgg/php83fpm:cli-dev .
```

To build for a specific platform:

```bash
docker buildx build --platform linux/amd64 -f fpm/8.3/Dockerfile.apache -t ianmgg/php83fpm:latest .
docker buildx build --platform linux/arm64 -f fpm/8.3/Dockerfile.apache -t ianmgg/php83fpm:latest .
```

---

### Architecture

```
┌─────────────────────────────┐
│  Apache (mpm_event :8080)   │
│  mod_proxy_fcgi             │
└────────────┬────────────────┘
             │ Unix socket
             │ /var/run/php-fpm.sock
┌────────────▼────────────────┐
│  PHP-FPM 8.3                │
│  OPcache + JIT              │
└─────────────────────────────┘
```

Both processes start via `/usr/local/bin/startup` (`php-fpm -D` then `apache2-foreground`).

---

### OPcache settings (production)

| Setting | Value | Reason |
|---------|-------|--------|
| `memory_consumption` | 256MB | Flarum + extensions have a large file set |
| `max_accelerated_files` | 20000 | Covers Flarum core + composer dependencies |
| `interned_strings_buffer` | 32MB | Reduces string duplication across cached files |
| `validate_timestamps` | 0 | No file stat on each request — restart FPM after deploys |
| `jit` | 1255 | Tracing JIT, best for repeated request patterns |
| `jit_buffer_size` | 128MB | |
| `enable_cli` | 1 | Benefits Horizon workers and websocket servers |
