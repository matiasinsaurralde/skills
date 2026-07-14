---
name: nil-pointer-deref-finder
description: Detects a nil pointer dereference on an unchecked optional struct field, commonly from JSON/proto unmarshal
---

**Finding ID Prefix:** `NILDEREF`.

**Bug shape:** A struct field of pointer type (`*SubStruct`) that a schema (JSON, protobuf, YAML) allows to be absent/null is accessed (`obj.Field.Nested`) without a preceding nil check. When the field is genuinely optional in the wire format, an attacker who omits it can trigger a nil dereference and crash the handling goroutine.

**Gates:**

1. A struct field is of pointer type, and the struct is populated via `json.Unmarshal`/`proto.Unmarshal`/`yaml.Unmarshal` (or an equivalent decode) from a source an attacker influences (request body, uploaded file, message payload).
2. The field is dereferenced (direct field access, method call) without a preceding `if field != nil` guard on the same path.
3. The wire schema does not mark the field as required in a way that's enforced before this access point (check for validation middleware/library calls like a `Validate()` method or JSON-schema check that would have already rejected a nil field).

**FPs (reject):**

- A validation step (explicit `Validate()` call, JSON schema enforcement, or protobuf `required` semantics enforced upstream) provably runs before this access and would reject the request if the field were nil.
- The field is a value type (not a pointer) — Go structs decode missing fields to their zero value, not nil, for non-pointer types, so there's no nil-deref risk there (though a zero-value logic bug might still exist — out of scope for this pass).

**Patch:** add an explicit nil check before dereferencing, or provide a default via a constructor/`Validate()` step that runs before the handler logic.

**Search patterns:**

```
json\.Unmarshal\(|proto\.Unmarshal\(|yaml\.Unmarshal\(
\.\w+\.\w+
if\s+\w+(\.\w+)*\s*!=\s*nil
```
