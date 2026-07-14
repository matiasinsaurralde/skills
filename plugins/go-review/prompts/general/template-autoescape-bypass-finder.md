---
name: template-autoescape-bypass-finder
description: Detects text/template used to render output ultimately served as HTML, losing html/template's automatic contextual escaping
---

**Finding ID Prefix:** `TEMPLATEXSS`.

**Bug shape:** Go has two nearly-identical templating packages: `html/template` (context-aware auto-escaping — safe default for HTML output) and `text/template` (no escaping at all — correct for non-HTML output like emails, config files, or CLI text). Using `text/template` to render a page/fragment that is ultimately served with `Content-Type: text/html` reintroduces classic reflected/stored XSS: any user-controlled value substituted into the template renders as raw, unescaped HTML/JS.

**Gates:**

1. `"text/template"` is imported and used to `Execute`/`ExecuteTemplate` a template.
2. The rendered output is written to an `http.ResponseWriter` (directly, or via a response body that's ultimately served with an HTML content type) rather than to a file, log, or non-HTML API response.
3. At least one template variable substituted into HTML-context output derives from user-controlled input.

**FPs (reject):**

- The `text/template` output is genuinely non-HTML (an email body, a YAML/JSON config file, a CLI report) and never reaches a browser as rendered HTML.
- The template's only variable substitutions are internally-generated, non-attacker-controlled values.
- `html/template` is already in use for the HTML-rendering path (confirm the import, not just the presence of `template.` calls, since both packages share the same package-local identifier `template` and are easy to confuse when skimming).

**Patch:** switch the import from `text/template` to `html/template` for any template whose output is served as HTML — the API is source-compatible, so the change is usually a one-line import swap plus re-verifying any manually-escaped values are no longer double-escaped.

**Search patterns:**

```
"text/template"
"html/template"
\.Execute\(|\.ExecuteTemplate\(
```
