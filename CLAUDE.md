# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **GitHub Pages documentation site** (`gh-pages` branch) for the **Atina JDE Anypoint Connector** — a MuleSoft connector that integrates Oracle JD Edwards EnterpriseOne with the Anypoint Platform. The site is hosted at `atina-connector.github.io/JDEAtinaJDEMuleConnectorDoc4/`.

This repo contains **no build system, tests, or linting** — it is purely static documentation served by GitHub Pages.

## Repository Structure

- `index.html` — Landing page with version table linking to docs, release notes, and demo JARs
- `index.css` / `index.js` — Styling and behavior for the landing page
- `1.0.0/` — Documentation for version 1.0.0:
  - `apidocs/` — Connector API reference and release notes (AsciiDoc `.adoc` + generated `.html`)
  - `functional/` — Functional documentation and deployment guides
  - `benchmark/` — Performance benchmark docs with screenshots
  - `migration/` — Migration guide (PortX to Atina)
  - `demo/` — Sample connector JAR files
  - `jde docs/` — Antora-structured module pages (demo guides for UBE, EDI, MBF events)
  - `exchange-home.md` — MuleSoft Exchange listing description
- `3.0.0/functional/` — Functional docs for version 3.0.0 (HTML only)

## Documentation Format

- Primary format is **AsciiDoc** (`.adoc`) with pre-rendered `.html` counterparts
- AsciiDoc files use standard Asciidoctor syntax with attributes like `:keywords:`, `[%header%autowidth.spread]` tables, and `ifdef` directives
- Images are stored in `images/` subdirectories adjacent to their docs
- The `jde docs/` folder follows **Antora** module conventions (`modules/ROOT/pages/`, `modules/ROOT/nav.adoc`)

## Workflow

- All content lives on the `gh-pages` branch (the only branch); there is no separate source/build branch
- To update docs: edit `.adoc` source files, regenerate `.html` (using Asciidoctor), and commit both
- Release notes in `apidocs/release-notes-jde-atina.adoc` track Atina JDE Microservices versions; `release-notes.adoc` tracks the MuleSoft connector versions
