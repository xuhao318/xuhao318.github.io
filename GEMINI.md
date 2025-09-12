# Project Overview

This is a personal blog created with [Hugo](https://gohugo.io/), a static site generator. The blog uses the "hugo-geekblog" theme. The content of the blog is written in Markdown and the website is deployed to GitHub Pages.

# Building and Running

To build and run the project locally, you need to have Hugo installed. You can find the installation instructions in the [Hugo documentation](https://gohugo.io/getting-started/installing/).

Once Hugo is installed, you can run the following command to start a local server:

```bash
hugo server
```

This will start a local server and you can view the website at `http://localhost:1313/`.

To build the website, you can run the following command:

```bash
hugo
```

This will generate the static website in the `public` directory.

# Development Conventions

The blog posts are located in the `content/posts` directory. To create a new post, you can use the following command:

```bash
hugo new posts/my-new-post.md
```

This will create a new Markdown file in the `content/posts` directory with some predefined front matter.

The configuration of the website is in the `hugo.toml` file. This file contains the title of the website, the theme, and other configuration options.

The theme used is "hugo-geekblog". The theme's files are located in the `themes/hugo-geekblog` directory. The theme's README suggests that it can be customized, but it requires building assets with `webpack`.
