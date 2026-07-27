# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **GitHub Pages documentation site** (`gh-pages` branch, the only branch) for the **Atina JDE connector family** — integrations between Oracle JD Edwards EnterpriseOne and integration platforms (MuleSoft Anypoint, Boomi, and a planned "Builder" platform). There is no build system, tests, or linting — it is purely static documentation (HTML/AsciiDoc) served directly by GitHub Pages.

## Repository Structure

The repo root has **no site content of its own** — it only holds `CLAUDE.md` and two Excalidraw diagram files (`WEB-ATINA.excalidraw`, `WEB2.excalidraw`). Each integration platform is a **self-contained site** in its own top-level directory, with its own `index.html`/`index.css`/`index.js`, icons, and header image:

- `mule/` — MuleSoft Anypoint connector docs (the original/most complete site)
  - `1.0.0/apidocs/` — Connector API reference and release notes (AsciiDoc `.adoc` + generated `.html`). Two release-note tracks live side by side: `release-notes.adoc` (MuleSoft connector versions) and `release-notes-jde-atina.adoc` (Atina JDE Microservices versions), plus per-tool notes (`release-notes-jd-create-ini-files.adoc`, `release-notes-jd-create-jar-files.adoc`, `release-notes-jd-docker-files.adoc`)
  - `1.0.0/functional/` — Functional documentation and deployment guides (`jde.adoc`, `jdeatina.adoc`, `appendix-deployment-*.adoc`)
  - `1.0.0/quick_start/` — Quick setup overview (AsciiDoc + a `.pptx` walkthrough)
  - `1.0.0/benchmark/` — Performance benchmark doc with screenshots and a rendered `.pdf`
  - `1.0.0/migration/` — PortX-to-Atina migration guide
  - `1.0.0/demo/` — Sample connector `.jar` files
  - `1.0.0/exchange-home.md` — MuleSoft Exchange listing description
  - `1.0.0/jde docs/jde/` — **Antora**-structured module (`antora.yml`, `modules/ROOT/pages/`, `modules/ROOT/nav.adoc`) covering UBE, EDI, and MBF event demo guides — note the space in the `jde docs` directory name
- `boomi/` — Boomi connector docs, mirroring `mule/`'s `apidocs/`, `functional/`, and `quick_start/` layout (no benchmark/migration/jde-docs/demo sections)
  - `boomi/docs/` — Word documents (`.docx`) covering connection, connector overview, and operations — unique to this platform, not present under `mule/`
- `builder/` — placeholder for a future third platform; currently just an empty `1.0.1/` directory with no tracked content yet

When porting or comparing content between platforms, `mule/1.0.0/functional/jde.adoc` and `boomi/1.0.0/functional/jde.adoc` (and similarly `jdeatina.adoc`) are close variants of each other — check `diff` before assuming they're identical.

## Documentation Format

- Primary format is **AsciiDoc** (`.adoc`) with pre-rendered `.html` counterparts checked in alongside — when editing an `.adoc` file, regenerate its `.html` with Asciidoctor and commit both
- AsciiDoc files use standard Asciidoctor syntax with attributes like `:keywords:`, `[%header%autowidth.spread]` tables, and `ifdef` directives
- Images live in `images/` subdirectories adjacent to the doc that references them
- The `jde docs/` folder under `mule/1.0.0/` follows **Antora** module conventions (`modules/ROOT/pages/`, `modules/ROOT/nav.adoc`) rather than the flat AsciiDoc-next-to-HTML pattern used elsewhere

## Workflow

- All content lives on the `gh-pages` branch; there is no separate source/build branch
- Each platform directory (`mule/`, `boomi/`) is versioned independently under its own `<version>/` folder (currently `1.0.0` for both) — adding a new version means creating a new sibling version folder, not modifying the existing one
- Commit messages in this repo are frequently in Spanish (e.g. "Actualizacion de la version", "Agregado de Words de Documentacion") — match the existing convention unless the user asks otherwise
