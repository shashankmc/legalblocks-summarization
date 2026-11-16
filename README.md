# Summarization

> Generates document summaries via legal-summarizer-service

**LegalBlocks module package** `lawnotation.summarization` v0.1.0

- **Execution**: backend
- **Entrypoint**: `src/ui/Component.vue`
- **Inputs**: `lawnotation.text_layer`
- **Outputs**: `lawnotation.summary_layer`

## Configuration

| name | type | label | default |
|---|---|---|---|
| `service_url` | string | Service URL | `"http://legal-summarizer-service:8000"` |
| `model` | enum | Model | `"lex_rank"` |
| `summary_length` | enum | Length | `"medium"` |
| `api_key` | string | API key (leave empty if auth is disabled) | (blank) |

## Registering with LegalBlocks

This module is a standalone, publishable package. To register it:

1. `npm publish` (or push to its own git repo).
2. Add the package to a LegalBlocks builder project; the builder reads `package.manifest.json` and exposes the inputs, outputs, and configuration above.

## Local development

This package depends on unpublished workspace packages (`@legalblocks/*`, `@lawnotation/*`). Until they are published, link them locally:

```bash
npm link @legalblocks/module-runtime   # if this module does artifact I/O
npm install
# no tests in this module
```
