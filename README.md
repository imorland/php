## PHP Docker Images

This repository contains Docker images tailored for PHP development. They are optimized for development purposes but can be adapted for production use with a few modifications.

### Maintenance Policy:
We actively maintain images only for PHP versions that are currently supported. Once a PHP version reaches its end-of-life, updates for that version will cease in this repository.

For a list of currently supported PHP versions, refer to the official [PHP Supported Versions](https://www.php.net/supported-versions.php).

### Rebuilds

The images are rebuilt on the 1st and 15th of each month (FPM) or every 14
days (mod_php), which picks up Debian updates. A daily check also rebuilds
them as soon as the official `php` image they are built from has a new PHP
release, or MaxMind releases a new `maxminddb` extension, so security
releases arrive within a day.

### GeoIP data and attribution

Some images bundle offline IP geolocation databases so lookups need no network
access at runtime. They are read via the `maxminddb` PHP extension (or the
`maxmind-db/reader` composer package) and located by environment variable:

| Variable | Database | Provides | Licence |
| --- | --- | --- | --- |
| `GEOIP_DB_PATH` | DB-IP Country Lite | continent, country | CC BY 4.0 |
| `GEOIP_CITY_DB_PATH` | DB-IP City Lite | + city, subdivision, lat/lon | CC BY 4.0 |
| `GEOIP_ASN_DB_PATH` | DB-IP ASN Lite | ASN, organisation | CC BY 4.0 |

Currently included in: `php83`/`php85` FPM (apache, cli) and `php85` (apache, cli).

The `maxminddb` extension is built from MaxMind's latest release rather than
PECL, which lags behind it. Each `maxmind-db/reader` release requires the
matching extension version, so the daily upstream check (below) also rebuilds
the images whenever MaxMind releases, and the reader can be bumped freely.

All three are redistributed unmodified and refreshed whenever the images are
rebuilt. `GEOIP_DB_PATH` keeps its original name for backwards compatibility.

Note that the city database's accuracy varies by range — it is a Lite edition,
and coordinates for some consumer ranges can be out by a considerable distance.
Treat city and lat/lon as approximate.

> **If you use this data, attribution is required.**
> CC BY 4.0 obliges you to credit DB-IP wherever results derived from these
> databases are displayed. For web applications DB-IP asks for
> a hyperlinked credit on the pages showing those results:
>
> ```html
> <a href="https://db-ip.com">IP Geolocation by DB-IP</a>
> ```
>
> This obligation is inherited by anything built on these images — it cannot be
> discharged on your behalf by the image itself.

A copy of this notice ships inside the image alongside the databases, at
`/usr/share/GeoIP/ATTRIBUTION.txt`. See [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md)
for the full terms.
