# alexy-dot's blog

Source for [alexy-dot.github.io](https://alexy-dot.github.io/).

Do not edit `gh-pages` by hand. Write content on `master`, push it, and let GitHub Actions build the static site into `gh-pages`.

## Local Preview

```shell
pip install -r requirements.txt
mkdocs serve
```

## Writing Flow

- Put future posts under `docs/Blogs/posts/`.
- Put reusable images under `docs/assets/images/`.
- Add a page to `mkdocs.yml` only after the Markdown file exists.
- Deploy by pushing `master`; `.github/workflows/mkdocs-deploy.yml` runs `mkdocs gh-deploy --force`.
