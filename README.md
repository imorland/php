## PHP Docker Images

This repository contains Docker images tailored for PHP development. They are optimized for development purposes but can be adapted for production use with a few modifications.

### Maintenance Policy:
We actively maintain images only for PHP versions that are currently supported. Once a PHP version reaches its end-of-life, updates for that version will cease in this repository.

For a list of currently supported PHP versions, refer to the official [PHP Supported Versions](https://www.php.net/supported-versions.php).

### GeoIP data and attribution

Some images bundle an offline IP geolocation database so lookups need no network
access at runtime. Where present, it is installed at the path given by the
`GEOIP_DB_PATH` environment variable and read via the `maxminddb` PHP extension.

Currently included in: `php83`/`php85` FPM (apache, cli) and `php85` (apache, cli).

The bundled database is **DB-IP Country Lite**, licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is redistributed
unmodified, and refreshed whenever the images are rebuilt.

> **If you use this data, attribution is required.**
> CC BY 4.0 obliges you to credit DB-IP wherever results derived from the
> database are displayed. For web applications DB-IP asks for a hyperlinked
> credit on the pages showing those results:
>
> ```html
> <a href="https://db-ip.com">IP Geolocation by DB-IP</a>
> ```
>
> This obligation is inherited by anything built on these images — it cannot be
> discharged on your behalf by the image itself.

A copy of this notice ships inside the image alongside the database, at
`/usr/share/GeoIP/ATTRIBUTION.txt`. See [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md)
for the full terms.
