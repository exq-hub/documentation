# Live Services Engine — Documentation

Documentation site for the **Exquisitor Live Services Engine (LSE)**, a Python/FastAPI backend for multimedia search powered by CLIP, SVM relevance feedback, and faceted filtering.

The site is built with [Eleventy (11ty)](https://www.11ty.dev/), [Basecoat](https://basecoatcss.com/), and [Tailwind CSS](https://tailwindcss.com/) using the [ReallySimpleDocs](https://reallysimpledocs.com/) template.

Published at: **https://docs.exquisitor.org**

## Local development

```bash
npm install
npm run dev
```

The site will be served at `http://localhost:8080` with live reload.

## Build

```bash
npm run build
```

Output goes to `_site/`.

## Project structure

```
docs/           Markdown source files (content)
_includes/      Eleventy layout templates
_data/          Global site data (site.json)
assets/         Static assets (CSS, JS)
src/css/        Tailwind CSS entry point
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Documentation content is licensed under [Creative Commons Attribution-ShareAlike 4.0 International](LICENSE).
