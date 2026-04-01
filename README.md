# jupaygon.com

Personal blog and portfolio built with [Hugo](https://gohugo.io/) and hosted on [GitHub Pages](https://pages.github.com/).

[![Deploy Hugo site to Pages](https://github.com/jupaygon/jupaygon-web/actions/workflows/hugo.yaml/badge.svg)](https://github.com/jupaygon/jupaygon-web/actions/workflows/hugo.yaml)

## Stack

- **SSG:** Hugo
- **Theme:** [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (dark mode, customized)
- **Hosting:** GitHub Pages
- **CI/CD:** GitHub Actions
- **Domain:** jupaygon.com

## Local development

```bash
hugo server --buildDrafts
```

Open [http://localhost:1313](http://localhost:1313)

## New post

```bash
hugo new content posts/my-new-post.md
```

## Multilingual cover images

Hugo PaperMod does **not** copy page bundle assets to translated versions. Cover images in `index.es.md` must use an absolute path pointing to the English version:

```yaml
# CORRECT
cover:
  image: '/en/posts/YYYY-MM-DD/slug/cover.png'
  relative: false

# WRONG — image won't exist at /es/.../cover.png
cover:
  image: 'cover.png'
  relative: true
```

## Author

**Juanjo Payá** — Senior Software Engineer · [GitHub](https://github.com/jupaygon) · [X](https://x.com/jupaygon)

AI-assisted development with [Jarvis](https://github.com/jarvis-aidev) as collaborator.
