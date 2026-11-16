# abruption.github.io

This repository contains the source for the production blog at [blog.abruption.dev](https://blog.abruption.dev/). The site is built with Jekyll and the [Chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy).

> **This is an existing production blog checkout. Do not run `bash tools/init.sh` here, or in any checkout that contains blog content.**
>
> `tools/init.sh` is a one-time initializer for a fresh fork of the theme, not a reset or publishing command. Its initialization check uses only a few markers. If it fails to recognize an existing checkout, it can remove `_posts/*`, `docs`, and existing GitHub workflow files, then create a commit. With `--no-gh`, it removes `.github` as well. It is not part of the normal blog workflow.

## Creating a separate Chirpy site

For a new blog, start with the upstream [Chirpy Starter](https://github.com/cotes2020/chirpy-starter) and follow its [Getting Started guide](https://chirpy.cotes.page/posts/getting-started/). The upstream guide covers initialization for a fresh theme fork; follow that only in a separate, new repository. Do not use this production blog repository as a starter.

## Writing and reviewing posts

- Keep unpublished drafts in `_drafts/`. The production build does not include drafts.
- When a post is ready for review, add it under `_posts/` using the Jekyll filename format: `YYYY-MM-DD-short-title.md`.
- Follow the front matter and content conventions of neighboring posts. Set `mermaid: true` when the post contains Mermaid diagrams.

## Local preview and validation

The GitHub Actions workflows use **Ruby 3.2**. Use the same Ruby major/minor version locally to catch dependency differences before opening a pull request.

Install the locked dependencies once:

```sh
bundle install
```

Start the local preview server:

```sh
bundle exec jekyll s
```

Open <http://localhost:4000> to preview the site. For the production build and HTML checks, use the same sequence as CI:

```sh
JEKYLL_ENV=production bundle exec jekyll b
bash tools/test.sh
```

As a convenience, `bash tools/test.sh --build` runs the production build before HTML-Proofer. The tests read the generated `_site/` output; they do not publish it.

## Publishing this blog

1. Create a topic branch from `master`, make the post or documentation change, and open a pull request targeting `master`. Follow the repository's pull request template and branch guidelines.
2. After an approved pull request is merged, a non-ignored push to `master` runs the **Automatic build** workflow. It uses Ruby 3.2, runs the production Jekyll build and `bash tools/test.sh`, then invokes `tools/deploy.sh`.
3. The deployment workflow publishes the generated site to `gh-pages`. GitHub Pages is configured to publish from **GitHub Actions**, not from a manually maintained branch.

The workflow deliberately ignores changes limited to `README.md`, `.gitignore`, or `LICENSE`. A README-only change still runs the pull request CI checks, but does not trigger a public Pages deployment after merge. Changes to `_posts/` do trigger the publishing workflow.

`tools/deploy.sh` is CI-only and refuses to run without `GITHUB_ACTION`. It replaces the ephemeral Actions checkout with the generated site as part of deployment. Do not run it manually or edit `gh-pages` directly.

## Credits

This theme is mainly built with the [Jekyll](https://jekyllrb.com/) ecosystem, [Bootstrap](https://getbootstrap.com/), [Font Awesome](https://fontawesome.com/) and other libraries. Their copyright information is kept in the relevant files.

Thanks to the volunteers who contributed to [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/graphs/contributors), and to those who reported bugs, shared ideas, or improved its documentation.

## Supporting Chirpy

If you find the upstream theme helpful, please consider sponsoring its maintainer through the [Chirpy repository](https://github.com/cotes2020/jekyll-theme-chirpy).

## License

The upstream theme is published under the [MIT License](https://github.com/cotes2020/jekyll-theme-chirpy/blob/master/LICENSE). This repository retains its existing upstream copyright and license notices.
