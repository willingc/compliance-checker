# compliance-checker

Privacy audit for [doc quality compliance checker](https://github.com/IloBe/doc_quality_compliance_check)

This repo holds information related to a privacy audit.

## Project for audit

[Repo for doc_quality_compliance_check](https://github.com/IloBe/doc_quality_compliance_check) by Ilona Brinkmeier

## Privacy audit

[Website](https://willingc.github.io/compliance-checker/)


---

## Contribute to the documentation

- Add a document page in markdown in the `docs` folder.
- Add the document to the `nav` section in the `zensical.toml` file.
- Add images to the `docs/images` folder.

### Build and run docs locally

1. Install pixi if needed: https://pixi.prefix.dev/v0.19.0/#installation
2. Using pixi from the terminal. At the root of the repo, run:

```sh
pixi run serve
```

3. Navigate to http://127.0.0.1:8000/compliance-checker in your browser to view the documentation.

---

## Capstone presentation

In `docs/capstone-presentation.md`

To work on the presentation:

1. In vscode, add Marp for VSCode extension

2. Open the `docs/capstone-presentation.md` file.

3. Preview the document and you should see the slides.

![](docs/images/image.png)

There is also a [marp](https://marp.app/).

### More marp info

There are three themes: default, gaia, and uncover. Set in the header block at top of file.

To convert md file to html:

```sh
marp docs/capstone-presentation.md -o docs/capstone-presentation.html
```

To present, either:

`open docs/captone-presentation.html`

or

`python3 -m http.server` and navigate to `localhost:8000/docs/capstone-presentation.html`

To convert md file to pdf:

```sh
marp --pdf --allow-local-files docs/capstone-presentation.md
```
