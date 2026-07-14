---
name: json-mass-assignment-finder
description: Detects an exported struct field an attacker can set via encoding/json Unmarshal that should not be externally controllable
---

**Finding ID Prefix:** `JSONMASSASSIGN`.

**Bug shape:** A struct with exported fields is unmarshaled directly from a request body (`json.Unmarshal(body, &user)` / `json.NewDecoder(r.Body).Decode(&user)`), and that same struct is then used directly for a privileged operation (saved to a database, used to authorize an action) without stripping or re-validating fields that shouldn't be attacker-settable — most classically a `Role`/`IsAdmin`/`Verified`/`Balance` field that should only ever be set server-side.

**Gates:**

1. A struct is populated via `json.Unmarshal`/`Decode` from a request body.
2. The struct has at least one exported field whose value plausibly grants elevated capability, bypasses a check, or represents server-controlled state (role, permission flag, ownership ID, verified/paid status, price/amount in a financial context).
3. No intervening allowlist/DTO-remapping step exists between the raw decode and the field's use in a privileged operation (e.g., no explicit copy of only the permitted fields into a separate internal struct).

**FPs (reject):**

- The struct used for decoding is a dedicated request DTO that intentionally excludes privileged fields (the privileged fields live only on a separate internal/domain struct never directly unmarshaled from the request).
- The privileged field is re-validated/overwritten from a trusted source (session, database lookup) immediately after decode, before any use — confirm this happens on every path, not just the common one.

**Patch:** decode into a narrow request-specific struct containing only the fields a client should be able to set, then explicitly copy those into the internal/domain struct field-by-field, leaving privileged fields to be set only by server-side logic.

**Search patterns:**

```
json\.Unmarshal\(
\.Decode\(
type\s+\w+\s+struct\s*\{
```
