# Web App Guidebook

A textbook-style reference to building and hosting web applications: the
concepts behind domains, identities, databases, services, storage, email and
releases, and step-by-step procedures for setting them up on a specific
platform.

**Read it at <https://alkaidcheng.github.io/webapp_guidebook/>.**

## Contents

- **Part I: Foundations**: the anatomy of a web application; domains, origins
  and HTTPS; identity and access; databases; API and web services; object
  storage; transactional email; environments and releases.
- **Part II: Tencent CloudBase**: environments and API keys, CAM users and
  policies, the PostgreSQL database, Cloud Run services, storage and email,
  and operating and moving an environment.

## Building locally

The site is generated from Markdown by
[bookspine](https://github.com/AlkaidCheng/bookspine), which needs Python 3.11 or
later.

```bash
pip install -r requirements.txt
bookspine build
python3 -m http.server --directory site 8000
```

Then open <http://127.0.0.1:8000/>. The build fails on a broken internal link,
a missing figure, a citation without a reference entry, or a reference that is
never cited.

## Writing a chapter

1. Write the chapter as Markdown in its part's directory under `book/`, with a
   YAML front matter block giving its `title`, a one-sentence `summary`, an
   optional `status` (for example `in-preparation`) and a `references` map of
   citation keys to formatted entries.
2. Add the file name to the part's `chapters` list in `book.yaml`. Chapters
   are numbered in that order, continuously through the book.
3. Use `##` for sections and `###` for subsections; the build numbers them.
   An image becomes a numbered figure with its alt text as caption; a table
   followed by a paragraph starting with `Table:` becomes a numbered table.
4. Cite with `[@key]`; the reference list is generated at the end of the
   chapter in order of first citation.
5. Set material apart with `!!! definition`, `!!! example`, `!!! remark`,
   `!!! recipe`, `!!! warning` or `!!! summary` blocks. The first four are
   numbered within the chapter.

Every section, subsection, figure, table and numbered box has a permalink:
the `§` mark that appears beside it copies a URL that opens the book at that
place. Section anchors are derived from the heading text, so renaming a
heading changes its link. Figures and tables open enlarged from their button.

A new part is a new directory with an `index.md` opener and an entry in
`book.yaml`. A new platform part follows the order of Part I, so that each of
its chapters can refer back to the concept it puts into practice.

Every pull request is built by CI, and each push to `main` is deployed to
GitHub Pages.

## License

[MIT](LICENSE)
