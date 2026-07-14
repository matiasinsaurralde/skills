---
name: xml-entity-expansion-finder
description: Detects a custom encoding/xml Decoder.Entity map that reintroduces external entity expansion the stdlib disables by default
---

**Finding ID Prefix:** `XMLENTITY`.

**Bug shape:** Go's `encoding/xml` decoder does **not** resolve external entities by default — a bare `xml.Unmarshal`/`xml.NewDecoder(...).Decode(...)` is not vulnerable to classic XXE. The bug only exists when application code explicitly sets a custom `Decoder.Entity` map (intended to support a fixed, known set of custom internal entities) in a way that inadvertently allows attacker-supplied entity definitions to be resolved, or that maps entity names to values sourced from untrusted input.

**Gates:**

1. Code sets `decoder.Entity = ...` to a custom map.
2. The map's values (or its keys, if dynamically constructed) derive from or are influenced by untrusted input, rather than being a small, fixed, hardcoded set of internal entity substitutions.

**FPs (reject):**

- No custom `Entity` map is set anywhere — the default decoder is safe by construction; do not file a finding merely because `encoding/xml` is imported.
- The `Entity` map is a small, fixed, hardcoded set of internal substitutions with no attacker influence over keys or values.

**Patch:** if a custom `Entity` map is genuinely needed, keep it a fixed, hardcoded, minimal set with no attacker-influenced content; otherwise remove the custom map entirely and rely on the safe default.

**Search patterns:**

```
xml\.Decoder\{|\.Entity\s*=
xml\.Unmarshal\(|xml\.NewDecoder\(
```
