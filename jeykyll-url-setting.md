---
title: Jeykyll URL setting
---

This is caused by **Jekyll’s permalink rules** and usually depends on whether the Markdown file is a **post**, a **page**, or has custom YAML front matter.

A URL ending in `/` normally corresponds to a directory containing `index.html`:

```text
filename/
└── index.html
```

So GitHub Pages can serve it as:

```text
https://username.github.io/filename/
```

A URL ending in `.html` corresponds to a file directly:

```text
filename.html
```

and is served as:

```text
https://username.github.io/filename.html
```

Jekyll’s default permalink style commonly produces `.html` URLs, while the `pretty` style produces directory-style URLs with trailing slashes. Jekyll’s documentation describes permalinks as the output paths for generated pages, and they can be set globally or per file. <citation src="2"></citation>

For example, this Markdown file:

```text
_posts/2026-09-22-filename.md
```

might generate:

```text
/2026/09/22/filename.html
```

with the default configuration, but:

```text
/2026/09/22/filename/
```

with a pretty permalink configuration.

To make an individual Markdown page use the slash-style URL, add this at the very top:

```yaml
---
permalink: /filename/
---
```

Jekyll will generate something equivalent to:

```text
_filename/
└── index.html
```

The source file can still be named differently, for example:

```text
my-document.md
```

with:

```yaml
---
permalink: /filename/
---
```

To use pretty URLs more broadly, add this to `_config.yml`:

```yaml
permalink: pretty
```

For posts, this produces date-based slash URLs such as:

```text
/2026/09/22/filename/
```

rather than:

```text
/2026/09/22/filename.html
```

For a custom site-wide structure, you could use:

```yaml
permalink: /:title/
```

Then a page named `filename.md` generally becomes:

```text
/filename/
```

Be aware that pages and posts are treated differently. Jekyll’s global permalink setting primarily controls posts; regular pages can have their own defaults or explicitly override the URL using front matter. <citation src="2"></citation>

Also check whether the files have different front matter. One may contain:

```yaml
---
permalink: /filename/
---
```

while another contains:

```yaml
---
permalink: /filename.html
---
```

If you want consistent links inside your Markdown or templates, use Jekyll’s `link` tag where possible:

```liquid
{% link filename.md %}
```

That lets Jekyll use the page’s actual configured permalink instead of making you guess whether it ends in `/` or `.html`. <citation src="3"></citation>
