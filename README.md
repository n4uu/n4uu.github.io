# n4uu.github.io

Source of https://n4uu.github.io. Hugo, a small hand-written layout, no theme, no JavaScript.

```sh
hugo server -D          # local preview, drafts included
hugo --minify           # what CI builds
```

Posts live in `content/posts/`. A post with `draft: true` is not built by CI. Indicator CSVs per post go in `static/iocs/` (defanged).

Pushing to `main` builds and deploys through GitHub Actions (`.github/workflows/pages.yml`, Hugo pinned and checksum-verified).
