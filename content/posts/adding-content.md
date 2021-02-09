---
title: "Adding content"
description: "How to use Hugo to write new blog posts"
draft: false
author: "Robert Marciniak"
date: 2021-02-05T14:24:45+01:00
tags: ["blog","content"]
categories: ["getting-started"]
---
<!--more-->
# Adding content to Hugo

## Prerequisites

- git
- hugo
- web browser

## Checking out repo

```shell
git clone --recurse-submodules https://github.com/AutoForecastTeam/autoforecastteam.github.io.git
```

## Creating new page

To create a blog post run

```shell
hugo new posts/<post-name>.md
```

This is going to create new markdown page and fill out [front matter](https://gohugo.io/content-management/front-matter/) of the post, according to `/archetypes/default.md`. Front-matter is basically a metadata for any content created in Hugo, and `/archetypes/default.md` is a template that's used to create new posts.
It's convenient, because it can set up title, date and other fields automatically when used with `hugo new ...`

Alternatively one can just add new markdown file under `content/posts` and fill out the front-matter by themselves.

One thing worth noting - default template includes `draft: true` in front-matter. It means that post is marked as draft, and will not be displayed in final website render, after deployment.
To include post in rendered page one should change the flag to `false`

## Previewing changes

```shell
hugo server # --buildDrafts --disableFastRender
```

This command will, when executed from repo root, will start up a local hugo development server, by default under `http://localhost:1313`. The server includes a file watcher, so any file changes will be reflected in browser automatically.

Commented-out settings are related to including drafts in rendering and a more robust rebuild process on file changes.

## Deploying changes

`autoforecastteam.github.io` repository has github workflow attached, which listens for changes on `main` branch and deploys rendered website to `gh-pages` branch.

What it means is that every commit pushed to `origin/main` will trigger website rebuild and new stuff will show up after deploy action is completed.

## Front-matter README

There are a few bugs related to front-matter and some things behave unexpectedly. Below are the proposed workarounds, until it gets fixed by theme author.

### Author

When adding `author: "<Author Name>"` to front-matter one must include appropriately named `<author>.json` entry in `/data/authors/`. This is a bit cumbersome, but it only needs to be added once and will probably get fixed soon.

### Tags / categories

Empty strings (`""`) in tags or categories result in hugo errors.

This results in error:
```yaml
# ...
tags: [""]
categories: ["",""]
# ...
```

while this doesn't
```yaml
# ...
tags: [] # <- empty array
categories: ["anything", "other", "than empty", "string"]
# ...
```

## External documentation

- [Hugo](https://gohugo.io/documentation/)
- uBlogger theme
  - [docs](https://ublogger.netlify.app/categories/documentation/)
  - [preview](https://ublogger.netlify.app/)
  - [repo](https://github.com/uPagge/uBlogger)
