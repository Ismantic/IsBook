# Repository Guidelines

## Project Structure & Module Organization

This repository contains three independent Chinese-language `mdBook` projects. `Text/` is the text-processing book, `Matx/` covers compiler and runtime internals, and `Zero/` covers training-engine implementation. Each directory owns its `book.toml`, `README.md`, `SUMMARY.md`, and chapter files. `Legacy/` contains the previous combined-book structure and should only be edited when recovering or comparing old material. Generated output directories are `book-text/`, `book-matx/`, and `book-zero/` and must not be committed.

## Build, Test, and Development Commands

Install mdBook with the same locked Cargo workflow used in CI:

```sh
cargo install mdbook --locked
```

Build or preview the relevant book from the repository root:

```sh
mdbook build Text
mdbook build Matx
mdbook build Zero
mdbook serve Text --open
```

Run the build for every book affected by a change. GitHub Actions currently deploys the `Text/` book on pushes to `main`.

## Coding Style & Naming Conventions

Write concise Markdown in Chinese, matching the existing instructional tone. Use ATX headings (`#`, `##`), fenced code blocks with language identifiers, and relative links for repository content. Keep one top-level heading per chapter, add blank lines around headings and lists, and avoid unnecessary raw HTML. New chapter filenames should use lowercase kebab-case, for example `unicode-and-utf8.md`. When adding or renaming a chapter, update the `SUMMARY.md` belonging to that book.

## Testing Guidelines

There is no standalone automated test suite or coverage target. Treat the relevant `mdbook build <directory>` command as the required check. Preview changed chapters with `mdbook serve <directory>`, verify navigation from that book's `SUMMARY.md`, test internal links and images, and confirm code samples and terminology render correctly. For broad edits, inspect both light and navy themes.

## Commit & Pull Request Guidelines

Recent history uses short, imperative, sentence-case subjects such as `Refine Unicode chapter` and `Set up mdBook structure for Yishi Guide`. Follow that style and keep each commit focused. Pull requests should summarize affected chapters, explain structural decisions, and report `mdbook build` results. Link relevant issues; include screenshots when layout, equations, diagrams, or theme-sensitive rendering changes.
