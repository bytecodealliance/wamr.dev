<h1 align="center">
  wamr.dev
</h1>

![Linux](https://github.com/bytecodealliance/wamr.dev/workflows/github%20pages/badge.svg?branch=main)

https://bytecodealliance.github.io/wamr.dev

## Folder structure

``` bash
.
├── archetypes                      # used by hugo, no need to care
├── assets                          # used by hugo, no need to care
├── config                          # global configuration files, used by maintainer
├── content                         # all content stores here, authors only add file here
│   └── en                              # english content
│       └── blog                            # blog posts
│           └── wamr_blog_system                # article name
│               ├── image.png                   # image used by the article
│               └── index.md                    # article content
├── i18n                            # internationalization
│   ├── de.yaml
│   ├── en.yaml
│   └── nl.yaml
├── images                          # default images folder
│   └── tn.png
├── layouts                         # layout templates, used by maintainer
│   ├── ......
│   ├── 404.html
│   └── index.html
├── LICENSE
├── package.json
├── README.md
├── SECURITY.md
├── static
│   ├── ......
│   ├── fonts
│   ├── images
│   └── videos
└── theme.toml                      # theme configuration, used by maintainer
```

## Get started
### Writing blog

All the blogs should put under `content/en/blog/`

- using hugo for local preview

  > install hugo firstly, see the installation guide in [site-development section](#site-development)

  ``` bash
  # create new blog
  hugo new content/en/blog/<article_name>/index.md
  code content/en/blog/<article_name>/index.md
  # write the blog ...
  # local preview
  npm install
  npm run dev
  # Open browser and navigate to preview url ...
  # submit
  ```

- without hugo

  ``` bash
  code content/en/blog/<article_name>/index.md
  # write the blog ...
  # submit
  ```

#### Markdown front matter

Hugo use markdown front matter to get the article information, please ensure your index.md has the following content at the front:

``` bash
---
title: "Your blog title"
description: "your blog description, may be used by search engine"
excerpt: "Same to description, but will displayed on the blog's introduction card"
date: 2022-10-12T21:27:24+08:00
lastmod: 2022-10-12T21:27:24+08:00
draft: false
weight: 50
images: ["image.jpg"]
categories: ["some_category"]
tags: []
contributors: ["your name"]
pinned: false
homepage: false
mermaid: true
---
```

- `title`, `excerpt`, `date`, `images`, `categories` and `contributors` are used to display the cover of this blog

  - if `images` not provided, will use a default image with `WebAssembly` logo

- `draft` must be `false`, otherwise this blog will not be displayed in the final page
- `mermaid` should be `true` if you want to draw mermaid diagram in your blog

### Add an event

Event is almost the same with a blog, it's separated just for management convenience.

The article folder under `content/en/events/` will be treated as an event.

#### Event information

An event require these two additional fields in the markdown header:

- `event_date`, the event date
- `event_location`, the event location

There is no format rule, the raw string will be displayed on the final page.

### Add a resource

Just add the resource link to [content/en/resources/index.md](./content/en/resources/index.md)

### Site development

For site maintainer, please follow this guide to setup development environment.

1. install `hugo`

    Please refer to `hugo`'s [doc](https://gohugo.io/getting-started/installing/) for your OS

    For ubuntu you can use snap:

    ``` bash
    snap install hugo --channel=extended
    ```

    If snap install failed, we can also try the released deb package

    ``` bash
    wget https://github.com/gohugoio/hugo/releases/download/v0.104.3/hugo_extended_0.104.3_linux-amd64.deb
    sudo dpkg -i hugo_extended_0.104.3_linux-amd64.deb
    ```

2. install dependencies

    ``` bash
    npm install
    ```

3. launch dev server

    ``` bash
    npm run dev
    # This will launch a server and serve the blog as localhost:1313
    ```

4. generate static site

    ``` bash
    hugo --minify
    ```

5. deployment

This site is deployed by Github Pages, simply submit the commit and open a PR, the page will automatically updated once PR is merge.

## Acknowledgement

This site uses the hugo theme [doks](https://github.com/h-enk/doks). [LICENSE](https://github.com/h-enk/doks/blob/master/LICENSE)
