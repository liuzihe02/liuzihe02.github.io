# liuzihe02.github.io
Zihe's personal website using the Hugo [Ed](https://themes.gohugo.io/themes/gohugo-theme-ed/) theme.

## Ed

This version of Ed is written with Hugo, where the documentation is available [here](https://gohugo-theme-ed.netlify.app/documentation/). An [older](https://github.com/minicomp/ed) version is written in Jekyll.

## Hugo

- Modify the `hugo.toml` for site settings
- To build your site (publish the files to the `public` directory, simply run `hugo`)
- To develop and test your site, run `hugo server` in the root directory which builds and serves our site using a minimal HTTP server

> The latest version of Hugo `v0.145.0` causes some errors so we use `dpkg` and the GitHub repo to download an older version `v0.127.0` instead

Note that the theme layout files are not present in the layout folder, only in the site layout folder. Ed provides for layouts out of the box `['posts', 'dramas', 'narratives', 'poems']`. We need to create our own layout if we wish to. Same goes for the custom stylesheet files.

The Hugo [content management](https://gohugo.io/content-management/) sites especially on [sections](https://gohugo.io/content-management/sections/) and [pages](https://gohugo.io/content-management/page-bundles/) are especially relevant.

[Module management](https://gohugo.io/hugo-modules/use-modules/) like cleaning the module cache with `hugo mod clean`

### Adding Content

Only use the `content/` folder to add new content

### Modifying Styles

You will see that alot of `scss` files are stored in the `resources/_gen/assets/` directory. Do not modify this file. Hugo will modify this stuff automatically whenever you build your project, so your changes will be lost.

Instead, use the `assets/`, `layouts/`,`static/` direcotry in the project root. Copy the relevant file from the github repo theme and modify it there. Hugo will use the file locally before using the version online. I've tried to keep the folder structure as similar to Ed's structure as I can.

## Deployment

- The `yaml` file is a GitHub actions workflow that automates the build and deployment process to GitHub Pages
- The `hugo.yaml` is a `CI/CD` workflow where a push to GitHUb automatically triggers building and deployment

## Guides

The follow tutorials were used to learn Hugo and Github Pages:
- [Ed theme](https://gohugo-theme-ed.netlify.app/documentation/)
- [Hugo Quickstart](https://gohugo.io/getting-started/quick-start/)
- [Deploying Hugo on Github Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
