# alexsch.dev

Alex Schneider's personal blog, built with Jekyll and the local [no-style-please](https://github.com/riggraz/no-style-please) theme. Published to GitHub Pages through `.github/workflows/jekyll.yml` on pushes to `master`.

## Local preview

```sh
bundle install
bundle exec jekyll serve
```

## Add a post

Create `_posts/YYYY-MM-DD-post-name.md` with `layout: post`, `title`, and `date` in its YAML front matter.

For another Road to Nationals post, also add:

```yaml
categories: [road-to-nationals]
series: Road to Nationals
series_part: 2
series_url: /road-to-nationals.html
```

The series index and navigation update automatically. The first post retains a placeholder for the Radio Club rack photo. Proxmox and scoring screenshots are in `assets/images/road-to-nationals/`.

## Hosting

The site uses `https://alexsch.dev`, an empty `baseurl`, and a `CNAME` file. GitHub Pages must use GitHub Actions as its source. Point the domain's DNS to GitHub Pages and enable HTTPS when its certificate is ready.

The original theme remains under the MIT license in `LICENSE.txt`.
