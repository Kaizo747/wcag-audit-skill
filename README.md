# wcag-audit-skill
A reusable Claude skill for running WCAG 2.2 accessibility audits with automated scans, static checks, and manual verification guidance.

## Why this exists

Accessibility audits can be slow and inconsistent when they are done manually. This skill helps produce structured, repeatable reports for web projects.

## Features

- Runtime accessibility scans.
- Static analysis for JSX/TSX projects.
- Manual verification checklist for WCAG 2.2 criteria that cannot be fully automated.
- Structured reporting for developer handoff.

## What’s included

- `SKILL.md` — the Claude skill definition.
- `examples/sample-report.md` — a sample report.
- `LICENSE` — reuse terms.
- `CONTRIBUTING.md` — how to contribute improvements.

## Usage

Ask Claude to audit a web project for WCAG 2.2 AA or AAA compliance, then follow the structured report format defined in `SKILL.md`.

### Example prompts

- Audit this React app for WCAG 2.2 AA issues.
- Run an accessibility scan and list violations with fixes.
- Check this Next.js project for contrast, labels, focus, and keyboard issues.

## Project structure

```text
claude-accessibility-audit-skill/
├─ SKILL.md
├─ README.md
├─ LICENSE
├─ CONTRIBUTING.md
└─ examples/
   └─ sample-report.md
```

## Limitations

Automated tools do not replace manual accessibility testing. Some WCAG 2.2 criteria require human review or testing with assistive technology.

## Contributing

See `CONTRIBUTING.md`.

## License

MIT License.
