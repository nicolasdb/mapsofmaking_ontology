# `crosswalk.csv` — the MOM overlap-resolution table

**Status: living bridge registry, v1** (Story 3.5, Epic 3). See [ADR-016](../_bmad-output/planning-artifacts/architecture.md) for the four-layer bundle model this operationalizes.

## What this is

`crosswalk.csv` is the single table that records, for every concept in the Maps of Making graph:

- which **`core:` / `mom:` predicate** the ingestion pipeline actually writes,
- which **SpaceAPI v15 field** (if any) it derives from,
- which **community-namespace field** (`fab:`, `omt:`, `edu:` …) aliases to it, and
- the **mapping type** that licenses the alias.

It is the documented form of the SpaceAPI → RDF mapping that `infra/link_handler/transformer.py::transform_to_sparql` performs implicitly today. The crosswalk makes that mapping reviewable and validatable.

## How to read a row

| Column | Meaning |
|---|---|
| `concept` | Stable, namespace-free name for the thing being mapped. |
| `core_field` | **The predicate the code actually emits today** — verified against `transformer.py`. Not the idealized `core:` name from the handoff. |
| `spaceapi_field` | The SpaceAPI v15 input field it derives from. Empty = MOM-derived (operational/metadata), no SpaceAPI source. |
| `fab_field` / `omt_field` / `edu_field` | Community-namespace fields that alias to `core_field`. |
| `mapping_type` | `skos:exactMatch` / `skos:closeMatch` for aliased rows; `direct` / `derived` / `metadata` / `snapshot` / `extension` otherwise. |
| `notes` | Layer, caveats, draft status. |

## The no-redefinition rule

No extension namespace may **redefine** a `core:`/`mom:` field — it may only **alias** it. `scripts/validate_crosswalk.py` enforces this: any row that fills both `core_field` and an extension field must declare a `skos:` mapping type. Run it standalone:

```bash
source venv/bin/activate
python scripts/validate_crosswalk.py
```

## The concept pivot (`schema:knowsAbout`)

The `activities` row is the hub of the model. `schema:knowsAbout` is the **shared concept pivot** — the "wormhole hub". Every community's specialised skill field (`fab:equipment`, `omt:treatmentFocus`, `edu:subjects`) aliases to it via `skos:closeMatch`. Because all of them resolve to the *same* concept IRIs in the activity scheme, a query crossing two communities works with **zero coordination** between them. Activity/skill concepts are labelled "concept commons (shared layer)" so the future `fab.ttl` extraction (Epic 9) does not mis-file them into `fab:`.

## Known incoherences (recorded, not silently fixed)

**`address`:** The handoff Layer-2 table and ADR-015 specify `schema:address → schema:PostalAddress`, but the live transformer (`transformer.py:507`) emits `mom:address` as a plain `xsd:string`. The crosswalk documents what the code does — `mom:address` — and flags the discrepancy. Reconciling is future work.

**`contact`:** SpaceAPI v15 has a rich `contact` object (`contact.email`, `contact.phone`, `contact.twitter`, `contact.mastodon`, `keymasters` …). The transformer serializes the whole object as a JSON string under the predicate `schema:contactJson` — a non-standard IRI that does not exist on schema.org (`schema:contactPoint` is the canonical term). The crosswalk documents `schema:contactJson` as-emitted; the `core:` concept IRI for this field is `mom:contactJson`. Properly modeling contact sub-fields as individual RDF properties is future work.

## Living document

This is **v1**. The crosswalk grows as the federation grows: every time a new community is onboarded and a cross-community concept overlap is discovered, a new `skos:closeMatch` bridge row is appended. `omt:` and `edu:` rows are `status: draft` placeholders — their namespace design is out of scope and requires community input. The crosswalk is never "finished"; it is a registry that accumulates bridges.
