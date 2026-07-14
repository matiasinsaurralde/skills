---
name: cluster-deserialization-safety
kind: cluster
consolidated: false
covers:
  - json-mass-assignment            # JSONMASSASSIGN
  - xml-entity-expansion            # XMLENTITY
  - gob-arbitrary-type              # GOBTYPE
  - recursive-decode-stack-overflow # RECURSEDECODE
---

# Cluster: Deserialization safety

Go's `encoding/json` is memory-safe by construction (no buffer overflows from malformed input), so this cluster's bugs are all **logic/availability** issues: fields an attacker shouldn't be able to set, entity-expansion footguns reintroduced by custom XML decoder config, `encoding/gob`'s interface-decoding trusting the wire format's declared type, and unbounded recursion in custom `UnmarshalJSON`/`UnmarshalYAML` methods — this last one is Go's analogue of `rust-review`'s `recursion-dos` cluster (stack-overflow DoS on attacker-shaped nested input), scoped down to a single pass since Go has far fewer recursive-deserialization entry points than Rust's `serde` ecosystem.

ID prefixes: `JSONMASSASSIGN`, `XMLENTITY`, `GOBTYPE`, `RECURSEDECODE`.

## Phase A

```
rg seed: "json\.Unmarshal\(|json\.NewDecoder\("        # JSON deserialization sites
rg seed: "type\s+\w+\s+struct\s*\{"                     # struct definitions (cross-reference exported fields against Unmarshal targets)
rg seed: "xml\.Decoder\{|d\.Entity\s*="                 # custom XML entity map (reintroduces expansion the stdlib disables by default)
rg seed: "gob\.Register\(|gob\.NewDecoder\("            # gob usage — Register calls reveal which concrete types an interface field can decode to
rg seed: "func\s*\([^)]*\)\s*UnmarshalJSON\(|func\s*\([^)]*\)\s*UnmarshalYAML\("  # custom unmarshalers (candidate unbounded recursion)
```

Run finders in declared order.

## Deconfliction

- `JSONMASSASSIGN` fires on a struct with exported fields (e.g. `IsAdmin bool`, `Role string`) that are directly unmarshaled from a request body with no allowlist/DTO layer separating wire format from internal/privileged fields — not merely because a struct has exported fields (most Go structs do; the finding requires the field to plausibly grant elevated capability if attacker-set).
- `XMLENTITY` is specifically about **custom** `Decoder.Entity` maps reintroducing external entity resolution — Go's default `encoding/xml` decoder does not resolve external entities, so a bare `xml.Unmarshal`/`xml.NewDecoder` with no custom `Entity` map is not vulnerable and should be marked `cleared`, not filed.
- `RECURSEDECODE` requires the custom `UnmarshalJSON`/`UnmarshalYAML` to recurse on a **nested, attacker-sized** structure with no depth limit — a flat unmarshal with no recursion is out of scope for this pass.
