# Access policies (ODRL)

FDPneo controls access to records with W3C ODRL **Offers**. A record opts into a policy through `dct:rights`, and the server's policy decision point enforces it on every request, whether it comes from the web client, the API or the SPARQL endpoint.

Open **avatar, Policies** as an administrator.

![The policies page](../assets/screenshots/09-policies.webp)

## The FDPneo ODRL profile

Policies must stay within a documented subset of ODRL, which keeps the evaluator simple and the editor coherent:

| Element | Supported |
|---|---|
| Policy types | `odrl:Offer` (authored) and `odrl:Agreement` (materialised by the server on grant) |
| Rules | `odrl:Permission` and `odrl:Prohibition`; no Duties |
| Actions | `odrl:read`, `odrl:modify`, `odrl:delete`, `odrl:distribute` |
| Constraints | party identity, role membership, group or organisation membership, time windows |
| Conflict strategy | deny wins by default; `odrl:conflict` may override |

Purpose, spatial, payment and count constraints are not part of the profile.

## The guided composer

The composer builds an Offer from the profile only, so it cannot produce an out-of-profile policy:

1. Pick an **action** (read, modify, delete, distribute).
2. Add **permissions** and **prohibitions**.
3. Attach **constraints**: a role (for example `steward`), a specific party, a group, or a time window.
4. Choose the **conflict strategy**.

A live Turtle preview and inline validation show exactly what will be stored, and **Validate** runs the server's profile check. Save, publish or delete managed policies; published policies appear in the record form's *Access policy* picker.

## Inheritance and the system default

A record without `dct:rights` inherits the policy of its parent: a Dataset from its Catalog, a Catalog from the Repository, and the Repository from the system default. Inheritance is resolved by walking up at lookup time, so every record's effective policy can be discovered by following the chain.

The system ships a default Offer, `system-default`, that grants public read and steward modify. It is the baseline until you author your own.

## Versioning and audit

Offers are versioned and immutable. Editing access conditions creates a new version and points the record at it; earlier versions are preserved. When the server permits an access, it materialises an Agreement that references the exact Offer version in force, the assignee and the action, and stores it in the record's audit graph. Audit graphs are visible to the record's owner and to administrators, never to anonymous users.

## Policies and licenses are different things

A policy governs who may do what with the metadata record. A [license](licenses.md) describes the terms under which the described data may be reused. Records usually reference both.
