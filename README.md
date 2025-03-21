# liuzihe02.github.io
Zihe's personal website

## Notes

- The latest version of Hugo `v0.145.0` causes some errors so we use `dpkg` and the GitHub repo to download an older version `v0.127.0` instead
- The `yaml` file is a GitHub actions workflow that automates the build and deployment process to GitHub Pages
- Modify the `hugo.toml` for site settings

- To build your site (publish the files to the `public` directory, simply run `hugo`)
- To develop and test your site, run `hugo server` in the root directory which builds and serves our site using a minimal HTTP server
- The `hugo.yaml` is a `CI/CD` workflow where a push to GitHUb automatically triggers building and deployment

## Hugo

Note that the theme layout files are not present in the layout folder, only in the site layout folder. Ed provides for layouts out of the box `['posts', 'dramas', 'narratives', 'poems']`. We need to create our own layout if we wish to. Same goes for the custom stylesheet files.

The Hugo [content management](https://gohugo.io/content-management/) sites especially on [sections](https://gohugo.io/content-management/sections/) and [pages](https://gohugo.io/content-management/page-bundles/) are especially relevant.

[Module management](https://gohugo.io/hugo-modules/use-modules/) like cleaning the module cache with `hugo mod clean`

## Guides

The follow tutorials were used to learn Hugo and Github Pages:
- [Ed theme](https://gohugo-theme-ed.netlify.app/documentation/)
- [Hugo Quickstart](https://gohugo.io/getting-started/quick-start/)
- [Deploying Hugo on Github Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
