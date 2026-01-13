# Website Cloner

A Claude Code plugin that reverse-engineers websites into specs for cloning.

## Overview

Website Cloner uses AI agents to explore a target website, document its structure, components, and behavior, and generate detailed specs that can be used to rebuild it. This is spec-driven development in reverse - instead of writing specs first, you discover them from an existing site.

## Features

- **Iterative Discovery** - Explores websites page-by-page, building a comprehensive understanding
- **Component Extraction** - Identifies reusable UI components and documents them separately
- **Fidelity Levels** - Choose between pixel-perfect, functional, or structural cloning
- **Human-in-the-Loop** - Asks for clarification when behavior is ambiguous
- **Ralph Integration** - Generated specs work seamlessly with the Ralph methodology for implementation

## Requirements

- Claude Code CLI
- [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome) extension for browser automation
- [Ralph plugin](https://github.com/eumemic/ralph) (optional, for building the clone)

## Installation

```bash
/plugin install website-cloner@eumemic
```

Or add via marketplace:

```bash
/plugin marketplace add eumemic/claude-plugins
/plugin install website-cloner@eumemic
```

## Usage

Simply tell Claude what you want to clone:

```
"Help me clone the homepage of example.com"
"I want to reverse-engineer this e-commerce checkout flow"
"Clone the dashboard from this SaaS app"
```

Or invoke the skill directly:

```
/clone-site
```

### Workflow

1. **Setup** - Provide target URL, scope, and fidelity preferences
2. **Discovery** - Agents explore the site and generate specs
3. **Refinement** - Review specs, answer open questions, run more discovery waves
4. **Build** - Transition to Ralph plan/build to implement the clone

### Configuration

After setup, configuration is stored in `.ralph/discovery-config.yaml`:

```yaml
target:
  url: https://example.com
  scope: |
    Clone the main dashboard and settings pages.
    Ignore marketing pages and blog.
  url_patterns:
    include:
      - /dashboard/*
      - /settings/*
    exclude:
      - /blog/*

fidelity: functional  # pixel-perfect | functional | structural
backend_scope: frontend-only  # frontend-only | full-stack
workers: 1
```

### Output

Generated specs follow the Ralph spec format:

```
specs/
├── dashboard.md
├── settings.md
└── components/
    ├── navigation-header.md
    ├── sidebar.md
    └── data-table.md
```

Each spec includes:
- Source URLs for traceability
- Layout and structure documentation
- Component references
- Behavior descriptions
- Design decisions for ambiguous cases

## Fidelity Levels

| Level | Use Case | What's Captured |
|-------|----------|-----------------|
| **pixel-perfect** | Exact visual clone | Colors, typography, spacing, responsive breakpoints |
| **functional** | Same features, different look | Component structure, interactions, data flows |
| **structural** | Information architecture only | Page hierarchy, content types, navigation |

## License

MIT
