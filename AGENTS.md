# Repository Guidelines

## Project Structure & Assets

The main Beamer source lives in `report.tex`; use it to orchestrate slides via `\input{}` if you split sections into `sections/*.tex`. Supporting bibliography is in `ref.bib`, while generated intermediates (`*.aux`, `*.nav`, etc.) are regenerated on every build and should stay untracked. Place new figures in the repository root with clear, lowercase names like `liveval4.png`; CSV inputs such as `category_result.csv` sit alongside media so they are easy to cite with `\pgfplotstableread`.

## Build & Preview Commands

Run `latexmk -pdf report.tex` for a clean rebuild of the PDF; it handles bibtex and multiple passes automatically. Use `latexmk -pvc report.tex` during drafting to trigger incremental recompiles on save. For quick one-off checks, `pdflatex report.tex` is acceptable, while `bibtex report` refreshes the bibliography after edits to `ref.bib`. Generated output is `report.pdf`.

## Coding Style & Naming Conventions

Keep indentation at two spaces inside environments; align optional arguments on separate lines when they get long. Define reusable macros in the preamble and prefix custom commands with `\newcommand{\my...}` to avoid collisions. Labels follow `\label{fig:tim_overview}` or `\label{sec:evaluation}` patterns so references stay descriptive. Comment large blocks with `% --- Section Title ---` to mark logical dividers.

## Testing & Quality Checks

Always compile with `latexmk -pdf` before submitting to confirm the slide deck builds without warnings. When LaTeX packages change, run `latexmk -gg -pdf report.tex` for a clean build. Optionally run `chktex report.tex` (if available) to catch typographic issues. Review the produced `report.pdf` to ensure figures render sharply and bibliography entries resolve.

## Commit & Pull Request Guidelines

Commit messages in this project are short descriptive sentences (Chinese or English) that summarise the change, for example “更新报告标题和内容…”. Use present-tense summaries and include filenames when meaningful. Each pull request should: link relevant issues, describe key slide additions or asset updates, and attach or reference the freshly compiled `report.pdf`. If introducing new datasets or graphics, note the source and licensing in the PR description.
