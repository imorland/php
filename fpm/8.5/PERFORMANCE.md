## PHP 8.5 FPM - Performance Findings

Why the OPcache, JIT, FPM pool and `mpm_event` values in these images are what they are.

Measurements were taken against `ianmgg/php85fpm:latest` / `:cli` (PHP 8.5.9, `linux/arm64`) with no application code deployed — so per-worker figures are floors, not Flarum numbers. Everything labelled *measured* is reproducible with the commands at the end; everything else is sourced and linked.

---

### JIT

#### The defaults inverted in PHP 8.4

| Directive | Default | Note |
|-----------|---------|------|
| `opcache.jit` | `disable` | Was `tracing` before 8.4. `disable` cannot be re-enabled at runtime; `off` can |
| `opcache.jit_buffer_size` | `64M` | Was `0`. A zero value disables the JIT outright |

The documented aliases are `tracing` = `1254` and `function` = `1205`. These images previously set `1255`, which is not the `tracing` alias — the trailing digit is the optimisation level, so `1255` is O=5 (*optimize whole script*) against the stock O=4 (*use call graph*). It appears widely in blog posts but is not a documented recommendation, and nothing in this repo needed it. Now set to the alias.

The 4-digit form is `CRTO`: CPU flags, register allocation, trigger, optimisation level.

#### The buffer is allocated on top of `memory_consumption`

The manual is explicit that interned strings come *out of* the OPcache pool, but says nothing either way about the JIT buffer. Measured — total address space with `memory_consumption = 256`:

| `jit_buffer_size` | VmSize | VmRSS |
|-------------------|--------|-------|
| `128M` (old value) | 520 MB | 47 MB |
| `64M` (current) | 456 MB | 47 MB |
| `32M` | 424 MB | 47 MB |
| `0` | 392 MB | 47 MB |
| `0` + `opcache.enable_cli=0` | 136 MB | 38 MB |

Each step matches the buffer size exactly, so the buffer is an additional shared-memory allocation: the real reservation is `memory_consumption + jit_buffer_size` = 320 MB at current settings, of which 32 MB is interned strings taken out of the 256 MB, leaving ~224 MB for bytecode.

**The reservation is virtual, not resident.** RSS is flat at 47 MB across every buffer size — it is an `mmap` that fills as bytecode and traces are actually compiled. Consequences:

- cgroup / `docker --memory` limits count RSS, so the reservation does not push a container toward its limit.
- Strict overcommit (`vm.overcommit_memory=2`) does account for it, and it is multiplied per process for CLI, where each process maps its own segment rather than inheriting the master's.

#### JIT does close to nothing for Flarum — but not because it is I/O bound

Calling Flarum I/O bound is wrong. A warm API request is roughly **60–75% CPU in PHP**: FPM keeps no application between requests, so every request pays a full framework boot, and an SPA client turns one interaction into several requests, each booting again. Request *count* per interaction goes up, and with it total CPU.

JIT still does not help, because of *where* that CPU goes:

- Container resolution and reflection during boot — type inference cannot see through it.
- Internal C functions (`json_encode`, `preg_*`, `array_*`, PDO) — already native; there is nothing to compile.
- Polymorphic object graphs in serialization — defeats the type inference JIT depends on.
- Most boot code runs once or twice per request, so tracing has little that gets hot. Traces do persist across requests in a long-lived pool, which is why the result is "no measurable gain" rather than a regression.

JIT wins on monomorphic scalar and loop code with inferable types. Framework request handling is the textbook case where it does not apply — php.watch measured Laravel ~2% *worse* with JIT on. The CPU-heavy paths in these images (imagick, ghostscript, PCRE, bcrypt) are all C already.

It is also not free: 64 MB of reservation, and JIT-compiled code is opaque to debuggers and profilers.

Hence the flag (see README) with `off` reclaiming the buffer. The default stays `tracing` to preserve the prior behaviour of these images.

#### arm64 is a supported JIT backend

php-src's `ext/opcache/jit/README.md`: *"The JIT supports 3 different architectures: `X86_64`, `i386`, and `arm64`."* The frequently repeated "JIT is x86-only, via DynASM" claim predates the IR-based JIT that replaced DynASM in 8.4. The `linux/arm64` variants of these images need no JIT caveat — verified by building and running on arm64, where `opcache.jit=tracing` reports `jit.on = true`.

---

### OPcache

| Finding | Detail |
|---------|--------|
| `max_accelerated_files` rounds up to a prime | From a fixed set; `20000` yields **32531** slots (measured). No reason to raise it for Flarum |
| `interned_strings_buffer` is inside the pool | The 32 MB comes out of `memory_consumption`, not on top of it |
| `max_wasted_percentage` default 5% is a trap | Below the threshold OPcache never restarts, so scripts that no longer fit are recompiled *every request*, silently, as if OPcache were absent. Raised to 10 |
| `validate_timestamps = 0` needs a restart | No stat per request; code changes require restarting FPM. Correct for immutable container deploys, which is how these images ship |
| `enable_cli = 1` maps a segment per process | CLI has no master to inherit from. Kept on: long-running queue workers amortise it, and RSS impact measured at ~9 MB |
| OPcache is mandatory in 8.5 | Always compiled into the binary; can still be disabled by ini, which is what the dev images do |
| `opcache.file_cache_read_only` is new in 8.5 | Lets a warm file cache be baked at build time and mounted read-only — a genuine cold-start win for containers. Not used here yet |

Monitor via `opcache_get_status()`: `cache_full` with `num_cached_keys == max_cached_keys` → raise `max_accelerated_files`; `cache_full` with low waste → raise `memory_consumption`; frequent `oom_restarts` → memory; frequent `hash_restarts` → file count; hit rate below 99% → memory pressure.

OPcache memory must be set in `php.ini` / `conf.d`, never in an FPM pool file — the shared memory is allocated before pool config is applied. These images use `conf.d/flarum.ini`, which is correct.

---

### Apache and the FPM pool

**Debian's `mpm_event` default did not match the pool.** The stock file ships `MaxRequestWorkers 150` / `ThreadsPerChild 25` against a pool of 10 workers — Apache accepted 15× what PHP could serve, and with `Timeout 300` the surplus waited up to five minutes in the FCGI queue. httpd's own `mod_proxy_fcgi` documentation warns about exactly this relationship. Now `MaxRequestWorkers 50` against 24 workers (measured at runtime: 2 children × 25 worker threads).

**Connection reuse is deliberately left off.** `mod_proxy_fcgi` disables it by default for the `SetHandler "proxy:unix:…|fcgi://localhost"` form. With `enablereuse=on` the pool can grow to `MaxRequestWorkers` backend connections, and httpd warns that workers may all end up busy on idle persistent connections, producing *"a pile of HTTP request timeouts"*. A Unix-socket connect is cheap enough that reuse is not worth that failure mode.

**`request_terminate_timeout` is the only real backstop.** `max_execution_time` does not tick while a worker is blocked in a socket read, so it cannot reap a worker hung on a database or HTTP call. Set to 310s, just above `max_execution_time = 300` so PHP's own limit fires first in the normal case.

**Worker sizing.** Measured idle floor is ~14 MB RSS per worker with no application (Apache children ~8 MB); budget 60–100 MB for real Flarum. On 4 CPUs / 4 GB with the database off-host that supports ~24 workers — beyond which only I/O wait is being covered, since 4 CPUs run 4 requests at a time. An SPA client raises concurrency per *user* well above 1: several API calls per interaction, and browsers open up to 6 connections per host.

Note the arithmetic that does not close: `memory_limit = 256M` × 24 workers is a 6 GB ceiling on a 4 GB box. It is a per-request ceiling rather than typical use, but these images carry imagick and ghostscript, which is precisely the workload that reaches for it. Size from `pm.status_path` under real traffic.

**The dev images are unaffected by all OPcache and JIT tuning.** `flarum-dev.ini` sets `opcache.enable = 0`, which forces `jit = disable` / `jit.on = NULL` regardless of the flag (measured). The pool settings *are* inherited, which helps: with Xdebug attached, a request paused on a breakpoint holds its worker while the SPA keeps issuing API calls, and 24 workers removes the starvation the old 10 allowed. Cost of the pre-warm in the dev image: 126 MB total RSS for 9 idle processes.

---

### Reproducing the measurements

```bash
# Address space and pool for any JIT setting
docker run --rm -e PHP_OPCACHE_JIT=off ianmgg/php85fpm:cli php -r '
$s = opcache_get_status(false);
preg_match("/VmSize:\s+(\d+)/", file_get_contents("/proc/self/status"), $v);
printf("jit_on=%s buffer=%dMB pool=%dMB slots=%d VmSize=%dMB\n",
  var_export($s["jit"]["on"], true), $s["jit"]["buffer_size"] / 1048576,
  ($s["memory_usage"]["used_memory"] + $s["memory_usage"]["free_memory"]) / 1048576,
  $s["opcache_statistics"]["max_cached_keys"], $v[1] / 1024);'

# Per-worker RSS and Apache thread layout
docker exec <container> ps -eo rss=,comm= --sort=-rss
docker exec <container> ps -eLf | awk '/apache2/ {c[$2]++} END {for (p in c) print p, c[p]}'
```

---

### Sources

- [php.net: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php) — defaults, CRTO digits, prime rounding, interned strings
- [php-src: `ext/opcache/jit/README.md`](https://github.com/php/php-src/tree/master/ext/opcache/jit) — supported architectures
- [php.watch: Opcache JIT INI changes in 8.4](https://php.watch/versions/8.4/opcache-jit-ini-default-changes)
- [php.watch: PHP JIT in Depth](https://php.watch/articles/jit-in-depth) — buffer sizing, Laravel measurement, debugger opacity
- [Tideways: Fine-tune your OPcache configuration](https://tideways.com/profiler/blog/fine-tune-your-opcache-configuration-to-avoid-caching-suprises) — monitoring thresholds, wasted-memory behaviour
- [Tideways: An introduction to PHP-FPM tuning](https://tideways.com/profiler/blog/an-introduction-to-php-fpm-tuning)
- [Tideways: What's new in PHP 8.5 for performance and operations](https://tideways.com/profiler/blog/whats-new-in-php-8-5-in-terms-of-performance-debugging-and-operations)
- [httpd: `mod_proxy_fcgi`](https://httpd.apache.org/docs/current/mod/mod_proxy_fcgi.html) — connection reuse and pool sizing
