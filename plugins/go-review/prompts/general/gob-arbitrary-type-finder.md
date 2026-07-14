---
name: gob-arbitrary-type-finder
description: Detects encoding/gob decoding attacker-controlled bytes into an interface-typed field, letting the wire format pick the concrete type
---

**Finding ID Prefix:** `GOBTYPE`.

**Bug shape:** `encoding/gob` can decode into an `interface{}`-typed field, with the concrete type determined by what the encoder registered via `gob.Register(...)` and what the wire data declares — if the decoded byte stream is attacker-controlled (a cache value, a message from an untrusted peer, a session blob), the attacker effectively chooses which registered type gets instantiated and populated, which can be abused if any registered type's zero-value-to-populated transition has a side effect, or if type confusion downstream leads to a logic bug.

**Gates:**

1. `gob.NewDecoder(...).Decode(...)` targets a field/variable of interface type (directly, or via a struct field typed as an interface).
2. The gob-encoded byte stream's origin is at least partially attacker-controlled (deserializing a value from a cache, queue, or session store that an attacker can influence — not purely internal, process-generated data).
3. More than one type is `gob.Register`ed for that interface, giving the attacker a choice.

**FPs (reject):**

- The gob stream is exclusively produced and consumed by the same trusted process (e.g., serializing to a local temp file with no external write access) with no attacker influence over its contents.
- Only one type is ever registered for the interface, so there's no meaningful type-confusion choice even though the mechanism is technically present.

**Patch:** avoid decoding untrusted gob data into interface-typed fields; if cross-service serialization of untrusted data is required, prefer a schema-strict format (protobuf with explicit oneof, or JSON into a concrete, non-interface struct) instead of gob's registry-based interface decoding.

**Search patterns:**

```
gob\.Register\(
gob\.NewDecoder\(
interface\{\}
```
