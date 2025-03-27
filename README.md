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

Use ISO 8601 for the datetime added to the frontmatter of the markdown page.

### Modifying Styles

> **Important:** Do not modify the `public` folder or `resources` folder. You will see that alot of `scss` files are stored in the `resources/_gen/assets/` directory. Hugo will modify this stuff automatically whenever you build your project, so any modifications you make will be lost.

Instead, use the `assets/`, `layouts/`,`static/` direcotry in the project root. Copy the relevant file from the github repo theme and modify it there. Hugo will use the file locally before using the online theme for files not found locally. Keep the folder structure as similar to Ed's structure as you can, and make minimum modifications.

> For `layouts/`, we usually modify the stuff in `layouts/partials/`

The `poems/` folder especially `o-captain` contains some very interesting footnotes and stylizations for the `poems` layout

The `narratives/` also contain some very interesting stylizations

#### Shortcodes

The Ed Github `layouts/shortcodes/` contains a bunch of Hugo shortcuts to add a bunch of components like table of contents etc.

##### Images

We use a `img` shortcode

```
{{< img src="/profile.jpeg" alt="A description" width="400" height="400" class="test">}}
```

Put your images under `/static`, and when using the path immediately use the image name without the `/static/profile.jpeg`

##### Hyperlinks

We use a custom `link` shortcode

```
{{< link src="http://gohugo.io" class="external" target="_blank" rel="noopener noreferrer" >}}University of Cambridge{{< /link >}}
```

This creates a hyperlink to "http://gohugo.io" with the visible text "University of Cambridge" and several important attributes:

1. `class="external"` - Adds a CSS class to style links that go to external sites
2. `target="_blank"` - Opens the link in a new browser tab/window
3. `rel="noopener noreferrer"` - Security attributes that:
   - `noopener`: Prevents the new page from accessing the window.opener property (security best practice)
   - `noreferrer`: Prevents passing the referrer information to the linked website

The `{{< >}}` syntax indicates this is a Hugo shortcode, which is Hugo's way of creating reusable components within your content. This particular shortcode is called "link" and is likely defined in your Hugo theme or site configuration.

### Overleaf Documents

Most of my documents are written in overleaf. We will go through the procedure of converting an overleaf zipped package into GitHub Flavored Markdown. Make sure you have an updated version of `pandoc`

**Step 1: Extract Overleaf ZIP source file**

> This is assuming we have downloaded the directory to the root directory

Go to the zip file and extract contents to a directory

```bash
unzip DFT-FFT.zip -d dft-fft
```

**Step 2: Convert main LaTEX file to Markdown**

```bash
cd dft-fft
pandoc main.tex -o ../content/posts/dft-fft.pdc \
  --from=latex \
  --to=markdown \
  --extract-media=../static \
  --citeproc \
  --bibliography=references.bib \
  --standalone \
  --mathjax
```

#### Math Content

See [here](https://gohugo.io/content-management/mathematics/) on how to render `$` and `$$` in Hugo markdown.

We have used the `$` delimiter for inline equations. However, to avoid confusion, to use the regular `$` sign, we will need to add a double escape `A \\$5 bill is a awesome`

```html
<!-- add math rendering. will check frontmater, then section parameters, then site config in hugo.toml -->
{{ if .Param "math" }} {{ partialCached "math.html" . }} {{ end }}
```

We have also used `{{ if .Param "math" }}` instead of the reccomended `{{ if .Params.math }}` in step 3 of the guide. The reccomended version only checks the page configuration in `frontmatter`, but our version checks the site settings too. We have set default math rendering for everything.

> If you find that this makes loading the page much slower, you can simply set `math` to be `false` in the site config and selectively apply `math` to be `true` in the page frontmatter

## Deployment

- The `yaml` file is a GitHub actions workflow that automates the build and deployment process to GitHub Pages
- The `hugo.yaml` is a `CI/CD` workflow where a push to GitHub automatically triggers building and deployment

## Guides

The follow tutorials were used to learn Hugo and Github Pages:

- [Ed theme](https://gohugo-theme-ed.netlify.app/documentation/)
- [Hugo Quickstart](https://gohugo.io/getting-started/quick-start/)
- [Deploying Hugo on Github Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
