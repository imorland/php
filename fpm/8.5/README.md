## PHP 8.5 FPM Images

PHP 8.5 images built on `php:8.5-fpm` (rolling, not pinned to a Debian release).
Apache runs in **mpm_event** mode and proxies to PHP-FPM over a Unix socket for better concurrency than the traditional mod_php / mpm_prefork setup.

OPcache is tuned for [Flarum](https://flarum.org/) — large composer autoloader, many files, JIT enabled.

The measurements and reasoning behind the OPcache, JIT, pool and `mpm_event` values are in [PERFORMANCE.md](PERFORMANCE.md).

---

### Images

| Image | Tag | Dockerfile | Description |
|-------|-----|------------|-------------|
| `ianmgg/php85fpm` | `latest` | `Dockerfile.apache` | Production web image. Apache mpm_event + PHP-FPM. OPcache on, JIT enabled. |
| `ianmgg/php85fpm` | `dev` | `Dockerfile.apache.dev` | Development web image. Extends `latest`. Adds Xdebug, OPcache disabled. |
| `ianmgg/php85fpm` | `cli` | `Dockerfile.cli` | CLI image for queue workers and websocket servers (Horizon, Reverb). OPcache enabled for long-running processes. |
| `ianmgg/php85fpm` | `cli-dev` | `Dockerfile.cli.dev` | Development CLI image. Extends `cli`. Adds Xdebug, OPcache disabled. |

---

### Building locally

All commands are run from the **repository root** (the build context must be the repo root so `COPY fpm/8.5/...` paths resolve correctly).

```bash
# Production web image
docker buildx build -f fpm/8.5/Dockerfile.apache -t ianmgg/php85fpm:latest .

# Development web image (requires latest to be built or available on Docker Hub first)
docker buildx build -f fpm/8.5/Dockerfile.apache.dev -t ianmgg/php85fpm:dev .

# CLI image
docker buildx build -f fpm/8.5/Dockerfile.cli -t ianmgg/php85fpm:cli .

# Development CLI image (requires cli to be built or available on Docker Hub first)
docker buildx build -f fpm/8.5/Dockerfile.cli.dev -t ianmgg/php85fpm:cli-dev .
```

To build for a specific platform:

```bash
docker buildx build --platform linux/amd64 -f fpm/8.5/Dockerfile.apache -t ianmgg/php85fpm:latest .
docker buildx build --platform linux/arm64 -f fpm/8.5/Dockerfile.apache -t ianmgg/php85fpm:latest .
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
│  PHP-FPM 8.5                │
│  OPcache + JIT              │
└─────────────────────────────┘
```

Both processes start via `/usr/local/bin/startup` (`php-fpm -D`, then `apache2ctl -D FOREGROUND` after sourcing `/etc/apache2/envvars`).

---

### OPcache settings (production)

| Setting | Value | Reason |
|---------|-------|--------|
| `memory_consumption` | 256MB | Flarum + extensions have a large file set |
| `max_accelerated_files` | 20000 | Covers Flarum core + composer dependencies |
| `interned_strings_buffer` | 32MB | Reduces string duplication across cached files |
| `validate_timestamps` | 0 | No file stat on each request — restart FPM after deploys |
| `jit` | tracing | Documented alias (= 1254). Flarum is I/O bound, so set `off` when profiling |
| `jit_buffer_size` | 64MB | Shared memory allocated on top of `memory_consumption`, not out of it |
| `enable_cli` | 1 | Benefits Horizon workers and websocket servers |

---

### JIT feature flag

JIT is on by default (`tracing`, 64MB buffer) and can be switched per container. `opcache.jit_buffer_size` is `INI_SYSTEM`, so the flag is resolved before PHP starts, into `conf.d/zz-jit.ini`, which loads after `flarum.ini`. It is applied by the image `ENTRYPOINT` on both the web and CLI images, so it keeps working for consuming images that replace `/usr/local/bin/startup` with their own script.

| Variable | Values | Effect |
|----------|--------|--------|
| `PHP_OPCACHE_JIT` | `1`, `on`, `true`, `tracing` | Tracing JIT |
| | `function` | Function JIT |
| | `0`, `off`, `false`, `disable` | JIT off, buffer dropped to 0 — reclaims 64MB of reservation |
| | unset | Keeps the `flarum.ini` default |
| `PHP_OPCACHE_JIT_BUFFER_SIZE` | e.g. `32M` | Buffer size while JIT is on; ignored when off |

```bash
docker run -e PHP_OPCACHE_JIT=off ianmgg/php85fpm:latest
```

An unrecognised value logs a warning and leaves the default in place. The `dev` and `cli-dev` images disable OPcache entirely, so JIT is off there regardless of the flag.

---

### Sizing

The FPM pool and `mpm_event` are sized together for **4 CPUs / 4GB**, with the database and Redis on other hosts, assuming ~80MB per Flarum worker.

| Setting | Value | Reason |
|---------|-------|--------|
| `pm.max_children` | 24 | 4 CPUs only run 4 requests at once; the rest cover I/O wait. Use 16 (start 6, spare 4–10) when MySQL and Redis share the same 4GB |
| `pm.start_servers` | 8 | Pre-warmed for the request fan-out of one client page load |
| `MaxRequestWorkers` | 50 | ~2x the pool, so overload queues shallowly rather than piling up behind `Timeout` |

A Flarum client issues several API requests per interaction and browsers open up to 6 connections per host, so concurrency per active user is well above 1. Measure `pm.status_path` under real traffic before changing `pm.max_children`; it is not routed through Apache, so reach it with a socket client or add a `ProxyPass` rule.

The `dev` and `cli-dev` images inherit this pool.
