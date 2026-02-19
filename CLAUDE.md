# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WinOpsLab is a Windows operations and automation laboratory project. The repository is currently in its initial state — architecture, tooling, and conventions will be documented here as the project takes shape.

## Platform

- Target: Windows 11
- Shell: PowerShell / Bash (Git Bash)

## Documentation (MkDocs)

- Static site generator: MkDocs with Material theme
- Documentation content is written in **French** (personal learning notes)
- Code comments in configuration files (mkdocs.yml, CSS) remain in **English**
- Virtual environment: `.venv/` (activate with `.\.venv\Scripts\Activate.ps1`)
- Preview: `mkdocs serve` (http://127.0.0.1:8000)
- Build: `mkdocs build --strict`

### Documentation conventions

- Section index pages use Material card grids (`<div class="grid cards" markdown>`)
- Content pages include YAML frontmatter (`title`, `description`, optionally `tags`)
- Admonitions for tips, warnings, info blocks (`!!! tip`, `!!! warning`, etc.)
- Collapsible sections for lab solutions (`??? success "Solution"`)
- PowerShell code blocks with syntax highlighting (` ```powershell `)
- Navigation structure defined explicitly in `mkdocs.yml` `nav:` section
- Stub pages use `!!! warning "En cours de redaction"` banner
