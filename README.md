# packit.dev

[packit.dev](https://packit.dev) website content

## Clone

This repository uses git submodules, so you have to `git clone --recurse-submodules` it.
If you forget and realize later, just run:

    $ git submodule init themes/book
    $ git submodule update

## Hugo

The website is generated by [Hugo](https://gohugo.io) (see also the [documentation](https://gohugo.io/getting-started)).
If you want to contribute to our website content and want to see how your contribution would look like,
you have to install Hugo on your machine.

#### Fedora

    $ dnf install hugo
    $ hugo help

#### CentOS/Epel

    $ dnf copr enable daftaupe/hugo
    $ dnf install hugo

#### MacOS

    $ brew install hugo

### Add new post

    $ hugo new posts/packit-xyz.md

### Content

All content [is organized](https://gohugo.io/content-management/organization)
in [content](content/) directory tree.

A newly `hugo new` created content file contains only `title` and `date`
metadata in the [front matter](https://gohugo.io/content-management/front-matter).
You should also add `weight` to re-arrange your post in the file-tree menu.
If you want your post to be the uppermost one, set the `weight` to a value
(one) lesser than the currently last/uppermost post has.

### Start Hugo server

1. `hugo server`
2. [Web Server](http://localhost:1313)

### Rebuild content & GitHub Pages

It's done automatically with each push to main. We use
[Hugo Deploy GitHub Pages Action](https://github.com/marketplace/actions/hugo-deploy-github-pages)
configured in [.github/workflows/deploy-pages.yml](.github/workflows/deploy-pages.yml)
which pushes the generated content into
[packit/packit.dev-github-pages](https://github.com/packit/packit.dev-github-pages)
from where the GitHub Pages are served.
The secret used by the action is stored in
[settings/secrets](https://github.com/packit/packit.dev/settings/secrets).

### Themes

Currently, we use [Book theme](https://themes.gohugo.io/hugo-book).
For complete list of themes for Hugo, see [this](https://themes.gohugo.io).
If you want to use a new theme:

1. `git submodule init themes/<theme_name>`
2. `git submodule update`
3. set `theme = "theme_name"` in [config.toml](config.toml)

### Site Configuration

[Configuration](https://gohugo.io/getting-started/configuration/)
is stored in [config.toml](config.toml).
