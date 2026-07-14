# Repository Guidelines

## Project Structure & Module Organization

This repository builds a Chinese-language NLP guide with `mdBook`. Published content lives under `src/`; `src/SUMMARY.md` defines chapter order and navigation, while `src/README.md` is the book landing page. Numbered directories such as `src/02-Regex/` and `src/05-IsmaTokenizer/` group related chapters. `book.toml` contains title, language, theme, and GitHub Pages settings. The similarly named top-level numbered directories contain legacy source documents; edit `src/` for book-facing changes unless intentionally migrating material. The generated `book/` directory is build output and should not be committed.

## Build, Test, and Development Commands

Install mdBook with the same locked Cargo workflow used in CI:

```sh
cargo install mdbook --locked
mdbook serve --open
```

`mdbook serve --open` starts a local preview and rebuilds after edits. Run `mdbook build` before submitting changes; it renders the site into `book/` and catches structural or parsing failures. GitHub Actions repeats this build on pushes to `main` and deploys the result to GitHub Pages.

## Coding Style & Naming Conventions

Write concise Markdown in Chinese, matching the existing instructional tone. Use ATX headings (`#`, `##`), fenced code blocks with language identifiers, and relative links for repository content. Keep one top-level heading per chapter, add blank lines around headings and lists, and avoid unnecessary raw HTML. Chapter filenames use lowercase kebab-case, for example `unicode-and-utf8.md`; retain the numbered topic-directory convention. When adding or renaming a chapter, update `src/SUMMARY.md` in the same change.

## Testing Guidelines

There is no standalone automated test suite or coverage target. Treat `mdbook build` as the required check. Preview changed chapters with `mdbook serve`, verify navigation from `SUMMARY.md`, test internal links and images, and confirm code samples and terminology render correctly. For broad edits, inspect both light and navy themes.

## Commit & Pull Request Guidelines

Recent history uses short, imperative, sentence-case subjects such as `Refine Unicode chapter` and `Set up mdBook structure for Yishi Guide`. Follow that style and keep each commit focused. Pull requests should summarize affected chapters, explain structural decisions, and report `mdbook build` results. Link relevant issues; include screenshots when layout, equations, diagrams, or theme-sensitive rendering changes.
