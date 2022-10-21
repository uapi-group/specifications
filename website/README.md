# Static website generation for UAPI group specifications

This repository uses Hugo for static HTML generation.
See https://gohugo.io/getting-started/quick-start/ for a brief intro.

The website uses the [hugo-book](https://github.com/alex-shpak/hugo-book) theme; it is included in this repo as a git submodule.
After cloning this repo please run `git submodule init; git submodule update`.
If you check out a branch or tag, make sure the submodule is up to date by running `git submodule update`.

## Website repo layout

Content resides in the [content](content/) folder.
Top-level intro pages and menu entries should be put there.
The repository's [specs](../specs) folder is soft-linked there (as [content/docs](content/docs)).
New specs are automatically added to the navigation menu on the left.
Optionally, the ([index](content/_index.md) page may be updated to reference / feature important specs.

## Making changes and testing

You'll need [hugo installed](https://gohugo.io/getting-started/installing/) for rendering changes.

First, make your edits.
Then, start hugo locally (in the repo's `website` directory)to review your changes:

```shell
$ hugo server --minify --disableFastRender
```

Review your changes at http://localhost:1313/specifications/ .
