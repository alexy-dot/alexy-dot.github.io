# alexy-dot.github.io

This is the source branch for [alexy-dot.github.io](https://alexy-dot.github.io/).

Do not edit `gh-pages` by hand. Write content on `master`, push it, and let GitHub Actions build the static site into `gh-pages`.

## Install

```shell
conda create -f environment.yml

pip cache purge
```

## Writing Flow

- Put posts under `docs/Blogs/posts/`.
- Put reusable images under `docs/assets/images/`.
- Add new pages to `mkdocs.yml` only after the Markdown file exists.
- Preview locally with `mkdocs serve`.
- Deploy by pushing `master`; `.github/workflows/mkdocs-deploy.yml` runs `mkdocs gh-deploy --force`.
