---
title: Flarum folder arrangerment
date: 2026-10-06 08:10:05
---

^The webhosting I'm using is forceing htdocs folder as index folder and no change for that? Can I upload every folders except Public into same path as htdocs and then upload everything in Public into  htdocs/ ? Does this work?

Yes, that arrangement can work when `htdocs` is forced to be the web root.

Upload the **contents** of Flarum’s `public` folder directly into `htdocs`, then place the remaining Flarum files and folders in that same `htdocs` directory:

```text
htdocs/
├── index.php              ← from public/
├── .htaccess              ← from public/
├── assets/                ← from public/
├── api.php                ← from public/, if present
├── config.php
├── site.php
├── vendor/
├── storage/
├── extend.php
├── composer.json
└── ...
```

Do not create:

```text
htdocs/public/
```

The files inside `public` should be directly inside `htdocs`.

For Flarum 1.8.10, edit `htdocs/index.php` and make sure this line is:

```php
$site = require './site.php';
```

Because `site.php` is now in the same directory as `index.php`, you should not use `../site.php`.

In `htdocs/site.php`, use paths appropriate for everything being in the same directory:

```php
'base' => __DIR__,
'public' => __DIR__,
'storage' => __DIR__ . '/storage',
```

Also check `htdocs/config.php` and set the URL without `/public`:

```php
'url' => 'https://example.com',
```

The important security step is `.htaccess`. Flarum’s normal `public/.htaccess` does not necessarily protect sensitive files when you remove the `public` directory. Open it and uncomment the protection rules specified by Flarum—lines 9–15 in the relevant Flarum 1.8 file. These rules help prevent public access to files and folders such as:

```text
config.php
site.php
composer.json
composer.lock
storage/
vendor/
```

Flarum’s documentation explicitly supports this setup for hosts that force `public_html` or `htdocs`: move the contents of `public`, including `.htaccess`, into the web root, enable the protection rules, update `index.php` and `site.php`, and verify `config.php`. <citation src="1"></citation>

After uploading, visit:

```text
https://example.com
```

If you see a directory listing, a 404, or Flarum routes fail after the homepage loads, ask the hosting provider to confirm that Apache `mod_rewrite` and `.htaccess` overrides are enabled. Flarum requires those for URL rewriting. <citation src="1"></citation>