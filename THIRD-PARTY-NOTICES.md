# Third-party notices

Images built from this repository bundle third-party data and software that
carry their own licence terms. Those terms are listed here, along with what they
require of anyone using the images.

## DB-IP Country Lite

- **Used in:** `php83`/`php85` FPM images (apache, cli); `php85` images (apache, cli)
- **Installed at:** the path in `GEOIP_DB_PATH` (`/usr/share/GeoIP/dbip-country-lite.mmdb`)
- **Licence:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
- **Source:** https://db-ip.com/db/download/ip-to-country-lite

The database is redistributed unmodified. A fresh edition is downloaded each
time the images are rebuilt, so the bundled data tracks DB-IP's monthly release.

### What CC BY 4.0 requires

Attribution. If your application surfaces results derived from this database,
you must credit DB-IP wherever those results are displayed. For web
applications DB-IP asks for a hyperlinked credit:

```html
<a href="https://db-ip.com">IP Geolocation by DB-IP</a>
```

This obligation passes downstream: it applies to applications built on these
images, not only to the images themselves. The images cannot satisfy it for you,
because the requirement attaches to the pages that display the results.

The same notice is written into each image at
`/usr/share/GeoIP/ATTRIBUTION.txt`, so it is discoverable from a running
container without reference to this repository.
