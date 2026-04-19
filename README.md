# seoul.rs

## How to write your content

1. Write your article in Markdown format with the following heading.  
```
$ cd content/blog
$ cat john-smith-first-blog.md
+++
title = "John Smith's first blog"
date = "2026-03-02"
[taxonomies]
authors = ["John Smith", "John's Cat"]
tags = ["rust", "hello"]
+++

My first blog here.
```

2. Preview with zola

Make sure you initialise the `themes/zola-bearblog` git submodule.

```sh
    $ git submodule update --init --recursive
```

Either install zola by one of [these methods](https://www.getzola.org/documentation/getting-started/installation/)
```sh
    $ zola serve
```

or run `docker compose up`
