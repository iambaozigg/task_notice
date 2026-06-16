# CloudBase SDK Vendor Files

The app first tries the local v2 full bundle:

```text
vendor/cloudbase.full.js
```

If that is unavailable, it falls back to the official CloudBase v2.28.6 CDN full bundle.

You may also provide the v2 module files as a secondary local fallback:

```text
vendor/cloudbase.js
vendor/cloudbase.auth.js
vendor/cloudbase.database.js
```

Only auth and document database modules are needed for this app.
